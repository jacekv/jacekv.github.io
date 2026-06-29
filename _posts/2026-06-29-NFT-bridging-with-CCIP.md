---
layout: post
title: "Bridging NFTs Cross-Chain with Chainlink CCIP"
description: "A step-by-step guide to building a lock-and-mint NFT bridge between Ethereum Sepolia and OP Sepolia using Chainlink CCIP."
tags: Ethereum Blockchain CCIP Chainlink NFT Bridging Cross-Chain
---

# Bridging NFTs Cross-Chain with Chainlink CCIP

NFTs are native to the chain they are minted on. An ERC-721 token deployed on Ethereum lives on Ethereum - there is no built-in way to "move" it to another chain. But users increasingly want their NFTs to be usable across multiple ecosystems, and protocols want to support them without requiring users to abandon the original.

In this post we'll walk through building a cross-chain NFT bridge using the **lock-and-mint** pattern on top of **Chainlink CCIP** (Cross-Chain Interoperability Protocol). We'll go from the contract design all the way to a live testnet round-trip: locking an NFT on Ethereum Sepolia, minting a wrapped copy on OP Sepolia, and then burning it to get the original back.

## Table of Contents

1. [Two bridging patterns: burn+mint vs lock+mint](#two-bridging-patterns)
2. [Architecture overview](#architecture-overview)
3. [Source chain: NFTVault](#source-chain-nftvault)
4. [Destination chain: WrappedNFT](#destination-chain-wrappednft)
5. [Security: the allowlist system](#security-the-allowlist-system)
6. [Security: tokenSourceVault](#security-tokensourcevault)
7. [Deployment and setup](#deployment-and-setup)
8. [Testing end-to-end](#testing-end-to-end)
9. [Design decisions worth knowing](#design-decisions-worth-knowing)
10. [Conclusion](#conclusion)

---

## Two Bridging Patterns

Before writing a line of code it is worth deciding which bridging pattern to use.

**Burn-and-mint** destroys the token on the source chain and mints a new one on the destination. It is conceptually clean — there is only ever one copy in existence — but it permanently removes the original. If the destination chain's contract is ever compromised, the original is gone. More importantly, burn-and-mint requires minting permissions on the source collection: when the token returns from the destination chain, something has to call `mint` on the original contract. If you control the NFT contract, that is straightforward. If you are building a bridge for an existing third-party collection where you have no such permissions, burn-and-mint is simply not an option.

**Lock-and-mint** keeps the original on the source chain, locked in a vault contract. A synthetic wrapped copy is minted on the destination. When the user wants their original back they burn the wrapped copy and the vault releases the original. The original is never destroyed, and — crucially — the vault only needs `transferFrom` permission (which the user grants via `approve`), not any special role on the NFT contract itself. This makes it the right choice whenever you are bridging an existing collection you do not control.

We are building the lock-and-mint variant. The NFT contract (`ProviderNFT`) is a stand-in for any third-party ERC-721 — the vault is entirely collection-agnostic.

---

## Architecture Overview

The system has two contracts, one on each chain:

- **`NFTVault`** (Ethereum Sepolia) — receives deposits, locks NFTs, and releases them when a burn confirmation arrives from the other chain.
- **`WrappedNFT`** (OP Sepolia) — mints wrapped tokens when a deposit message arrives, and burns them when the user wants to return to the source chain.

CCIP is the transport layer. There is no custom relayer, no off-chain oracle, and no multisig bridge committee. The two contracts talk to each other through the Chainlink router, and the only trust assumption beyond the contracts themselves is the CCIP protocol.

Here is the full round-trip flow:

```
User (Sepolia)
  │
  ├─ approve(NFTVault, tokenId)
  └─ deposit(tokenId, WrappedNFT, OP_SELECTOR)
       │  locks NFT in vault
       └─ CCIP ──────────────────────────────────────────────────────►
                                                                       │
                                                             WrappedNFT._ccipReceive
                                                               ├─ mint wrapped token
                                                               ├─ store tokenURI
                                                               └─ store sourceVault

User (OP Sepolia)
  └─ burn(tokenId, recipient, SEPOLIA_SELECTOR)
       │  burns wrapped token
       └─ CCIP ◄──────────────────────────────────────────────────────
  │
  NFTVault._ccipReceive
    └─ safeTransferFrom(vault → recipient)
```

---

## Source Chain: NFTVault

`NFTVault` inherits from three interfaces:
- `CCIPReceiver` — so it can receive inbound CCIP messages (the burn confirmation)
- `OwnerIsCreator` — Chainlink's lightweight ownership primitive
- `IERC721Receiver` — so `safeTransferFrom` does not revert when the NFT is pulled in

### Locking: the `deposit` function

```solidity
function deposit(
    uint256 tokenId,
    address receiver,
    uint64 destinationChainSelector
)
    public
    payable
    onlyAllowlistedDestinationChain(destinationChainSelector)
    validateReceiver(receiver)
    returns (bytes32 messageId)
{
    if (msg.sender != nft_contract.ownerOf(tokenId)) revert NotOwner(msg.sender, tokenId);
    if (nft_contract.getApproved(tokenId) != address(this)) revert NotApproved(tokenId, address(this), address(nft_contract));

    string memory tokenUri = nft_contract.tokenURI(tokenId);

    bytes memory payload = abi.encode(
        tokenUri,
        tokenId,
        msg.sender,           // original owner — will be minted-to on the destination
        address(nft_contract) // source collection address (carried for extensibility)
    );

    Client.EVM2AnyMessage memory evm2AnyMessage = buildCCIPMessage(receiver, payload, address(0));
    uint256 fees = getCCIPMessageFee(destinationChainSelector, evm2AnyMessage);
    if (fees == 0 || msg.value < fees) revert NotEnoughBalance(msg.value, fees);

    nft_contract.safeTransferFrom(msg.sender, address(this), tokenId); // lock first

    IRouterClient router = IRouterClient(this.getRouter());
    messageId = router.ccipSend{value: fees}(destinationChainSelector, evm2AnyMessage);

    emit MessageSent(messageId, destinationChainSelector, receiver, payload, fees);

    uint256 refundAmount = msg.value - fees;
    if (refundAmount > 0) {
        (bool success,) = msg.sender.call{value: refundAmount}("");
        if (!success) revert RefundFailed(msg.sender, refundAmount);
    }

    return messageId;
}
```

A few things worth noting here:

**The `tokenURI` is read on-chain and travels with the message.** This means metadata is available the instant the message lands on the destination — no extra cross-chain call is needed to query it. The tradeoff is a larger CCIP payload for tokens with long URIs.

**The NFT is transferred into the vault *before* the CCIP send, not after.** This eliminates a race window where the user could transfer the NFT to someone else between approval and the cross-chain message being sent.

**Fees are paid in native ETH (`feeToken: address(0)`), and any excess is refunded.** Callers can safely overpay — they will always get the difference back.

### Unlocking: `_ccipReceive`

When the wrapped token is burned on OP Sepolia, a CCIP message arrives here carrying `(tokenId, recipient)`. The vault simply releases the original:

```solidity
function _ccipReceive(
    Client.Any2EVMMessage memory any2EvmMessage
)
    internal
    override
    onlyAllowlisted(any2EvmMessage.sourceChainSelector, abi.decode(any2EvmMessage.sender, (address)))
{
    (uint256 tokenId, address recipient) = abi.decode(any2EvmMessage.data, (uint256, address));

    emit NFTUnlocked(
        any2EvmMessage.messageId,
        any2EvmMessage.sourceChainSelector,
        abi.decode(any2EvmMessage.sender, (address)),
        tokenId,
        recipient
    );

    nft_contract.safeTransferFrom(address(this), recipient, tokenId);
}
```

---

## Destination Chain: WrappedNFT

`WrappedNFT` inherits from `ERC721`, `OwnerIsCreator`, and `CCIPReceiver`. Because both `ERC721` and `CCIPReceiver` define `supportsInterface`, we need to resolve the conflict manually:

```solidity
function supportsInterface(bytes4 interfaceId)
    public pure virtual override(CCIPReceiver, ERC721)
    returns (bool)
{
    return
        interfaceId == type(IERC165).interfaceId ||
        interfaceId == type(IERC721).interfaceId ||
        interfaceId == type(IERC721Metadata).interfaceId ||
        interfaceId == type(IAny2EVMMessageReceiver).interfaceId;
}
```

The contract also stores two extra mappings beyond the standard ERC-721 state:

```solidity
mapping(uint256 => string)  public tokenUriMapping;   // tokenId → URI from source chain
mapping(uint256 => address) public tokenSourceVault;  // tokenId → vault that locked it
```

### Minting: `_ccipReceive`

```solidity
function _ccipReceive(
    Client.Any2EVMMessage memory any2EvmMessage
)
    internal
    override
    onlyAllowlisted(
        any2EvmMessage.sourceChainSelector,
        abi.decode(any2EvmMessage.sender, (address))
    )
{
    address sourceVault = abi.decode(any2EvmMessage.sender, (address));

    (
        string memory tokenUri,
        uint256 tokenId,
        address originalOwner,
    ) = abi.decode(any2EvmMessage.data, (string, uint256, address, address));

    emit MintMessageReceived(
        any2EvmMessage.messageId,
        any2EvmMessage.sourceChainSelector,
        sourceVault,
        tokenId,
        originalOwner
    );

    _mint(originalOwner, tokenId);
    tokenUriMapping[tokenId]   = tokenUri;
    tokenSourceVault[tokenId]  = sourceVault;
}
```

The vault address is read from the CCIP `sender` field — not from the payload — because CCIP populates `sender` from the `msg.sender` of the `ccipSend` call on the source chain. By the time we reach this line the `onlyAllowlisted` modifier has already verified that this sender is the trusted vault. More on why this matters in [the next section](#security-tokensourcevault).

### Burning: `burn`

```solidity
function burn(
    uint256 tokenId,
    address recipient,
    uint64 destinationChainSelector
)
    external
    payable
    onlyAllowlistedDestinationChain(destinationChainSelector)
    validateReceiver(recipient)
{
    if (ownerOf(tokenId) != msg.sender) revert NotOwner(msg.sender, tokenId);

    address vault = tokenSourceVault[tokenId]; // read before destroy

    _burn(tokenId);
    delete tokenUriMapping[tokenId];
    delete tokenSourceVault[tokenId];

    bytes memory payload = abi.encode(tokenId, recipient);

    Client.EVM2AnyMessage memory message = Client.EVM2AnyMessage({
        receiver:     abi.encode(vault),
        data:         payload,
        tokenAmounts: new Client.EVMTokenAmount[](0),
        extraArgs:    Client._argsToBytes(
            Client.GenericExtraArgsV2({ gasLimit: 300_000, allowOutOfOrderExecution: true })
        ),
        feeToken: address(0)
    });

    IRouterClient router = IRouterClient(this.getRouter());
    uint256 fees = router.getFee(destinationChainSelector, message);
    if (msg.value < fees) revert NotEnoughBalance(msg.value, fees);

    bytes32 messageId = router.ccipSend{value: fees}(destinationChainSelector, message);
    emit BurnMessageSent(messageId, destinationChainSelector, vault, tokenId, recipient);

    uint256 refund = msg.value - fees;
    if (refund > 0) {
        (bool ok,) = msg.sender.call{value: refund}("");
        if (!ok) revert RefundFailed(msg.sender, refund);
    }
}
```

Notice that `vault` is read from `tokenSourceVault` *before* the burn — not passed in by the caller. This is intentional and important (see below).

---

## Security: The Allowlist System

Both contracts carry three mappings:

```solidity
mapping(uint64  => bool) public allowlistedDestinationChains;
mapping(uint64  => bool) public allowlistedSourceChains;
mapping(address => bool) public allowlistedSenders;
```

The **outbound guard** (`allowlistedDestinationChains`) is applied as a modifier on `deposit` and `burn`. You cannot send a CCIP message to a chain that has not been explicitly enabled by the owner.

The **inbound guard** (`allowlistedSourceChains` + `allowlistedSenders`) is applied inside `_ccipReceive` via the `onlyAllowlisted` modifier:

```solidity
modifier onlyAllowlisted(uint64 _sourceChainSelector, address _sender) {
    if (!allowlistedSourceChains[_sourceChainSelector])
        revert SourceChainNotAllowlisted(_sourceChainSelector);
    if (!allowlistedSenders[_sender])
        revert SenderNotAllowlisted(_sender);
    _;
}
```

The sender address is ABI-decoded from `any2EvmMessage.sender`, which CCIP encodes as `bytes`. This means:

- `NFTVault._ccipReceive` only processes messages from OP Sepolia's chain selector **and** from the specific `WrappedNFT` contract address.
- `WrappedNFT._ccipReceive` only processes messages from Sepolia's chain selector **and** from the specific `NFTVault` contract address.

> **Without this guard, anyone could craft a CCIP message and either mint wrapped tokens for free or drain the vault.** The allowlist is the trust boundary between the two contracts.

---

## Security: tokenSourceVault

Here is a subtle but important design decision. The `burn` function needs to know which vault to send the unlock message to. The naive approach would be to accept it as a parameter:

```solidity
// BAD — do not do this
function burn(uint256 tokenId, address recipient, address vault, uint64 destinationChain) external { ... }
```

The problem: a malicious user could pass the address of a contract they control as `vault`. The burn message would be delivered there instead of the real vault, and the original NFT would be stuck in the vault forever with no way to recover it.

The fix is to record the vault address at mint time, sourced from the CCIP `sender` field:

```solidity
// _ccipReceive on WrappedNFT — vault address comes from the authenticated CCIP message
address sourceVault = abi.decode(any2EvmMessage.sender, (address));
// ...
tokenSourceVault[tokenId] = sourceVault; // stored as trusted state
```

Because the `onlyAllowlisted` modifier has already verified this sender against the allowlist, `tokenSourceVault` is grounded in the CCIP attestation — not in anything the user touches. When `burn` is called later it reads from this trusted mapping, and the user has no input into where the unlock message goes.

We also `delete tokenSourceVault[tokenId]` as part of the burn — along with `delete tokenUriMapping[tokenId]` — before the CCIP send, following the checks-effects-interactions pattern.

---

## Deployment and Setup

The project uses Foundry. There are two deploy scripts and two setup scripts.

### Deploy

```shell
# Source chain (Sepolia)
forge script script/DeploySource.s.sol --rpc-url $SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY --broadcast

# Destination chain (OP Sepolia)
forge script script/DeployDest.s.sol --rpc-url $OP_SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY --broadcast
```

Each script logs the deployed address. Note them — you need both before running setup.

### Wire up the allowlists

Each contract needs to know the other's address, so the setup scripts must be run after both contracts are deployed:

```shell
# Tell the vault about WrappedNFT and OP Sepolia
forge script script/SetupSource.s.sol --rpc-url $SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY --broadcast \
  --sig "run(address,address)" $NFT_VAULT $WRAPPED_NFT

# Tell WrappedNFT about the vault and Sepolia
forge script script/SetupDest.s.sol --rpc-url $OP_SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY --broadcast \
  --sig "run(address,address)" $WRAPPED_NFT $NFT_VAULT
```

The CCIP chain selectors are centralized in `script/CCIPConfig.sol` so they can be cross-checked and updated in one place:

```solidity
library CCIPConfig {
    uint64  constant SEPOLIA_CHAIN_SELECTOR       = 16015286601757825753;
    address constant SEPOLIA_ROUTER               = 0x0BF3dE8c5D3e8A2B34D2BEeB17ABfCeBaf363A59;

    uint64  constant OP_SEPOLIA_CHAIN_SELECTOR    = 5224473277236331295;
    address constant OP_SEPOLIA_ROUTER            = 0x114A20A10b43D4115e5aeef7345a1A71d2a60C57;
}
```

I deployed this on testnet. The live contracts are:

- **ProviderNFT** (Sepolia): [0xE4BF4837573b7AFeeDA149661A7D4bc6e30A4618](https://sepolia.etherscan.io/address/0xE4BF4837573b7AFeeDA149661A7D4bc6e30A4618)
- **NFTVault** (Sepolia): [0x6A34b2410f6325944f05cEAA087700E0C6aE7C46](https://sepolia.etherscan.io/address/0x6A34b2410f6325944f05cEAA087700E0C6aE7C46)
- **WrappedNFT** (OP Sepolia): [0x04bD0c9C8aa8fC9a9887024C1F3bE2911909D2A4](https://optimism-sepolia.etherscan.io/address/0x04bD0c9C8aa8fC9a9887024C1F3bE2911909D2A4)

---

## Testing End-to-End

I wrote a `verify.sh` shell script that wraps the `cast` calls for the full round trip:

```shell
# Set up env vars once
export SEPOLIA_RPC_URL="https://ethereum-sepolia-public.nodies.app"
export OP_SEPOLIA_RPC_URL="https://optimism-sepolia.gateway.tenderly.co"
export PROVIDER_NFT=0xE4BF4837573b7AFeeDA149661A7D4bc6e30A4618
export NFT_VAULT=0x6A34b2410f6325944f05cEAA087700E0C6aE7C46
export WRAPPED_NFT=0x04bD0c9C8aa8fC9a9887024C1F3bE2911909D2A4
export TOKEN_ID=0
export PRIVATE_KEY="..."

# Full round trip
./script/verify.sh approve        # approve vault to move token #0
./script/verify.sh deposit        # lock + CCIP message to OP Sepolia
# wait for delivery, check https://ccip.chain.link
./script/verify.sh state          # confirm vault owns original, you own wrapped
./script/verify.sh burn           # burn wrapped + CCIP message back to Sepolia
# wait for delivery
./script/verify.sh state          # confirm you own original again
```

After `deposit`, the state looks like this:

```
── Source chain (Sepolia) ──
ProviderNFT owner:    0x6A34b2410f6325944f05cEAA087700E0C6aE7C46  ← vault
ProviderNFT approved: 0x0000000000000000000000000000000000000000

── Destination chain (OP Sepolia) ──
WrappedNFT owner:     0x98A0D43e51bBc9649ab279219cD8ca4c2253850C  ← your wallet
Source vault:         0x6A34b2410f6325944f05cEAA087700E0C6aE7C46
```

You can also verify the URI was bridged correctly:

```shell
cast call $WRAPPED_NFT "tokenURI(uint256)(string)" $TOKEN_ID --rpc-url $OP_SEPOLIA_RPC_URL
```

### A lesson learned: gas limits

My first attempt failed with an out-of-gas error on the destination chain. The CCIP `extraArgs` gas limit was set to the Chainlink example default of 200,000. That turned out to be far too low for the mint path:

- `_mint` involves multiple ERC-721 SSTOREs
- `tokenUriMapping[tokenId] = tokenUri` stores a ~170-character URI string (~6 storage slots ≈ 120k gas)
- `tokenSourceVault[tokenId] = sourceVault` is another SSTORE

The fix was to set the gas limit to **500,000** for the deposit message (mint path) and **300,000** for the burn message (the unlock path only does a `safeTransferFrom` which is much cheaper).

> **Tip:** When setting CCIP gas limits, think about what the *receiver* contract will do, not what the sender does. The gas limit is consumed on the destination chain.

---

## Design Decisions Worth Knowing

**URI bridging vs. on-demand resolution.** Instead of only bridging the token ID and letting the destination chain call back to the source for the URI, we copy the full `tokenURI` string into the CCIP payload. This makes `tokenURI()` work instantly on the destination with no additional cross-chain calls, at the cost of a larger payload for tokens with long URIs.

**`allowOutOfOrderExecution: true`.** This opts out of CCIP's default sequencing guarantee. It is safe here because each token can only have one in-flight message at a time: a second deposit is impossible while the NFT is locked in the vault, and a second burn is impossible after the wrapped token is destroyed. Allowing out-of-order execution reduces latency and cost when multiple different tokens are bridged concurrently.

**NFT locked before CCIP send.** The `safeTransferFrom` in `deposit` happens *before* `ccipSend`. This means the vault holds the NFT the instant the Sepolia transaction is mined, with no window where the user could move the token after approval but before the cross-chain message is dispatched.

**The source NFT contract address travels with the message but is currently discarded.** The deposit payload includes `address(nft_contract)` as a fourth field that `WrappedNFT` decodes as `_` and ignores. It is there for forward compatibility — a future version that wants to track which source collection each wrapped token came from can do so without changing the payload format.

---

## Conclusion

We built a complete lock-and-mint NFT bridge with two contracts, CCIP as the transport layer, and no custom relayer. The full round trip — lock on Sepolia, mint on OP Sepolia, burn on OP Sepolia, unlock on Sepolia — works end-to-end on testnet.

The most important security properties:
- The allowlist ensures only the trusted counterpart contract on the trusted chain can trigger state changes
- `tokenSourceVault` ensures the burn message always targets the real vault, not an attacker-supplied address

A production version would need a few more things: an emergency unlock mechanism for failed CCIP deliveries, support for multiple NFT collections per vault, governance over the allowlist, and probably an upgrade path. But as a foundation for understanding how cross-chain NFT bridging works in practice, this is a solid starting point.

The full source code is available at [github.com/jacekv/nft_bridging](https://github.com/jacekv/nft_bridging).

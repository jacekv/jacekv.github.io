---
layout: post
title: "Token Approvals in Ethereum: From ERC20 to Permit2"
tags: Solidity ERC20 Approval Permit Permit2 EIP2612
---

# Token Approvals in Ethereum: From ERC20 to Permit2

## Table of contents

1. [Introduction](#introduction)
    1. [Why Token Approvals Matter](#why-token-approvals-matter)
    2. [From Problem to Evolution](#from-problem-to-evolution)
2. [ERC20 token approval](#erc20-token-approval)
3. [EIP2612 or Permit](#eip2612-or-permit)
    1. [Implementing `permit` function in ERC20](#implementing-permit-function-in-erc20)
4. [Permit2](#permit2)
    1. [Integrating Permit2 into a contract](#integrating-permit2-into-a-contract)
    2. [PermitWitnessTransferFrom](#permitwitnesstransferfrom)
    3. [Nonces in Permit2](#nonces-in-permit2)
5. [Real-world implementation examples](#real-world-implementation-examples)
6. [Recommendation for new ERC20 Token developers](#recommendation-for-new-erc20-token-developers)
7. [Conclusion](#conclusion)


## Introduction <a name="introduction"></a>

Imagine you want to swap tokens on Uniswap, deposit assets into Aave, or buy an
NFT on OpenSea. In each case, you'll notice something common: before executing
the main transaction, you need to approve the platform to use your tokens.
This two-step process is a fundamental part of the Ethereum ecosystem, but it's
also one of its biggest friction points.

### Why Token Approvals Matter <a name="why-token-approvals-matter"></a>

Token approvals are the mechanism that allows smart contracts to access and
transfer your tokens. This capability is crucial for three main reasons:

1. **Security**: You maintain custody of your assets while granting limited permissions to specific contracts
2. **Composability**: Different protocols can interact with your tokens, enabling the "money legos" of DeFi
3. **Flexibility**: You control who can access your tokens and how much they can use

However, the standard approval method has significant drawbacks.
The average user wastes gas on multiple approval transactions, often grants
excessive permissions, and creates security vulnerabilities from forgotten approvals.

### From Problem to Evolution <a name="from-problem-to-evolution"></a>

In this post, we'll trace the evolution of token approval mechanisms.

![Evolution of Token Approvals](/assets/images/ERC20_Approval_Evolution.png)
*The progression from basic approvals to modern signature-based approaches*

First, we'll explore the traditional ERC20 `approve` function and understand its
limitations. Then we'll look at how EIP2612's `permit` function improved the user
experience by eliminating separate approval transactions.
Finally, we'll dive into Uniswap's `Permit2` contract, which brings
these benefits to all tokens while adding powerful new features.

By the end of this article, you'll understand how token approvals work under
the hood, how to implement each approach, and which method best suits different
use cases. Whether you're a developer integrating with tokens or a user wanting
to understand what you're signing, this knowledge will help you interact more
safely and efficiently with the Ethereum ecosystem.

Let's start with the basics: how does the standard ERC20 approval mechanism work?


## ERC20 token approval <a name="erc20-token-approval"></a>

If you ever used a DEX to swap tokens or provided liquidity, you probably had to
sign 2 transactions. The first transaction is the `approve` transaction, in which
you approve the DEX contract to spend your tokens. Usually it comes with an
amount, which is most of the time set to the maximum, which is 
`2**256 - 1` or `115792089237316195423570985008687907853269984665640564039457584007913129639936`.
This means, you allow the DEX contract to spend up to `2**256 - 1` tokens on your
behalf.

The second transaction is the actual transaction that swaps or deposits your tokens
into the DEX contract.

Here is a visualization of the process:

![ERC20 Approve](/assets/images/ERC20_Approve.png)

1. Alice calls `approve()` on an ERC20 to grant the protocol contract allowance to spend tokens on her behalf
2. Alice calls an interaction function on the protocol contract, which in turn calls `transferFrom()` on the ERC20 token contract, moving Alice tokens.

This model works well for many use cases, since the protocol contract has
long-term access to the user's tokens. But it has three well-known issues:

1. Bad UX: Users must approve every new protocol on each token they intend to use with it, and this is almost always a separate transaction.
2. Bad security: Applications often ask for unlimited allowances to avoid having to repeat the above UX issue. This means that if the protocol ever gets exploited, every user's token that they've approved the protocol to spend can potentially be taken right out of their wallets.
3. Gas costs: Users have to pay for two transactions, which can be expensive and slow.

Since every interaction with a protocol requires 2 transactions, approve and the initial interaction,
and requires the users to hold Ether to pay for those transactions, much research has been done to
improve the user experience, one of which is EIP2612's `permit` function.

## EIP2612 or Permit <a name="eip2612-or-permit"></a>
EIP2612 is an extension of the ERC20 standard, which adds 3 new functions to the ERC20
standard: `permit`, `nonces`, and `DOMAIN_SEPARATOR`.

The `permit` function allows users to approve a third-party contract to spend their tokens
on their behalf by signing a message with their private key. This message ad signature are then
sent to the third-party contract, forwards the message and signature to `permit` function 
of the ERC20 contract. If the signature is valid, the ERC20 contract approves the 
third-party contract to spend the specified amount of tokens on behalf of the user.

The `nonces` function returns the current nonce of the user, which is used to prevent replay attacks.

The `DOMAIN_SEPARATOR` function returns the domain separator of the token, which is used to
sign the message. The domain separator is a unique identifier for the token and is used to
prevent replay attacks across different tokens. More about it can be found in
[EIP712](https://eips.ethereum.org/EIPS/eip-712).

Here the flow visualized with a diagram using EIP2612's `permit` function:

![ERC20 Permit](/assets/images/ERC20_Permit.png)

1. Alice signs an off-chain "permit" message, signaling that she wishes to grant a contract an allowance to spend a (EIP-2612) token. Based on the UX,
    a. Alice submits the signed message as part of her interaction with said contract.
    b. Alice submits the signed message to a relayer, which then submits the signed message as part of her interaction with said contract, therefore paying the gas fees.
3. The contract calls `permit()` on the token, which consumes the permit message and signature, granting the contract an allowance.
4. The contract now has an allowance so it can call `transferFrom()` on the token, moving tokens held by Alice.

This solves both issues from the conventional ERC20 approve approach:

1. The user never has to submit a separate approve() transaction.
2. There is no longer a necessary evil of dangling allowances since permit messages grant an instantaneous allowance that is often spent right away. These messages can also choose a more reasonable allowance amount and, more importantly, an expiration time on when the permit message can be consumed.

But the reality is that the `permit` function is not available on all ERC20 tokens.
It can only be added to new or upgradeable tokens. This means that most of the existing
ERC20 tokens do not support the `permit` function. This is a major limitation of the
`permit` function, as it cannot be used with existing tokens.

To overcome this limitation, Uniswap introduced the `Permit2` contract, which allows
users to approve multiple tokens at once and use the `permit` function on any ERC20 token.
But before we dive into `Permit2`, let's take a look at the `permit` function
in detail, by implementing it ourselves.

### Implementing `permit` function in ERC20 <a name="implementing-permit-function-in-erc20"></a>

Here is a simple implementation of the `permit` function in an ERC20 token contract:

```solidity
pragma solidity >=0.8.4;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/EIP712.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Nonces.sol";

contract Test is ERC20, EIP712, Nonces {
    bytes32 private constant PERMIT_TYPEHASH =
        keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");

    error ExpiredSignature(uint256 deadline);
    error InvalidSigner(address signer, address owner);

    constructor(string memory name, string memory symbol, string memory version) EIP712(name, version) ERC20(name, symbol) {
        _mint(msg.sender, 1337);
    }

    // as defined in EIP2612
    function DOMAIN_SEPARATOR() external view returns (bytes32) {
        return _domainSeparatorV4();
    }

    function nonces(address owner) public view virtual override(Nonces) returns (uint256) {
        return super.nonces(owner);
    }

    function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
        if (block.timestamp > deadline) {
            revert ExpiredSignature(deadline);
        }

        bytes32 structHash = keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline));

        bytes32 hash = _hashTypedDataV4(structHash);

        address signer = ECDSA.recover(hash, v, r, s);
        if (signer != owner) {
            revert InvalidSigner(signer, owner);
        }

        _approve(owner, spender, value);
    }
}
```

You can copy the code and paste the code into Remix and deploy it.

Since EIP2612 is an extension of the ERC20 standard and EIP712, we are importing
the `ERC20`, `EIP712`, and `ECDSA` contracts from OpenZeppelin. And let them
handle the heavy lifting for us. We are also importing the `Nonces` contract
from OpenZeppelin, which is used to manage the nonces of the users. This is
important to prevent replay attacks.

Bear in mind that the `Nonces` contract offers a sequential nonce for each user.
This might be a limitation if lots of third-party contracts are using the `permit` function
and the user is using them at the same time. In this case, some transaction might fail
due to the nonce being used already.

Now let's take a look at the `permit` function in detail:
1. The function takes the following parameters:
   - `owner`: The address of the user who is granting the allowance.
   - `spender`: The address of the contract that is being granted the allowance.
   - `value`: The amount of tokens that are being approved.
   - `deadline`: The timestamp until when the permit message is valid.
   - `v`, `r`, `s`: The signature of the user, which is used to verify the message.
2. The function checks if the `deadline` has expired. If it has, it reverts the transaction.
3. The function creates a hash of the permit message using the `keccak256` function.
4. The function hashes the message using the `_hashTypedDataV4` function, which is
   part of the `EIP712` contract. This function adds the domain separator to the hash.
5. The function recovers the signer of the message using the `ECDSA.recover` function.
6. The function checks if the signer is the same as the `owner`. If it is not, it reverts
   the transaction.
7. The function calls the `_approve` function to approve the `spender` to spend
    the specified amount of tokens on behalf of the `owner`.

But how do we sign the message? Here you go with a simple Python script that
signs the message and submits it to the `permit` function:

```python
from web3 import Web3
import web3

ERC20_PERMIT_ABI =[{}] # copy contract to remix and extract ABI otherwise its too long

_PERMIT_MESSAGE_TYPES = {
    'Permit': [{
        'name': 'owner',
        'type': 'address'
    }, {
        'name': 'spender',
        'type': 'address'
    }, {
        'name': 'value',
        'type': 'uint256'
    }, {
        'name': 'nonce',
        'type': 'uint256'
    }, {
        'name': 'deadline',
        'type': 'uint256'
    }]
}

RPC_URL = "RPC URL"
PRIVATE_KEY_TOKEN_OWNER = 'private key of the token owner'
PRIVATE_KEY_SENDER = 'private key of the sender'

ERC20PERMIT_CONTRACT_ADDRESS = "address of contract deployed"
TOKEN_OWNER_ADDRESS = w3.eth.account.from_key(PRIVATE_KEY_TOKEN_OWNER)
SENDER_ADDRESS = w3.eth.account.from_key(PRIVATE_KEY_SENDER)

DOMAIN_DATA = {
    "name": "<NAME OF TOKEN>",
    "version": "1",
    "chainId": <chain id as int>, 
    "verifyingContract": ERC20PERMIT_CONTRACT_ADDRESS,
}

w3 = Web3(Web3.HTTPProvider(RPC_URL))


erc20_permit_contract = w3.eth.contract(address=ERC20PERMIT_CONTRACT_ADDRESS, abi=ERC20_PERMIT_ABI)
value_to_spend = 1

latest_block = w3.eth.get_block('latest')
deadline = latest_block['timestamp'] + 1000000

msg_data = {
    'owner': TOKEN_OWNER_ADDRESS,
    'spender': SENDER_ADDRESS,
    'value': value_to_spend,
    'nonce': erc20_permit_contract.functions.nonces(TOKEN_OWNER).call(),
    'deadline': deadline
}

signed_msg = web3.Account.sign_typed_data(PRIVATE_KEY_TOKEN_OWNER, DOMAIN_DATA, _PERMIT_MESSAGE_TYPES, msg_data)
signature = signed_msg.signature.hex()
r = Web3.to_bytes(hexstr=f"0x{signature[0:64]}")
s = Web3.to_bytes(hexstr=f"0x{signature[64:128]}")
v = int(signature[128:130], 16)

print(f"Balance of {TOKEN_OWNER}: {erc20_permit_contract.functions.balanceOf(TOKEN_OWNER).call()}")
print(f"Allowance of {TOKEN_OWNER}: {erc20_permit_contract.functions.allowance(TOKEN_OWNER, spender_address).call()}")

print("Calling permit function...")
tx_nonce = w3.eth.get_transaction_count(SENDER_ADDRESS)
tx = erc20_permit_contract.functions.permit(
    TOKEN_OWNER, SENDER_ADDRESS, value_to_spend, deadline, v, r, s
).build_transaction({"from": SENDER_ADDRESS, "nonce": tx_nonce})

signed_tx = w3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY_SENDER)

tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)

print(f"Transaction hash: {tx_hash.hex()}")
print("Waiting for transaction receipt...")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

print(f"Balance of {TOKEN_OWNER}: {erc20_permit_contract.functions.balanceOf(TOKEN_OWNER).call()}")
print(f"Allowance of {TOKEN_OWNER}: {erc20_permit_contract.functions.allowance(TOKEN_OWNER, sender_address).call()}")
```

This script uses the `web3.py` library to interact with the Ethereum blockchain.
It signs the permit message and sends it to the `permit` function of the ERC20
contract. The script also prints the balance and allowance of the token owner
before and after the transaction. You can run this script in your local environment
but you will notice that the balance won't change, since we are not calling the
`transferFrom` function after the `permit` function. You can do that by
adding the `transferFrom` function call after the `permit` function call in
the script.

And that's it! You have successfully implemented the `permit` function
in an ERC20 token contract and signed the message using Python. You can now
use the `permit` function to approve a third-party contract to spend your
tokens on your behalf without having to submit a separate approve() transaction.

Let's move on to the `Permit2` contract, which is adding the `permit` functionality
to any ERC20 token, even if it does not implement the `permit` function.

## Permit2 <a name="permit2"></a>

Permit2 is a token approval contract that iterates on existing token approval
mechanisms by introducing signature-based approvals and transfers for any
ERC20 token, regardless of EIP-2612 support.

For each token, users have to submit a one-time traditional approval that
sets Permit2 as an approved spender. Contracts that integrate with Permit2
can "borrow" the spender status for any token a user has approved on the
canonical Permit2 contract. Any integrated contract can temporarily become
a spender with a signed offchain Permit2 message. This effectively "shares"
the one-time traditional token approval and skips the individual approval
transaction for every protocol that supports Permit2 -- until the Permit2
message expires. It's a major UX upgrade with additional safety bumpers
like expiring approvals and the lockdown feature.

If you would like to learn more about the `Permit2` contract, I highly recommend
reading the [Uniswap documentation](https://docs.uniswap.org/contracts/permit2/overview) and
check out the [Github](https://github.com/Uniswap/permit2/tree/main) repository. 

Here are some features of the Permi2 contract:

 1. Permits for any token. Applications can have a single transaction flow by 
    sending a signature along with the transaction data for any token,
    including those not supporting a native permit method.
 2. Expiring approvals. Approvals can be time-bound, removing security
    concerns around hanging approvals on a wallet's entire token balance.
    Revoking approvals do not necessarily have to be a new transaction.
 3. Signature-based transfers. Users can bypass setting allowances
    entirely by releasing tokens to a permissioned spender through a one-time signature.
 4. Batch approvals and transfers. Users can set approvals on multiple
    tokens or execute multiple transfers with one transaction.
 5. Batch revoking allowances. Remove allowances on any number of tokens and
    spenders in one transaction.

Before we dive into the code and see how to integrate the `Permit2` contract
into our project, let's take a look at the flow of the `Permit2` contract:

![Permit2](/assets/images/ERC20_Permit2.png)

1. Alice calls `approve()` on an ERC20 to grant an infinite allowance to the canonical Permit2 contract.
2. Alice signs an off-chain `permit2` message that signals that the protocol contract is allowed to transfer tokens on her behalf.
3. The next step will vary depending on UX choices:
   
   a. In the simple case, Alice can just submit a transaction herself, including the signed `permit2` message as part of an interaction with the protocol contract.
   
   b. If the protocol contract allows it, Alice can transmit her signed `permit2` message to a relayer service that will submit interaction the transaction on Alice's behalf.
4. The protocol contract calls `permitTransferFrom()` on the Permit2 contract, which in turn uses its allowance (granted in 1.) to call `transferFrom()` on the ERC20 contract, moving the tokens held by Alice.


It might seem like a regression to require the user to grant an explicit allowance first. But rather than granting it to the protocol directly, the user will instead grant it to the canonical Permit2 contract. This means that if the user has already done this before, say to interact with another protocol that integrated Permit2, every other protocol can skip that step.

Instead of directly calling `transferFrom()` on the ERC20 token to perform a transfer, a protocol will call `permitTransferFrom()` on the canonical Permit2 contract. Permit2 sits between the protocol and the ERC20 token, tracking and validating `permit2` messages, then ultimately using its allowance to perform the `transferFrom()` call directly on the ERC20. This indirection is what allows Permit2 to extend EIP-2612-like benefits to every existing ERC20 token!

Also, like EIP-2612 permit messages, `permit2` messages expire to limit the the attack window of an exploit. It's much also easier to secure the small Permit2 contract than the contracts of individual DeFi protocols, so having an infinite allowance there is less of a concern.

Great, now that we have an understanding of the `Permit2` flow and how it works,
let's build a simple contract that integrates the `Permit2` contract and allows
us to transfer tokens using the `permitTransferFrom()` function.

### Integrating Permit2 into a contract <a name="integrating-permit2-into-a-contract"></a>

Here is a simple implementation of a contract that integrates the `Permit2` contract
and allows us to transfer tokens using the `permitTransferFrom()` function:

```solidity
pragma solidity ^0.8.23;

// DO NOT USE IN PRODUCTION :!:
contract Permit2Vault {
    bool private _reentrancyGuard;
    // The canonical permit2 contract.
    IPermit2 public immutable PERMIT2;
    // User -> token -> deposit balance
    mapping (address => mapping (IERC20 => uint256)) public tokenBalancesByUser;

    constructor(IPermit2 permit_) {
        PERMIT2 = permit_;
    }

    // Deposit some amount of an ERC20 token from the caller
    // into this contract using Permit2.
    function depositERC20(
        IERC20 token,
        uint256 amount,
        uint256 nonce,
        uint256 deadline,
        bytes calldata signature
    ) external {
        // Credit the caller.
        tokenBalancesByUser[msg.sender][token] += amount;
        // Transfer tokens from the caller to ourselves.
        PERMIT2.permitTransferFrom(
            IPermit2.PermitTransferFrom({
                permitted: IPermit2.TokenPermissions({
                    token: token,
                    amount: amount
                }),
                nonce: nonce,
                deadline: deadline
            }),
            IPermit2.SignatureTransferDetails({
                to: address(this),
                requestedAmount: amount
            }),
            msg.sender,
            signature
        );
    }

    // Return ERC20 tokens deposited by the caller.
    function withdrawERC20(IERC20 token, uint256 amount) external {
        tokenBalancesByUser[msg.sender][token] -= amount;
        //In production, use an ERC20 compatibility library to
        // execute this transfer to support non-compliant tokens.
        token.transfer(msg.sender, amount);
    }
}

// Minimal Interface of the Permi2 Contract
interface IPermit2 {
    // Token and amount in a permit message.
    struct TokenPermissions {
        // Token to transfer.
        IERC20 token;
        // Amount to transfer.
        uint256 amount;
    }

    // The permit2 message.
    struct PermitTransferFrom {
        // Permitted token and maximum amount.
        TokenPermissions permitted;// deadline on the permit signature
        // Unique identifier for this permit.
        uint256 nonce;
        // Expiration for this permit.
        uint256 deadline;
    }

    // Transfer details for permitTransferFrom().
    struct SignatureTransferDetails {
        // Recipient of tokens.
        address to;
        // Amount to transfer.
        uint256 requestedAmount;
    }

    // Consume a permit2 message and transfer tokens.
    function permitTransferFrom(
        PermitTransferFrom calldata permit,
        SignatureTransferDetails calldata transferDetails,
        address owner,
        bytes calldata signature
    ) external;
}

// Minimal ERC20 interface.
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}
```

Here we have a simple `Permit2Vault` contract that uses Signature Transfers of
the `Permit2` contract to transfer tokens on behalf of the user.
SignatureTransfer is a one-time permit for token transfers.

Spenders are permitted to transfer tokens, but only in the same transaction
where the signature is spent. To transfer funds again, you would need to
request this signature again. The signature transfer method is great for
applications that infrequently need tokens from users and want to ensure
tighter restrictions on approvals. Persistent limits and expiries are not
stored and never leave "hanging approvals." 
SignatureTransfer also lets you do a more advanced integration with Permit2
through `permitWitnessTransferFrom`, which we will have a look into in a bit.

The `Permit2Vault` contract allows users to deposit and withdraw ERC20 tokens
using the `permitTransferFrom()` function of the `Permit2` contract. The
contract keeps track of the user's token balances in a mapping and allows
users to withdraw their tokens at any time. This transfer is based on signatures
and is called SignatureTransfer in the `Permit2` contract

Let's have a look at the Python code to sign the `PermitTransferFrom` message
and submit it to the `depositERC20()` function of the `Permit2Vault` contract,
which ultimately calls the `permitTransferFrom()` function of the `Permit2` contract:

```python
from web3 import Web3
import web3

PRIVATE_KEY_TOKEN_OWNER = 'private key of the token owner'
PRIVATE_KEY_SENDER = 'private key of the sender'
RPC_URL = "RPC URL"

w3 = Web3(Web3.HTTPProvider(RPC_URL))
token_owner_account = w3.eth.account.from_key(PRIVATE_KEY_OWNER)
TOKEN_OWNER_ADDRESS = token_owner_account.address

sender_account = w3.eth.account.from_key(PRIVATE_KEY_SENDER)
SENDER_ADDRESS = sender_account.address

ERC20_CONTRACT_ADDRESS = "address of your ERC20 contract"
PERMIT2_CONTRACT_ADDRESS = "address of the Permit2 contract"
PERMIT2_VAULT_CONTRACT_ADDRESS = "address of the Permit2Vault contract"
VALUE_TO_SPEND = 10

ERC20_ABI = [] # copy contract to remix and extract ABI otherwise its too long
PERMIT2_VAULT_ABI = [] # copy contract to remix and extract ABI otherwise its too long

ERC20_CONTRACT = w3.eth.contract(address=ERC20_CONTRACT_ADDRESS, abi=ERC20_ABI)
PERMIT2_VAULT_CONTRACT = w3.eth.contract(address=PERMIT2_VAULT_CONTRACT_ADDRESS, abi=PERMIT2_VAULT_ABI)

# all types are defined here: https://github.com/Uniswap/permit2/blob/main/src/libraries/PermitHash.sol
_PERMIT2_TRANSFER_FROM_MESSAGE_TYPES = {
    'TokenPermissions': [{
        'name': 'token',
        'type': 'address'
    }, {
        'name': 'amount',
        'type': 'uint256'
    }],
    'PermitTransferFrom': [{
        'name': 'permitted',
        'type': 'TokenPermissions'
    }, {
        'name': 'spender',
        'type': 'address'
    }, {
        'name': 'nonce',
        'type': 'uint256'
    }, {
        'name': 'deadline',
        'type': 'uint256'
    }]
}

DOMAIN_DATA = {
    "name": "Permit2",
    "chainId": <chain id as int>,, 
    "verifyingContract": PERMIT2_CONTRACT_ADDRESS,
}

print("Checking allowance for Permit2 contract...")
allowance = ERC20_CONTRACT.functions.allowance(TOKEN_OWNER_ADDRESS, PERMIT2_CONTRACT_ADDRESS).call()
if allowance == 0:
    print("Granting approval for Permit2 contract as token owner")
    tx_nonce = w3.eth.get_transaction_count(TOKEN_OWNER_ADDRESS)
    tx = ERC20_CONTRACT.functions.approve(
        PERMIT2_CONTRACT_ADDRESS, 1 * 10**18
    ).build_transaction({"from": TOKEN_OWNER_ADDRESS, "nonce": tx_nonce})
    signed_tx = w3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY_TOKEN_OWNER)
    tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
    print(f"Transaction hash: {tx_hash.hex()}")
    print("Waiting for transaction receipt...")
    tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    print("Transaction included. Waiting for a few more confirmations")
    tx_block_number = tx_receipt['blockNumber']
    latest_block = w3.eth.get_block('latest')

    while latest_block['number'] <= tx_block_number + 3:
        print(f"Waiting for block {latest_block['number']}...")
        latest_block = w3.eth.get_block('latest')
else:
    print("Permit2 contract already approved.\n")

print("ERC20 Token Information:")
print(f"    Allowance of Permit2 for {TOKEN_OWNER_ADDRESS}: {ERC20_CONTRACT.functions.allowance(TOKEN_OWNER_ADDRESS, PERMIT2_CONTRACT_ADDRESS).call()}")
print(f"    Allowance of Permit2Vault for {TOKEN_OWNER_ADDRESS}: {ERC20_CONTRACT.functions.allowance(TOKEN_OWNER_ADDRESS, PERMIT2_VAULT_CONTRACT_ADDRESS).call()}")

latest_block = w3.eth.get_block('latest')
deadline = latest_block['timestamp'] + 1000000  # 1 million seconds from now

NONCE = 0

msg_data = {
    'permitted': {
        'token': ERC20_CONTRACT_ADDRESS,
        'amount': VALUE_TO_SPEND
    },
    'spender': PERMIT2_VAULT_CONTRACT_ADDRESS,
    'nonce': NONCE,
    'deadline': deadline
}

signed_msg = web3.Account.sign_typed_data(PRIVATE_KEY_TOKEN_OWNER, DOMAIN_DATA, _PERMIT2_TRANSFER_FROM_MESSAGE_TYPES, msg_data)
signature = signed_msg.signature.to_0x_hex()

tx_nonce = w3.eth.get_transaction_count(TOKEN_OWNER_ADDRESS)
tx = PERMIT2_VAULT_CONTRACT.functions.depositERC20(
    ERC20_CONTRACT_ADDRESS, VALUE_TO_SPEND, NONCE, deadline, signature
).build_transaction({"from": TOKEN_OWNER_ADDRESS, "nonce": tx_nonce})
signed_tx = w3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY_TOKEN_OWNER)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
print("Waiting for transaction receipt...")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
print("Transaction included. Waiting for a few more confirmations")
tx_block_number = tx_receipt['blockNumber']
latest_block = w3.eth.get_block('latest')

while latest_block['number'] <= tx_block_number + 2:
    latest_block = w3.eth.get_block('latest')

print()
print("ERC20 Token Information:")
print(f"    Name: {ERC20_CONTRACT.functions.name().call()}")
print(f"    Symbol: {ERC20_CONTRACT.functions.symbol().call()}")
print(f"    Decimals: {ERC20_CONTRACT.functions.decimals().call()}")
print(f"    Total Supply: {ERC20_CONTRACT.functions.totalSupply().call()}")
print(f"    Balance of {TOKEN_OWNER}: {ERC20_CONTRACT.functions.balanceOf(TOKEN_OWNER).call()}")
print(f"    Balance of Permit2Vault: {ERC20_CONTRACT.functions.balanceOf(PERMIT2_VAULT_CONTRACT_ADDRESS).call()}")
print(f"    Allowance of Permit2 for {TOKEN_OWNER}: {ERC20_CONTRACT.functions.allowance(TOKEN_OWNER, PERMIT2_CONTRACT_ADDRESS).call()}")
print(f"    Allowance of Permit2Vault for {TOKEN_OWNER}: {ERC20_CONTRACT.functions.allowance(TOKEN_OWNER, PERMIT2_VAULT_CONTRACT_ADDRESS).call()}")
print()
```

And that's it! You have successfully integrated the `Permit2` contract
into your project and used the `permitTransferFrom()` function to transfer
tokens on behalf of the user. You can now deploy the `Permit2Vault` multiple
times and interact with the same ERC20 token without having to approve
the `Permit2` contract again. This is a major improvement over the
traditional ERC20 `approve` function and the EIP2612 `permit` function.

The `Permit2` contract also allows you to batch transfers, which we are not
going to cover in this post. But you can check out the [Github](https://github.com/Uniswap/permit2/tree/main)
repository on how to do that.

But there is one more feature I would like to get into and that is the
`permitWitnessTransferFrom()` function of the `Permit2` contract. In `permitTransferFrom()`
the user is signing the predefined message `PermitTransferFrom` with it's 
attributes `((token, amount), spender, nonce, deadline)`.

The `permitWitnessTransferFrom()` function allows the user to attach custom
witness data to the `PermitTransferFrom` message. This is useful for
protocols that want to attach additional custom message data and let the
`Permit2` contract verify the message.

### PermitWitnessTransferFrom <a name="permitwitnesstransferfrom"></a>

Let's start with having a look into the Solidity code which integrates the
`permitWitnessTransferFrom()` function into our `Permit2Vault` contract:

```solidity
pragma solidity ^0.8.23;
contract Permit2Vault {
    // The canonical permit2 contract.
    IPermit2 public immutable PERMIT2;

    struct ExampleTrade {
        address exampleTokenAddress;
        uint256 exampleMinimumAmountOut;
    }

    string private constant WITNESS_TYPE_STRING = "ExampleTrade witness)ExampleTrade(address exampleTokenAddress,uint256 exampleMinimumAmountOut)TokenPermissions(address token,uint256 amount)";
    bytes32 private constant WITNESS_TYPEHASH = keccak256("ExampleTrade(address exampleTokenAddress,uint256 exampleMinimumAmountOut)");

    constructor(IPermit2 permit_) {
        PERMIT2 = permit_;
    }

    // Deposit some amount of an ERC20 token from the caller
    // into this contract using Permit2.
    function deposit(
        IERC20 _token,
        uint256 _amount,
        uint256 _nonce,
        uint256 _deadline,
        bytes32 _witness,
        address _owner,
        bytes calldata _signature
    ) external {
        PERMIT2.permitWitnessTransferFrom(
            IPermit2.PermitTransferFrom({
                permitted: IPermit2.TokenPermissions({
                    token: _token,
                    amount: _amount
                }),
                nonce: _nonce,
                deadline: _deadline
            }),
            IPermit2.SignatureTransferDetails({
                to: address(this),
                requestedAmount: _amount
            }),
            _owner,
            // witness
            _witness,
            // witnessTypeString,
            WITNESS_TYPE_STRING,
            _signature
        );
    }
}

// Minimal Interface of the Permit2 Contract
interface IPermit2 {
    // Token and amount in a permit message.
    struct TokenPermissions {
        // Token to transfer.
        IERC20 token;
        // Amount to transfer.
        uint256 amount;
    }

    // The permit2 message.
    struct PermitTransferFrom {
        // Permitted token and maximum amount.
        TokenPermissions permitted;// deadline on the permit signature
        // Unique identifier for this permit.
        uint256 nonce;
        // Expiration for this permit.
        uint256 deadline;
    }

    // Transfer details for permitTransferFrom().
    struct SignatureTransferDetails {
        // Recipient of tokens.
        address to;
        // Amount to transfer.
        uint256 requestedAmount;
    }

    function permitWitnessTransferFrom(
        PermitTransferFrom memory permit,
        SignatureTransferDetails calldata transferDetails,
        address owner,
        bytes32 witness,
        string calldata witnessTypeString,
        bytes calldata signature
    ) external;
}

// Minimal ERC20 interface.
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}
```

In the contract we see that we defined a custom message type, `ExampleTrade`,
which has two attributes: `exampleTokenAddress` and `exampleMinimumAmountOut`.

Now we want to sign the message as a user and submit it to the `deposit()`
function of the `Permit2Vault` contract. Here is the Python code to do that:

```python
import eth_abi
from web3 import Web3
import web3


PRIVATE_KEY_TOKEN_OWNER = 'private key of the token owner'
PRIVATE_KEY_SENDER = 'private key of the sender'
RPC_URL = "RPC URL"

TOKEN_OWNER_ACCOUNT = w3.eth.account.from_key(PRIVATE_KEY_TOKEN_OWNER)
TOKEN_OWNER_ADDRESS = TOKEN_OWNER_ACCOUNT.address
ERC20_CONTRACT_ADDRESS = "address of your erc20 contract"
PERMIT2_CONTRACT_ADDRESS = "address of the Permit2 contract"
PERMIT2_WITNESS_CONTRACT_ADDRESS = "address of the Permit2Vault contract"

w3 = Web3(Web3.HTTPProvider(RPC_URL))

VALUE_TO_SPEND = 1
PERMIT2WITNESS_VAULT_ABI =[
	{
		"inputs": [
			{
				"internalType": "contract IPermit2",
				"name": "permit_",
				"type": "address"
			}
		],
		"stateMutability": "nonpayable",
		"type": "constructor"
	},
	{
		"inputs": [],
		"name": "PERMIT2",
		"outputs": [
			{
				"internalType": "contract IPermit2",
				"name": "",
				"type": "address"
			}
		],
		"stateMutability": "view",
		"type": "function"
	},
	{
		"inputs": [
			{
				"internalType": "contract IERC20",
				"name": "_token",
				"type": "address"
			},
			{
				"internalType": "uint256",
				"name": "_amount",
				"type": "uint256"
			},
			{
				"internalType": "uint256",
				"name": "_nonce",
				"type": "uint256"
			},
			{
				"internalType": "uint256",
				"name": "_deadline",
				"type": "uint256"
			},
			{
				"internalType": "bytes32",
				"name": "witness",
				"type": "bytes32"
			},
			{
				"internalType": "address",
				"name": "_owner",
				"type": "address"
			},
			{
				"internalType": "bytes",
				"name": "_signature",
				"type": "bytes"
			}
		],
		"name": "deposit",
		"outputs": [],
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"inputs": [
			{
				"internalType": "address",
				"name": "",
				"type": "address"
			},
			{
				"internalType": "contract IERC20",
				"name": "",
				"type": "address"
			}
		],
		"name": "tokenBalancesByUser",
		"outputs": [
			{
				"internalType": "uint256",
				"name": "",
				"type": "uint256"
			}
		],
		"stateMutability": "view",
		"type": "function"
	}
]

PERMIT2_ABI = [{
		"inputs": [
			{
				"internalType": "address",
				"name": "",
				"type": "address"
			},
			{
				"internalType": "uint256",
				"name": "",
				"type": "uint256"
			}
		],
		"name": "nonceBitmap",
		"outputs": [
			{
				"internalType": "uint256",
				"name": "",
				"type": "uint256"
			}
		],
		"stateMutability": "view",
		"type": "function"
	}
]

ERC20_ABI = [] # copy contract to remix and extract ABI otherwise its too long

ERC20_CONTRACT = w3.eth.contract(address=ERC20_CONTRACT_ADDRESS, abi=ERC20_ABI)
PERMIT2_WITNESS_VAULT_CONTRACT = w3.eth.contract(address=PERMIT2_WITNESS_CONTRACT_ADDRESS, abi=PERMIT2WITNESS_VAULT_ABI)
PERMIT2_CONTRACT = w3.eth.contract(address=PERMIT2_CONTRACT_ADDRESS, abi=PERMIT2_ABI)

WITNESS_TYPE_STRING = "ExampleTrade(address exampleTokenAddress,uint256 exampleMinimumAmountOut)"
WITNESS_TYPEHASH = w3.solidity_keccak(['string'], [WITNESS_TYPE_STRING])

DOMAIN_DATA = {
    "name": "Permit2",
    "chainId": <chain id as int>, 
    "verifyingContract": PERMIT2_CONTRACT_ADDRESS,
}


_PERMIT2_TRANSFER_FROM_WITNESS_MESSAGE_TYPES = { 
    'TokenPermissions': [{
        'name': 'token',
        'type': 'address'
    }, {
        'name': 'amount',
        'type': 'uint256'
    }],
    'ExampleTrade': [{
        'name': 'exampleTokenAddress',
        'type': 'address'
    }, {
        'name': 'exampleMinimumAmountOut',
        'type': 'uint256'
    }],
    'PermitWitnessTransferFrom': [{  # Note: Changed from 'PermitTransferFrom' to match the contract
        'name': 'permitted',
        'type': 'TokenPermissions'
    }, {
        'name': 'spender',
        'type': 'address'
    }, {
        'name': 'nonce',
        'type': 'uint256'
    }, {
        'name': 'deadline',
        'type': 'uint256'
    },{
        'name': 'witness',
        'type': 'ExampleTrade'
    }]
}

latest_block = w3.eth.get_block('latest')
deadline = latest_block['timestamp'] + 1000000 

NONCE = 7
wordPos = NONCE >> 8
bitPos = NONCE & 0xFF

bitmap = PERMIT2_CONTRACT.functions.nonceBitmap(TOKEN_OWNER_ADDRESS, wordPos).call()

# checking if bit is set
if bitmap & (1 << bitPos):
    print(f"Nonce {NONCE} is already used.")


msg_data = {
    'permitted': {
        'token': ERC20_CONTRACT_ADDRESS,
        'amount': VALUE_TO_SPEND
    },
    'spender': PERMIT2_WITNESS_CONTRACT_ADDRESS,
    'nonce': NONCE,
    'deadline': deadline,
    'witness': {
        'exampleTokenAddress': ERC20_CONTRACT_ADDRESS,
        'exampleMinimumAmountOut': VALUE_TO_SPEND
    }
}

witness = w3.keccak(eth_abi.encode(
    ['bytes32', 'address', 'uint256'], [WITNESS_TYPEHASH, ERC20_CONTRACT_ADDRESS, VALUE_TO_SPEND]
))

signed_msg = web3.Account.sign_typed_data(PRIVATE_KEY_TOKEN_OWNER, DOMAIN_DATA, _PERMIT2_TRANSFER_FROM_WITNESS_MESSAGE_TYPES, msg_data)
signature = signed_msg.signature.to_0x_hex()

tx_nonce = w3.eth.get_transaction_count(TOKEN_OWNER)
tx = PERMIT2_WITNESS_VAULT_CONTRACT.functions.deposit(
    ERC20_CONTRACT_ADDRESS,
    VALUE_TO_SPEND,
    NONCE,
    deadline,
    witness,
    TOKEN_OWNER,
    signature
).build_transaction({"from": TOKEN_OWNER_ADDRESS, "nonce": tx_nonce})


signed_tx = w3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY_TOKEN_OWNER)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
print("Waiting for transaction receipt...")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
print("Transaction included. Waiting for a few more confirmations")
tx_block_number = tx_receipt['blockNumber']
latest_block = w3.eth.get_block('latest')

while latest_block['number'] <= tx_block_number + 2:
    latest_block = w3.eth.get_block('latest')

print()
print("ERC20 Token Information:")
print(f"    Name: {ERC20_CONTRACT.functions.name().call()}")
print(f"    Symbol: {ERC20_CONTRACT.functions.symbol().call()}")
print(f"    Decimals: {ERC20_CONTRACT.functions.decimals().call()}")
print(f"    Total Supply: {ERC20_CONTRACT.functions.totalSupply().call()}")
print(f"    Balance of {TOKEN_OWNER_ADDRESS}: {ERC20_CONTRACT.functions.balanceOf(TOKEN_OWNER_ADDRESS).call()}")
print(f"    Balance of Permit2Vault: {ERC20_CONTRACT.functions.balanceOf(PERMIT2_WITNESS_CONTRACT_ADDRESS).call()}")
print(f"    Allowance of Permit2 for {TOKEN_OWNER}: {ERC20_CONTRACT.functions.allowance(TOKEN_OWNER_ADDRESS, PERMIT2_CONTRACT_ADDRESS).call()}")
```

This code is similar to the previous one, but we are using the `permitWitnessTransferFrom()`
function of the `Permit2` contract to transfer tokens on behalf of the user.
We are also signing the `ExampleTrade` message and passing it to the
`deposit()` function of the `Permit2Vault` contract, which ultimately
calls the `permitWitnessTransferFrom()` function of the `Permit2` contract and
verifies our `ExampleTrade` message for us.

As you may have noticed, we haven't approved the `Permit2` contract in the
ERC20 contract. This is because I reused the ERC20 contract from the previous
example and the `Permit2` contract is already approved.

### Nonces in Permit2 <a name="nonces-in-permit2"></a>

Permit2 has non-monotonic nonces.

Permit2's non-monotonic nonces are an intentional design feature that provides important flexibility beyond the traditional monotonic (always-increasing) nonce pattern.
In traditional nonce systems (like Ethereum transactions), nonces must be used sequentially. This creates several limitations:

* Parallelism issues: You can't execute multiple independent operations simultaneously because they must be processed in strict sequence.
* Cancellation problems: If you want to cancel a pending operation, you typically need to replace it with another one using the same nonce and higher gas price.
* Resource inefficiency: Requiring strict sequence can waste resources when operations aren't time-dependent on each other.

Permit2's non-monotonic nonce system removes these constraints by allowing:

* Multiple valid signatures simultaneously: Different operations can use different nonces from the available space (2^248 possible values).
* Domain-specific nonces: Operations in different domains or for different purposes can use separate nonce spaces.
* Independent validation: Each signature can be validated independently without checking the entire history of nonces.
* Selective cancellation: You can invalidate specific nonces without affecting others.

This design is particularly valuable in a permissioning system like Permit2, where various independent entities might want to obtain permissions without requiring coordination or sequential processing.
The security is maintained because once a nonce is used, it's permanently marked as used, preventing replay attacks just like monotonic nonces do, but with greater flexibility in how they're assigned and consumed.

The nonceBitmap maps an owner's address to a uint248 value, which we will call wordPos which is the index of the desired bitmap.
There are 2248 possible indices thus 2248 possible bitmaps where each bitmap holds 256 bits. A bit must be flipped on to prevent
replays of user's signatures. Bits that are dirtied may not be used again.

`mapping(address => mapping(uint248 => uint256)) public nonceBitmap;`
to access: nonceBitmap[address][wordPos]
where wordPos = nonce >> 8

to flip a bit: nonceBitmap[address][wordPos] ^= (1 << bitPos)
where bitPos = uint8(nonce)

## Real-world Implementation Examples <a name="real-world-implementation-examples"></a>

Different projects have adopted different approval mechanisms based on their specific needs:

### ERC20 `approve`
- **Uniswap V2**: Uses standard approvals for token swaps
- **Aave V2**: Requires approval for depositing assets

### EIP2612 `permit`
- **DAI Stablecoin**: One of the first tokens to implement permit functionality
- **Uniswap V3**: Supports permit for improved UX with compatible tokens
- **Compound V3**: Incorporates permit to streamline lending deposits

### Permit2
- **Uniswap Universal Router**: Uses Permit2 for gas-efficient swaps
- **Uniswap X**: Leverages Permit2 for aggregated and batched transactions
- **1inch**: Has integrated Permit2 compatibility for cross-protocol interactions

This progression shows how the ecosystem is gradually moving toward more efficient, secure, and flexible approval mechanisms.

## Recommendation for New ERC20 Token Developers <a name="recommendation-for-new-erc20-token-developers"></a>

Implement EIP2612 Permit in Your ERC20 Tokens

If you're developing a new ERC20 token contract, we strongly recommend implementing the EIP2612 permit functionality directly within your token. Here's why:

* Enhanced User Experience: Permit enables gasless approvals, allowing users to authorize token transfers with a signature rather than a separate transaction. This significantly reduces friction in your token's UX flow.
* Reduced Transaction Costs: By eliminating separate approval transactions, you save your users gas fees and simplify their interaction with your token.
* Future-Proof Compatibility: Many modern DeFi protocols and interfaces already support permit functionality. Building it in from the start ensures your token will work seamlessly with these systems.
* Implementation Simplicity: Using OpenZeppelin's ERC20Permit extension makes implementation straightforward. You can simply inherit from this contract rather than building the functionality from scratch.

```
solidity// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

contract MyToken is ERC20Permit {
    constructor(uint256 initialSupply) 
        ERC20("MyToken", "MTK") 
        ERC20Permit("MyToken") {
        _mint(msg.sender, initialSupply);
    }
}
```
* Security Considerations: The permit function provides time-bound approvals through the deadline parameter, reducing the security risks associated with unlimited approvals.

Even if your users don't immediately take advantage of permit functionality, implementing it future-proofs your token. As the ecosystem evolves toward more efficient user experiences, your token will be ready to participate in these improvements without requiring upgrades or wrapper contracts.
For tokenomics systems where you anticipate high volumes of approvals and transfers, this small upfront implementation cost pays significant dividends in long-term usability and integration potential.

## Conclusion <a name="conclusion"></a>

The evolution of token approval mechanisms in Ethereum represents a critical advancement in blockchain UX design. We've traced this journey from the basic but cumbersome ERC20 approve function, through the more streamlined EIP2612 permit approach, to Uniswap's versatile Permit2 contract.
Each iteration has addressed key limitations of its predecessors:

ERC20 approvals established the foundation but required separate transactions and created security concerns with unlimited allowances
EIP2612's permit eliminated separate approval transactions through off-chain signatures but was limited to tokens that implemented the standard
Permit2 brought these benefits to all tokens while adding batch operations, expiring approvals, and custom witness data through permitWitnessTransferFrom

These improvements collectively work toward a more user-friendly DeFi ecosystem where token interactions are faster, cheaper, and more secure. By reducing transaction count, minimizing gas costs, and adding time-bound permissions, these mechanisms are making blockchain applications more accessible to mainstream users.
For developers building on Ethereum, choosing the right approval mechanism depends on your specific use case, user base, and security requirements. New projects should strongly consider implementing EIP2612 permit functionality directly or integrating with Permit2 for maximum flexibility and compatibility.
As the ecosystem continues to evolve, we can expect further innovations that will make token interactions even more seamless. The ongoing focus on improving these fundamental building blocks is what will ultimately enable Ethereum to scale beyond early adopters and into widespread adoption.
By understanding these approval mechanisms and implementing them correctly, you'll be contributing to a more efficient, secure, and user-friendly DeFi landscape.
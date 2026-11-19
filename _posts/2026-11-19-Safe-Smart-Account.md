---
layout: post
title: "Safe Smart Account: What It Is and How the Contracts Fit Together"
description: "An introduction to Safe (formerly Gnosis Safe): what it is, what it is used for, and how its contracts are structured, from the proxy and singleton to owners, modules, guards, and the fallback handler."
tags: Ethereum Solidity Safe Multisig Smart-Contracts Account-Abstraction
---

# Safe Smart Account: What It Is and How the Contracts Fit Together

If you have spent some time in Ethereum, you have most likely come across Safe
(formerly Gnosis Safe). DAOs keep their treasuries in it, teams use it to hold
admin keys, and when I [set up my own OP Stack chain]({% post_url 2026-07-10-Setting-Up-An-Optimism-Chain %})
the recommendation for roles like `proxy_admin_owner` was to use a Safe instead
of a single key.

I have used Safe plenty of times, but never looked at how the contracts actually
work. So that's what this post is about. We'll go through:

* What Safe is
* What it is used for
* How the contracts are structured

This is the first post on Safe. There is more to cover (signatures, modules in
detail, ERC-4337), but let's start with the basics.

All code in this post refers to the
[safe-smart-account](https://github.com/safe-global/safe-smart-account)
repository. The latest release at the time of writing is **v1.5.0**, and **v1.4.1**
is still the most widely deployed version. Code snippets are shortened to the
parts that matter.

## Table of contents

1. [What Is Safe?](#1-what-is-safe)
2. [What Is It Used For?](#2-what-is-it-used-for)
3. [Architecture Overview](#3-architecture-overview)
4. [Proxy and Singleton](#4-proxy-and-singleton)
5. [The SafeProxyFactory](#5-the-safeproxyfactory)
6. [The Safe Contract](#6-the-safe-contract)
7. [Executing a Transaction](#7-executing-a-transaction)
8. [Owners](#8-owners)
9. [Modules](#9-modules)
10. [Guards](#10-guards)
11. [The Fallback Handler](#11-the-fallback-handler)
12. [Library Contracts](#12-library-contracts)
13. [Safe vs. SafeL2](#13-safe-vs-safel2)
14. [Conclusion](#conclusion)

## 1. What Is Safe?

Safe is a **smart contract account**. Instead of an Externally Owned Account
(EOA) controlled by a single private key, your funds sit in a contract, and the
contract decides who is allowed to move them.

By default, that rule is a **multisig**: a Safe has a list of owners and a
threshold. A transaction is only executed if at least `threshold` owners signed
it. For example, a 2-of-3 Safe has three owners, and any two of them together
can execute a transaction.

```text
           EOA                                   Safe (2-of-3)
┌──────────────────────────┐       ┌──────────────────────────────────────┐
│ 1 private key            │       │ Owners: Alice, Bob, Carol            │
│ Key lost  -> funds lost  │  vs.  │ Threshold: 2                         │
│ Key leaks -> funds gone  │       │ One key lost/leaked -> still safe    │
│ No custom rules          │       │ Extensible via modules and guards    │
└──────────────────────────┘       └──────────────────────────────────────┘
```

Since it's a contract, you are not limited to a multisig though. You can add
**modules** (e.g. spending limits, recovery, session keys) and **guards** (checks
that run before and after every transaction). We'll look at both further down.

A few things that make Safe stand out:

* **It has been around for a long time.** The first version was released in
  2018 and it holds a large amount of value on Ethereum and other EVM chains.
* **It is heavily audited** and has been formally verified (Certora did the
  formal verification for v1.5.0).
* **It is the same contract on many chains.** Thanks to deterministic
  deployments, the Safe singleton and factory have the same address on most EVM
  chains.

## 2. What Is It Used For?

Some common use cases:

* **DAO and company treasuries:** Funds can only move if several people agree.
  No single person can run off with the money.
* **Protocol admin keys:** Upgrading a proxy, changing parameters, pausing a
  contract. These are powerful operations and should not depend on a single key.
  Most protocols put a Safe (often behind a timelock) as the owner.
* **Personal security:** Even as an individual, you can use a 2-of-3 Safe with
  a hardware wallet, a mobile wallet, and a backup key. Losing one of them is not
  the end of the world.
* **Smart account for users:** With the Safe 4337 module, a Safe can be used as
  an ERC-4337 account. If you want to know more about that, check out my post on
  [account abstraction]({% post_url 2024-10-17-Account-Abstraction %}).
* **Batching transactions:** Approve and swap in one go, or pay 50 contributors
  in a single transaction.

## 3. Architecture Overview

Let's have a look at the big picture first. The main pieces are:

| Contract | Role |
|---|---|
| `Safe` / `SafeL2` | The singleton holding all the logic. Deployed once per chain. |
| `SafeProxy` | The actual account of a user. Holds the funds and storage, delegates every call to the singleton. |
| `SafeProxyFactory` | Deploys new `SafeProxy` contracts, optionally to a deterministic address. |
| `CompatibilityFallbackHandler` | Handles calls the Safe itself doesn't know (e.g. EIP-1271 `isValidSignature`, token callbacks). |
| Modules | Optional contracts that can execute transactions on behalf of the Safe. |
| Guards | Optional contracts that check transactions before and after execution. |
| Libraries (`MultiSend`, `CreateCall`, `SignMessageLib`, ...) | Helper contracts the Safe calls via `delegatecall`. |

And this is how they fit together:

```text
                           ┌───────────────────────────┐
     createProxyWithNonce  │     SafeProxyFactory      │
   ───────────────────────►│                           │
                           └─────────────┬─────────────┘
                                         │ deploys + calls setup()
                                         ▼
 User / Owners ──────────► ┌───────────────────────────┐   delegatecall   ┌───────────────────────────┐
  execTransaction          │        SafeProxy          │ ───────────────► │   Safe (singleton)        │
                           │  (funds + storage)        │                  │                           │
                           └─────────────┬─────────────┘                  │  OwnerManager             │
                                         │                                │  ModuleManager            │
              ┌──────────────────────────┼─────────────────────┐          │  GuardManager             │
              ▼                          ▼                     ▼          │  FallbackManager          │
     ┌─────────────────┐      ┌────────────────────┐   ┌───────────────┐  │  ...                      │
     │    Modules      │      │      Guard         │   │ Fallback      │  └───────────────────────────┘
     │ (execTransaction│      │ (checkTransaction, │   │ Handler       │
     │  FromModule)    │      │  checkAfterExec.)  │   │               │
     └─────────────────┘      └────────────────────┘   └───────────────┘
```

Let's go through them one by one.

## 4. Proxy and Singleton

Every Safe you create is a small `SafeProxy` contract. It holds the funds and
the storage (owners, threshold, nonce, ...), but it has no logic of its own.
Every call is forwarded via `delegatecall` to the `Safe` singleton, which
contains the actual code.

Why do it this way? Deploying the full Safe contract for every user would be
expensive. The proxy is tiny, so creating a new Safe is cheap. The logic is
deployed only once per chain and shared by everyone.

The proxy stores the singleton address in storage slot `0`:

```solidity
contract SafeProxy {
    // Singleton always needs to be first declared variable,
    // to ensure that it is at the same location in the contracts
    // to which calls are delegated.
    address internal singleton;

    constructor(address _singleton) {
        require(_singleton != address(0), "Invalid singleton address provided");
        singleton = _singleton;
    }

    fallback() external payable {
        assembly {
            let _singleton := and(sload(0), 0xffffffffffffffffffffffffffffffffffffffff)
            // 0xa619486e == keccak("masterCopy()")
            if eq(calldataload(0), 0xa619486e00000000000000000000000000000000000000000000000000000000) {
                mstore(0, _singleton)
                return(0, 0x20)
            }
            calldatacopy(0, 0, calldatasize())
            let success := delegatecall(gas(), _singleton, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            if eq(success, 0) {
                revert(0, returndatasize())
            }
            return(0, returndatasize())
        }
    }
}
```

There are two things worth pointing out:

* The proxy has only a `fallback()`. Any function you call on it ends up there
  and is forwarded to the singleton.
* There is one special case: `masterCopy()`. The proxy answers this call itself
  and returns the singleton address. That's how tools can find out which version
  a Safe is running.

Since the singleton runs in the context of the proxy, the `Safe` contract also
has to keep its first storage variable at slot `0`. That's what the `Singleton`
base contract is for:

```solidity
abstract contract Singleton {
    // singleton always has to be the first declared variable to ensure the same location as in the Proxy contract.
    address private singleton;
}
```

Note that this is **not** an upgradeable proxy in the usual sense. There is no
admin who can switch the implementation for you. The only way to change the
singleton is a transaction signed by the owners of that Safe (e.g. through a
migration contract via `delegatecall`).

## 5. The SafeProxyFactory

New Safes are created through the `SafeProxyFactory`. The most commonly used
function is `createProxyWithNonce`:

```solidity
function createProxyWithNonce(
    address _singleton,
    bytes memory initializer,
    uint256 saltNonce
) public returns (SafeProxy proxy) {
    // If the initializer changes the proxy address should change too.
    // Hashing the initializer data is cheaper than just concatinating it
    bytes32 salt = keccak256(abi.encodePacked(keccak256(initializer), saltNonce));
    proxy = deployProxy(_singleton, initializer, salt);
    emit ProxyCreation(proxy, _singleton);
}
```

It deploys the proxy with `CREATE2` and then calls the proxy with
`initializer`, which is usually an encoded call to `setup(...)`.

The salt includes the hash of the initializer. That's important: the address of
a Safe depends on its initial owners and threshold. Nobody can front-run your
deployment and create a Safe at "your" address with different owners.

It also means you can know the address of a Safe **before** deploying it. You
can send funds to it, and only deploy it when you actually need it. If you use
the same initializer and salt on another chain, you get the same address there
too (`SafeToL2Setup` was added to make this work across L1s and L2s).

The `setup` function looks like this:

```solidity
function setup(
    address[] calldata _owners,
    uint256 _threshold,
    address to,
    bytes calldata data,
    address fallbackHandler,
    address paymentToken,
    uint256 payment,
    address payable paymentReceiver
) external
```

* `_owners` and `_threshold`: the multisig configuration.
* `to` and `data`: an optional `delegatecall` executed during setup, e.g. to
  enable modules right away.
* `fallbackHandler`: usually the `CompatibilityFallbackHandler`.
* `paymentToken`, `payment`, `paymentReceiver`: optionally refund whoever paid
  for the deployment.

`setup` can only be called once. If the threshold is already set, it reverts.

## 6. The Safe Contract

The `Safe` singleton is not one big contract. It is composed of several base
contracts, each responsible for one part of the account:

| Base contract | Responsibility |
|---|---|
| `Singleton` | Keeps slot `0` reserved for the singleton address. |
| `OwnerManager` | Owners and threshold. |
| `ModuleManager` | Enabling, disabling, and executing modules. In v1.5.0 also the module guard. |
| `GuardManager` | Setting the transaction guard. |
| `FallbackManager` | Forwarding unknown calls to the fallback handler. |
| `NativeCurrencyPaymentFallback` | Accepting plain ETH transfers (`receive()`). |
| `SignatureDecoder` | Splitting the packed signatures into `v`, `r`, `s`. |
| `SecuredTokenTransfer` | Safe ERC-20 transfers for gas refunds. |
| `StorageAccessible` | Reading storage and simulating calls, mostly for off-chain tooling. |

On top of these, `Safe.sol` itself adds the parts that tie it all together:
`setup`, `execTransaction`, the signature checks, and the nonce.

## 7. Executing a Transaction

The main entry point is `execTransaction`:

```solidity
function execTransaction(
    address to,
    uint256 value,
    bytes calldata data,
    Enum.Operation operation,
    uint256 safeTxGas,
    uint256 baseGas,
    uint256 gasPrice,
    address gasToken,
    address payable refundReceiver,
    bytes memory signatures
) public payable virtual returns (bool success)
```

* `to`, `value`, `data`: what to call, like a normal transaction.
* `operation`: either `Call` or `DelegateCall`. With `DelegateCall` the target
  code runs in the context of the Safe. That's how library contracts like
  `MultiSend` work, but it's also dangerous: a malicious target can change the
  Safe's storage.
* `safeTxGas`, `baseGas`, `gasPrice`, `gasToken`, `refundReceiver`: used for gas
  refunds. The Safe can pay back whoever submits the transaction, in ETH or an
  ERC-20 token. If `gasPrice` is `0`, there is no refund.
* `signatures`: the owners' signatures, packed together.

What happens inside, simplified:

```text
execTransaction(...)
   │
   ├─ 1. Build the EIP-712 hash of the SafeTx (including the current nonce)
   ├─ 2. nonce++
   ├─ 3. checkSignatures(txHash, signatures)  ──► reverts if < threshold valid owner signatures
   ├─ 4. guard.checkTransaction(...)          ──► only if a guard is set
   ├─ 5. execute(to, value, data, operation, gas)
   ├─ 6. handlePayment(...)                   ──► only if gasPrice > 0
   ├─ 7. guard.checkAfterExecution(txHash, success)
   └─ 8. emit ExecutionSuccess / ExecutionFailure
```

Owners don't sign a raw transaction. They sign an [EIP-712](https://eips.ethereum.org/EIPS/eip-712)
hash of the `SafeTx` struct. This includes the Safe's address and chain ID (in
the domain separator) and the **nonce**, so a signature can't be replayed on
another Safe, another chain, or a second time on the same Safe.

The nonce is increased with every executed transaction, even if the inner call
fails. That's also why Safe transactions have to be executed in order: you
can't execute nonce 5 before nonce 4.

### Signatures

The `signatures` parameter is a concatenation of 65-byte signatures. They must be
sorted by owner address in ascending order. That's a cheap way to make sure that
the same owner isn't counted twice.

The `v` value of each signature tells the Safe what type of signature it is:

| `v` | Type | How it's checked |
|---|---|---|
| `0` | Contract signature | The owner is a contract; the Safe calls its EIP-1271 `isValidSignature`. |
| `1` | Approved hash | The owner approved the hash on-chain via `approveHash`, or the owner is the one submitting the transaction. |
| `> 30` | `eth_sign` | The hash was signed with the `\x19Ethereum Signed Message` prefix; `v` is reduced by 4. |
| `27` / `28` | ECDSA | A normal signature over the EIP-712 hash. |

The contract signature type is what allows a Safe to be the owner of another
Safe. We'll go into signatures in more detail in a later post.

## 8. Owners

Owners are managed in [`OwnerManager.sol`](https://github.com/safe-global/safe-smart-account/blob/main/contracts/base/OwnerManager.sol)
and stored in a mapping that works as a **linked list**:

```solidity
address internal constant SENTINEL_OWNERS = address(0x1);

mapping(address => address) internal owners;
uint256 internal ownerCount;
uint256 internal threshold;
```

Each owner points to the next one, and the last one points back to
`SENTINEL_OWNERS`:

```text
SENTINEL(0x1) ──► Alice ──► Bob ──► Carol ──► SENTINEL(0x1)
```

### Why a linked list?

When I first saw this I wondered: why not just use a
`mapping(address => bool)`?

A plain mapping can tell you **whether** an address is an owner, but not **who**
the owners are. In Solidity, mappings don't store their keys, so there is no way
to loop over them. The Safe needs that list though: `getOwners()` returns all
owners, and that's what wallets and other tools use to show who controls a Safe.

So the Safe needs two things at once: a cheap membership check and a way to list
all owners. The usual way to get both is a mapping plus an array:

```solidity
mapping(address => uint256) internal indexOf;
address[] internal ownersArray;
```

That works, but:

* Every owner takes up 2 storage slots (mapping entry and array element).
* Removing an owner needs swap-and-pop: move the last element into the gap,
  update its index, shrink the array, and clear the removed entry. That's
  around 4 storage writes and easy to get wrong.

The linked list gets both with a single mapping. Instead of `true`, the value is
the next owner:

* **Membership:** `owners[addr] != address(0)` means `addr` is an owner (as long
  as it isn't the sentinel). One storage read.
* **Listing:** start at `owners[SENTINEL_OWNERS]` and follow the pointers until
  you are back at the sentinel. That's exactly what `getOwners()` does.
* **Adding:** 2 storage writes. The new owner points to the current first owner,
  and the sentinel points to the new owner.
* **Removing:** 2 storage writes. The previous owner skips over the removed one,
  and the removed owner's slot is cleared.

The contract itself also mentions this: the code the compiler generates for the
linked list is more efficient than for a dynamic array.

There is one catch. To remove an owner you need to know the **previous** owner
in the list. Finding it on-chain would mean walking the list, so the caller
passes it in as `prevOwner`. Wallets compute it off-chain from `getOwners()`, and
the contract only checks that `owners[prevOwner] == owner` (otherwise it reverts
with `GS205`). That's why `removeOwner` and `swapOwner` have that extra
parameter, which can be confusing the first time you see it.

Signature checks don't need the list at all, by the way. For each signer, the
Safe only checks `owners[signer] != address(0)`, and the signatures have to be
sorted by signer address so nobody is counted twice.

### Managing owners

The owner functions look like this:

```solidity
function addOwnerWithThreshold(address owner, uint256 _threshold) public authorized
function removeOwner(address prevOwner, address owner, uint256 _threshold) public authorized
function swapOwner(address prevOwner, address oldOwner, address newOwner) public authorized
function changeThreshold(uint256 _threshold) public authorized
```

These functions also enforce a few rules:

* An owner can't be `address(0)`, the sentinel, or the Safe itself, and can't be
  added twice.
* The threshold has to be at least 1 and can't be higher than the number of
  owners.

All of these are `authorized`, which means `msg.sender` has to be the Safe
itself:

```solidity
function requireSelfCall() private view {
    if (msg.sender != address(this)) revertWithError("GS031");
}
```

So to add an owner, you create a Safe transaction that calls the Safe itself, and
it needs to be signed by the current owners like any other transaction.

You probably also noticed the error code `GS031`. Instead of long revert
strings, Safe uses short error codes to save gas. You can look them up in the
[error codes list](https://github.com/safe-global/safe-smart-account/blob/main/docs/error_codes.md).

## 9. Modules

Modules are contracts that are allowed to execute transactions **without** the
owners' signatures. A module calls one of these functions on the Safe:

```solidity
function execTransactionFromModule(
    address to,
    uint256 value,
    bytes memory data,
    Enum.Operation operation
) external returns (bool success)

function execTransactionFromModuleReturnData(...) external returns (bool success, bytes memory returnData)
```

The Safe only checks that `msg.sender` is an enabled module. No signatures, no
threshold, no nonce. The module itself has to implement whatever rules make
sense.

Modules are stored in the same linked-list style as the owners
(`SENTINEL_MODULES = address(0x1)`), and are enabled or disabled through
`enableModule` and `disableModule`, both `authorized`.

Some examples of modules:

* **Allowance / spending limit:** a team member can spend up to 1,000 USDC per
  week without asking the other owners.
* **Recovery:** if you lose access to your keys, guardians can replace the
  owners after a delay.
* **Safe 4337 module:** makes the Safe compatible with ERC-4337, so it can be
  used with bundlers and paymasters.
* **Zodiac modules:** e.g. Reality.eth or Roles, often used by DAOs.

This is very powerful, but also the biggest risk. **An enabled module has full
control over the Safe.** A buggy or malicious module can drain it. Only enable
modules you trust as much as your owners.

## 10. Guards

Guards are the opposite of modules. A module adds a way to execute
transactions; a guard adds restrictions.

A transaction guard implements two hooks:

```solidity
interface ITransactionGuard is IERC165 {
    function checkTransaction(
        address to,
        uint256 value,
        bytes memory data,
        Enum.Operation operation,
        uint256 safeTxGas,
        uint256 baseGas,
        uint256 gasPrice,
        address gasToken,
        address payable refundReceiver,
        bytes memory signatures,
        address msgSender
    ) external;

    function checkAfterExecution(bytes32 hash, bool success) external;
}
```

`checkTransaction` runs before the transaction, `checkAfterExecution` after it.
If either reverts, the whole transaction reverts. With this you can, for
example:

* block `DelegateCall` operations
* only allow calls to a list of approved contracts
* make sure certain storage (e.g. the owner list) didn't change

The guard is set with `setGuard` and stored in a fixed storage slot:

```solidity
// keccak256("guard_manager.guard.address")
bytes32 internal constant GUARD_STORAGE_SLOT = 0x4a204f620c8c5ccdca3fd54d003badd85ba500436a431f0cbda4f558c93c34c8;
```

Fixed slots like this are used so the values don't collide with the normal
storage layout. The same pattern is used for the fallback handler.

Before v1.5.0, guards only ran for `execTransaction`, so modules could skip them.
v1.5.0 added **module guards** (`IModuleGuard`) with `checkModuleTransaction`
and `checkAfterModuleExecution`, which run for module transactions as well.

Be careful with guards, though. If a guard reverts on every transaction (e.g.
because of a bug), the Safe is locked. Even `setGuard` to remove it is a
transaction that goes through the guard.

## 11. The Fallback Handler

The Safe contract has a fixed set of functions. But there are calls a Safe
should be able to answer that are not part of `Safe.sol`, for example:

* `isValidSignature` (EIP-1271), so other contracts can check signatures from
  the Safe
* `onERC721Received` / `onERC1155Received`, so the Safe can receive NFTs via
  `safeTransferFrom`
* `supportsInterface` (ERC-165)

Instead of adding all of that to the singleton, the `FallbackManager` forwards
any unknown call to a separate **fallback handler** contract. Usually that's the
`CompatibilityFallbackHandler`.

```text
Caller ──► SafeProxy ──delegatecall──► Safe singleton
                                          │ function not found -> fallback()
                                          ▼
                                   call(fallbackHandler, calldata ++ msg.sender)
```

Note that this is a `call`, not a `delegatecall`. The handler can't modify the
Safe's storage. And since the handler sees the Safe as `msg.sender`, the Safe
appends the original caller's address to the end of the calldata. The handler
can read it from the last 20 bytes if needed.

v1.5.0 also added the `ExtensibleFallbackHandler`, which lets you register
handlers per function selector. That's used for things like TWAP orders with
CoW Swap.

## 12. Library Contracts

Finally, there are a few helper contracts that are meant to be used with
`operation = DelegateCall`:

| Contract | What it does |
|---|---|
| `MultiSend` | Executes several transactions in one Safe transaction. |
| `MultiSendCallOnly` | Same, but only allows `Call`, not `DelegateCall`. The safer choice for most cases. |
| `CreateCall` | Deploys a contract with `CREATE` or `CREATE2` from the Safe. |
| `SignMessageLib` | Marks a message as signed by the Safe, for EIP-1271 checks. |

The Safe Wallet UI uses `MultiSendCallOnly` when you batch transactions, e.g.
approve and swap in one go.

## 13. Safe vs. SafeL2

There are two singletons: `Safe` and `SafeL2`. The logic is the same. The only
difference is that `SafeL2` emits extra events with all transaction parameters
(`SafeMultiSigTransaction`, `SafeModuleTransaction`).

On Ethereum mainnet, indexers like the Safe Transaction Service can find Safe
transactions by tracing. On many L2s tracing is expensive or not available, so
the extra events make indexing possible there. The events cost extra gas, but gas
on L2s is cheap.

## Conclusion

Here is a short summary of what we covered:

* **Safe** is a smart contract account, by default a multisig with owners and a
  threshold.
* Each Safe is a tiny **proxy** that delegates to a shared **singleton**, created
  through the **SafeProxyFactory** at a deterministic address.
* Transactions are executed with `execTransaction` after checking **EIP-712
  signatures** from at least `threshold` owners, protected by a **nonce**.
* **Owners** and **modules** are stored as linked lists.
* **Modules** can execute transactions without signatures, **guards** can block
  them, and the **fallback handler** answers calls the Safe doesn't know.

What I like about the design is how small the core is. Everything beyond
"owners sign, Safe executes" is pushed into modules, guards, and handlers. That
keeps the audited core stable while you can still extend the Safe in many ways.
The downside is that every extension can take control of your funds, so you
have to know exactly what you enable.

In the next posts I want to look at signatures in more detail, build a small
module, and see how Safe works with ERC-4337 and
[EIP-7702]({% post_url 2025-05-21-EIP-7702-set-code-for-EOAs %}).

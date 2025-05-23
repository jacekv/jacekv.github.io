---
layout: post
title: "The Game-Changing EIP-7702"
tags: Ethereum Blockchain Account-Abstraction EIP-7702
---

# The Game-Changing EIP-7702

At epoch 364032 of the Ethereum Mainnet, the Pectra network upgrade was
activated, introducing EIP-7702 - a revolutionary proposal that allows
Externally Owned Accounts (EOAs) to set executable code for their accounts for
the first time in Ethereum's history.

**This marks a historic milestone: the first concrete step toward full account abstraction on Ethereum.**

## Why This Matters
Before Pectra, Ethereum had two distinct account types:

* EOAs: User-controlled accounts with private keys but no code execution capabilities
* Contract Accounts: Accounts with code but no direct user control

EIP-7702 begins to blur this line, enabling EOAs to behave more like smart contracts while maintaining their user-controlled nature. This creates exciting possibilities for:

* Custom transaction validation logic
* Multi-signature wallets without deploying separate contracts
* Gas sponsorship mechanisms
* Account recovery options
* Batched transactions
* And much more...

## What We'll Explore
In this guide, we'll dive into:

* How EIP-7702 actually works under the hood
* The practical implications for users and developers
* A step-by-step example using web3.py

Our example will demonstrate a simple `Increment` contract that automatically
increments a counter whenever the EOA receives a transaction.
While this is just a basic demonstration (not a full smart wallet implementation),
it will clearly illustrate how to set and use code with an EOA.

Let's unlock the potential of this groundbreaking change to the Ethereum protocol!

## How EIP-7702 Works

EIP-7702 introduces a powerful mechanism that allows an **EOA** to delegate its 
behavior to an existing smart contract. Instead of executing the default EOA
logic (i.e., none), the EOA can **point to a deployed contract**,
and any transactions sent to the EOA will instead execute the code of that contract.

Crucially, this is done **without converting the EOA into a full contract account**, the account remains under the user's control and retains its fundamental identity as an EOA.

### How It Works — Step by Step:

1. **Transaction Setup**: A user submits a transaction with special fields (defined by EIP-7702) that set a pointer from their EOA to a smart contract address.
2. **Delegated Execution**: While this pointer is active, any transaction sent to the EOA's address will execute the code at the target contract instead.
3. **Reversible Behavior**: The pointer can be updated or removed by the user, allowing the EOA to switch between different behaviors or revert to its default state.

This pointer-based model brings powerful new flexibility. Instead of deploying separate contracts for wallets or validation logic, users can **keep their EOA and simply link it to reusable contract logic**, updating or replacing it as needed.

> ⚙️ Think of it as "plug-and-play" code execution for EOAs — without giving up user custody or bloating the blockchain with redundant deployments.

### Benefits Over Traditional Account Abstraction Models

EIP-7702 is a lean and elegant alternative to earlier account abstraction proposals
like EIP-4337, which relies on a separate transaction pool and entry point contracts.
In contrast, EIP-7702 operates at the protocol level and:

- **Does not require bundlers or relayers**
- **Maintains backward compatibility with EOAs**
- **Avoids persistent on-chain code bloat**

This makes it highly attractive for wallet developers and protocol designers
looking for simplicity, composability, and cost efficiency.

---

## Setting the Stage: Our Example

To demonstrate how this works in practice, we'll walk through an example where a
user-controlled EOA sets temporary executable code that increments a counter whenever
the account receives ETH.

Here's the plan:

- We'll define a minimal `Increment` contract in Solidity and deploy it.
- We'll use `web3.py` to send a transaction from an EOA and once from a relayer to set the pointer.
- We'll observe the counter increase, showing that the code was executed in the context of the EOA.

Let's begin by writing our simple contract.

---

## Step 1: The `Increment` Contract

Here's the Solidity code for our minimal example:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Increment {
    uint256 public counter;

    constructor() {
        counter = 0;
    }

    function increment() external {
        counter += 1;
    }

    receive() external payable {
        counter += 1;
    }
}
```

Next, you will have to deploy it. You could use remix or any other tool you prefer.
I deployed such contract on Sepolia Testnet at [0x3B9D350E08d230Edf026Fd993Ba62B78b225883D](https://sepolia.etherscan.io/address/0x3B9D350E08d230Edf026Fd993Ba62B78b225883D)

## Step 2a: Set pointer as EOA and interact with contract

Now, let's set the pointer from our EOA to the deployed contract. Here is a Python
script using `web3.py` to do this:

```python
from eth_account import Account
from dotenv import load_dotenv
import os
from web3 import Web3, HTTPProvider

load_dotenv()

USER = Account.from_key(os.getenv("USER_PRIVATE_KEY"))
print(f"User address: {USER.address}")

# 1. instantiate w3
w3 = Web3(HTTPProvider("https://eth-sepolia.public.blastapi.io"))

# 2. load EOA account from private key
user_account_code = w3.eth.get_code(USER.address)
user_nonce = w3.eth.get_transaction_count(USER.address)
print(f"User account code before: {user_account_code.hex()}") # should be 0x or None
print(f"User account nonce before: {user_nonce}")

# I deployed this already beforehand
COUNTER_CONTRACT_ADDRESS = "0x3B9D350E08d230Edf026Fd993Ba62B78b225883D"
COUNTER_CONTRACT_ABI = [
	{
		"inputs": [],
		"name": "increment",
		"outputs": [],
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"inputs": [],
		"stateMutability": "nonpayable",
		"type": "constructor"
	},
	{
		"inputs": [],
		"name": "counter",
		"outputs": [
			{
				"internalType": "uint256",
				"name": "",
				"type": "uint256"
			}
		],
		"stateMutability": "view",
		"type": "function"
}]

USER_COUNTER_CONTRACT_ADDRESS = USER.address
COUNTER_CONTRACT = w3.eth.contract(address=USER_COUNTER_CONTRACT_ADDRESS, abi=COUNTER_CONTRACT_ABI)

#3. build an authorization utilizing the counter contract
# this is part of the new transaction type 4, setting the pointer
auth = {
    "chainId": w3.eth.chain_id,
    "address": COUNTER_CONTRACT_ADDRESS,
    "nonce": user_nonce + 1, #setting the chain ID to 0 allows the authorization to be replayed across all EVM-compatible chains supporting EIP-7702, provided the nonce matches.
}

# 4. sign the auth with the EOA
signed_auth = USER.sign_authorization(auth)

# 5. build the type 4 transaction
tx = {
    "chainId": w3.eth.chain_id,
    "nonce": w3.eth.get_transaction_count(USER.address),
    "gas": 1_000_000,
    "maxFeePerGas": 10**11,
    "maxPriorityFeePerGas": 10**9,
    "to": USER.address, # sending it to the EOA address
    "authorizationList": [signed_auth],
    # "data": "", can be set if a function call is needed
}

# 6. sign and send the transaction
signed_tx = USER.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

# 7. load EOA account from private key
user_account_code = w3.eth.get_code(USER.address)
user_nonce = w3.eth.get_transaction_count(USER.address)
# should show 0xef0100<address>, where 0xef0100 are magic bytes
print(f"User account code before: {user_account_code.hex()}")
print(f"User account nonce before: {user_nonce}")

# remember, COUNTER_CONTRACT is pointing to the EOA address
print(f"Counter: {COUNTER_CONTRACT.functions.counter().call()}") # this call would fail if authorization is not set!

# 8. call the counter contract
tx = COUNTER_CONTRACT.functions.increment().build_transaction({
    "from": USER.address,
    "nonce": w3.eth.get_transaction_count(USER.address)
})

signed_tx = USER.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

print(f"Counter: {COUNTER_CONTRACT.functions.counter().call()}") # this call would fail if authorization is not set!

## Remove authorization ##

auth = {
    "chainId": w3.eth.chain_id,
    "address": "0x0000000000000000000000000000000000000000", # to clear, set to 0x0
    "nonce": user_nonce + 1,
}

# 9. sign the auth with the EOA
signed_auth = USER.sign_authorization(auth)

# 10. build the type 4 transaction
tx = {
    "chainId": w3.eth.chain_id,
    "nonce": w3.eth.get_transaction_count(USER.address),
    "gas": 1_000_000,
    "maxFeePerGas": 12**10,
    "maxPriorityFeePerGas": 10**10,
    "to": USER.address,
    "authorizationList": [signed_auth],
}

signed_tx = USER.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

# 11. load EOA account from private key
user_account_code = w3.eth.get_code(USER.address)
print(f"User account code after: {user_account_code.hex()}")
```

## Step 2b: Set pointer using a relayer and interact with contract

Now, let's set the pointer for our EOA using a relayer pointing to the
deployed contract. Here is a Python script using `web3.py` to do this:

```python
from eth_account import Account
from dotenv import load_dotenv
import os
from web3 import Web3, HTTPProvider
# from artifacts import MULTICALL_ABI, MULTICALL_ADDRESS, COUNTER_ADDRESS, COUNTER_ABI
# import json

load_dotenv()

RELAYER = Account.from_key(os.getenv("RELAYER_PRIVATE_KEY"))
USER = Account.from_key(os.getenv("USER_PRIVATE_KEY"))

print(f"Relayer address: {RELAYER.address}")
print(f"User address: {USER.address}")

# 1. instantiate w3
w3 = Web3(HTTPProvider("https://eth-sepolia.public.blastapi.io"))
# 2. load EOA account from private key
user_account_code = w3.eth.get_code(USER.address)
user_nonce = w3.eth.get_transaction_count(USER.address)
print(f"User account code before: {user_account_code.hex()}")
print(f"User account nonce before: {user_nonce}")

COUNTER_CONTRACT_ADDRESS = "0x3B9D350E08d230Edf026Fd993Ba62B78b225883D"
COUNTER_CONTRACT_ABI = [
	{
		"inputs": [],
		"name": "increment",
		"outputs": [],
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"inputs": [],
		"stateMutability": "nonpayable",
		"type": "constructor"
	},
	{
		"inputs": [],
		"name": "counter",
		"outputs": [
			{
				"internalType": "uint256",
				"name": "",
				"type": "uint256"
			}
		],
		"stateMutability": "view",
		"type": "function"
}]

USER_COUNTER_CONTRACT_ADDRESS = USER.address
COUNTER_CONTRACT = w3.eth.contract(address=USER_COUNTER_CONTRACT_ADDRESS, abi=COUNTER_CONTRACT_ABI)

auth = {
    "chainId": w3.eth.chain_id,
    "address": COUNTER_CONTRACT_ADDRESS,
    "nonce": user_nonce,
}

# 3. sign the auth with the EOA
signed_auth = USER.sign_authorization(auth)

tx = {
    "chainId": w3.eth.chain_id,
    "nonce": w3.eth.get_transaction_count(RELAYER.address),
    "gas": 1_000_000,
    "maxFeePerGas": 12**10,
    "maxPriorityFeePerGas": 10**9,
    "to": USER.address,
    "authorizationList": [signed_auth],
}

# 4. sign the transaction with the relayer and submit it
signed_tx = RELAYER.sign_transaction(tx) # compared to the previous example, we are using the relayer to sign the transaction
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

# 5. load EOA account from private key
user_account_code = w3.eth.get_code(USER.address)
print(f"User account code after: {user_account_code.hex()}")

print(f"Counter: {COUNTER_CONTRACT.functions.counter().call()}")

# 6. call the counter contract
tx = COUNTER_CONTRACT.functions.increment().build_transaction({
    "from": RELAYER.address,
    "nonce": w3.eth.get_transaction_count(RELAYER.address)
})

signed_tx = RELAYER.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

print(f"Counter: {COUNTER_CONTRACT.functions.counter().call()}") # this call would fail if authorization is not set!

## Remove authorization ##
auth = {
    "chainId": w3.eth.chain_id,
    "address": "0x0000000000000000000000000000000000000000",
    "nonce": user_nonce + 1,
}

# 7. sign the auth with the EOA
signed_auth = USER.sign_authorization(auth)

# 8. build the type 4 transaction
tx = {
    "chainId": w3.eth.chain_id,
    "nonce": w3.eth.get_transaction_count(USER.address),
    "gas": 1_000_000,
    "maxFeePerGas": 12**10,
    "maxPriorityFeePerGas": 10**10,
    "to": USER.address,
    "authorizationList": [signed_auth],
}

signed_tx = USER.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
print(f"Transaction hash: {tx_hash.hex()}")
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

# 9. load EOA account from private key
user_account_code = w3.eth.get_code(USER.address)
print(f"User account code after: {user_account_code.hex()}")
```

## Conclusion

EIP-7702 represents a paradigm shift in how we think about Ethereum accounts.
By allowing EOAs to delegate their execution to smart contracts while maintaining user control.
This upgrade opens the door to a new era of account flexibility and innovation.

Through our practical examples, we've seen how elegantly EIP-7702 works in practice.
Whether setting authorization directly from the EOA or using a relayer to sponsor
the transaction, the process is straightforward yet powerful.

The ability to point an EOA to different contract implementations—and switch between them or remove them entirely—creates unprecedented flexibility for wallet developers and users alike.

### Key Takeaways

For Users:

* Your EOA can now behave like a smart contract without losing control
* Gas sponsorship becomes possible through relayer transactions
* You can upgrade your account's functionality without creating new addresses
* Account recovery and multi-signature features become accessible

For Developers:

* Build reusable contract logic that any EOA can adopt
* Create sophisticated wallet experiences without complex infrastructure
* Implement account abstraction features with protocol-level support
* Maintain backward compatibility while adding advanced functionality

### Looking Forward
EIP-7702 is just the beginning. As the ecosystem evolves, we can expect to see:

* Smart Wallet Standards: Common interfaces for account authorization that enable interoperability
* Enhanced User Experience: Seamless transitions between different account behaviors
* Gas Optimization: More efficient patterns for sponsored transactions and batch operations
* Security Innovations: Novel approaches to account recovery and multi-party authentication

The beauty of EIP-7702 lies in its simplicity and reversibility. 
Users aren't locked into permanent changes, instead they can experiment with different contract implementations, revert to standard EOA behavior, or upgrade to newer contract versions as they become available.

This upgrade marks Ethereum's first concrete step toward full account abstraction,
but it does so in a way that preserves the fundamental nature of EOAs while
dramatically expanding their capabilities.
As more developers build on this foundation and more users adopt these new patterns, EIP-7702 will likely be remembered as the moment Ethereum accounts truly came alive.

The future of Ethereum accounts is here, and it's more flexible, powerful, and user-friendly than ever before.

---

Edit (23.05.2025):
I haven't put much thought into the relayer example, but if you look closely at
the transaction type 4, you see that the `authorizationList` is a list of
authorizations, which means you can set multiple authorizations at once.

Therefore, I changed the code to set the pointer to the contract address
for two accounts at once, two distinct users. Additionally I changed the `to`
field to the relayer address.

That means, the relayer can trigger a smart contract and set at the same time
the pointer for different users -> batching of authorizations.
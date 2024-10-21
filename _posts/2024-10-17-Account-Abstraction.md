---
layout: post
title: "Ethereum Account Abstraction"
---
# Account Abstraction

Today we are going to have a look in Account Abstraction.

Account Abstraction is a concept in blockchain technology, particularly within the context of Ethereum, that aims
to simplify and enhance how accounts and transactions are managed. We are going to have a
look into [ERC-4337: Account Abstraction Using Alt Mempool](https://eips.ethereum.org/EIPS/eip-4337)
because it does not required changes to the underlying Consensus Layer.
But before we can go into Account Abstraction, we need to understand the current state of Ethereum accounts.

## Background on Ethereum Accounts
In Ethereum, there are two types of accounts:

 1. Externally Owned Accounts (EOAs): Controlled by private keys, typically used by individuals.
 2. Contract Accounts: Represent smart contracts and are controlled by their code.

Currently, EOAs initiate transactions by signing it using their private key and pay for gas
(the fee for executing operations on the network) using the native coin. The network verifies the signature and processes the transaction accordingly.

Contract accounts on the other hand cannot initiate transactions on their own; 
they can only execute transactions triggered by EOAs.

What's the problem with EOAs?

* Lack of flexibility: EOAs are limited in terms of what they can do and how they can interact with the blockchain.
* Security concerns: EOAs are vulnerable to attacks and account loss due to the reliance on private keys.
* Usability issues: EOAs require users to manage private keys, which can be cumbersome and error-prone.
* Onboarding challenges: Complex for new blockchain users, since they would need to acquire native coins to pay for gas.

So we see that there are some issues with the current state of Ethereum accounts.

## Account Abstraction: What Is It?

Account Abstraction aims to generalize and unify the functionality of both EOAs and 
contract accounts, making the distinction between them less rigid. The primary goals of account abstraction is to:

 * Enhance Flexibility: Allow custom logic for validating transactions and handling account operations.
 * Improve Security: Enable more sophisticated security measures, such as multi-signature schemes and social recovery mechanisms.
 * Increase Usability: Simplify user interactions with the blockchain, making it more accessible to non-technical users.

## How Does Account Abstraction Work?

Let's have a look on how Account Abstraction works.

Based on the ERC-4337 proposal, an Alt mempool is introduced which stores 
so called User Operations (UOps). That means that users submit UOps to the separate mempool
instead of submitting transactions to the main mempool. 

Bundlers, which are operating the Alt mempool, bundle a number of UOps into a classic EOA transaction
and submit it to the main mempool, targeting a so called EntryPoint contract.

The EntryPoint contract verifies and executes the bundles UOps by going through some steps (reduced to the most important ones):

1. Create Smart Contract Wallet if required
2. Calculate the maximal possible fee the sender account needs to pay
3. Passes UOps to sender account, which verifies signature and pays the fee if it considers the operation valid
4. Validate the sender account’s deposit in the Entry Point contract is high enough to cover the max possible cost
5. Call sender account with UOp data to be executed
6. Deduct sender account deposit by the cost of the UOp
7. Refund sender account with excess gas cost that was pre-charged
8. After execution of all calls, pay the collected fees from all UOps to the bundler's provided address

And that's that. Here a picture to visualize the process:

![Account Abstraction](/assets/images/AccountAbstraction.png)

Let's talk shortly about the deposit in step 6 - since this seems a bit weird at first { ovo }.

To ensure that bundlers are compensated for paying gas fees, the EntryPoint contract introduces a deposit mechanism.
When a user or smart contract wallet interacts with the EntryPoint contract, they must maintain
a deposit in the contract to cover the costs of their operations.

The deposit prevents scenarios where a bundler might process user operations but left
without a way to recover gas costs, especially when users don't directly pay for gas.

A benefit is that a user doesn't need to hold any native coin their wallet to interact with the blockchain.

## Benefits of Account Abstraction

* Enhanced Security: Customizable security measures can be implemented at the account level, reducing the risk of account compromise.
* User-Friendly Experience: Features like social recovery and sponsored-transactions make the blockchain more accessible to everyday users.
* Innovative Use Cases: Developers can create new types of applications and services that require custom transaction logic, expanding the potential use cases for blockchain technology.

Let's have a look at a potential use case for account abstraction:

Imagine a corporate account that requires multiple approvals for a high-value transaction. With account abstraction, the sender account could be programmed to require signatures from three out of five designated executives before the transaction is considered valid. This adds an extra layer of security and reflects real-world organizational practices.

## Challenges and Considerations

* Complexity: Implementing account abstraction introduces additional complexity in terms of smart contract development and transaction handling.
* Compatibility: Ensuring that new abstraction mechanisms are compatible with existing infrastructure and do not disrupt the current ecosystem is crucial.
* Security Risks: As with any smart contract-based solution, vulnerabilities in the contract logic could be exploited, so rigorous testing and security audits are necessary.

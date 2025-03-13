---
layout: post
title: "Secure Gas Forwarding in Smart Contracts: Preventing Insufficient Gas Griefing Attacks"
tags: Solidity Security
---

# Secure Gas Forwarding in Smart Contracts: Preventing Insufficient Gas Griefing Attacks

When developing smart contracts that forward calls to other contracts, gas management becomes a critical security concern. One particularly subtle vulnerability is the "insufficient gas griefing attack," where a malicious actor can manipulate gas forwarding to cause subcalls to fail while the main transaction succeeds. This article explores this vulnerability and presents a solution for it.


## The Problem: Insufficient Gas Griefing

Consider a relayer (like an off-chain service) or proxy contract that forwards calls to other contracts on behalf of users. This pattern is common in:

* Meta-transactions
* Relayer networks
* Multi-signature wallets
* Contract upgradability patterns
* Cross-chain messaging systems

The vulnerability arises when a malicious actor intentionally provides just enough gas for the main transaction to complete but insufficient gas for the forwarded call to succeed. This creates several problematic scenarios:

* Silent Failures: The main transaction succeeds, but the intended operation fails
* State Inconsistency: Partial execution can leave contract state in an inconsistent state
* Economic Attacks: Attackers can manipulate transaction outcomes while paying minimal gas fees
* Denial of Service: Valid transactions appear to succeed but don't accomplish their intended purpose

Let's examine a naive implementation that is vulnerable to this attack:

```solidity
// VULNERABLE IMPLEMENTATION - DO NOT USE
function forwardCall(address target, bytes memory data) external {
    // No gas checks
    (bool success, ) = target.call(data);
    require(success, "Call failed");
}
```

In this implementation, an attacker could provide just enough gas for the forwarding contract to execute, but not enough for the target contract to complete its operations.

## Naive gas left check

One common approach to prevent insufficient gas griefing is to check the gas how much gas is left right before the subcall:

```solidity
// UNSAFE GAS CHECK - DO NOT USE
function forwardCall(address target, bytes memory data, uint256 gasLimit) external {
    // Verify that sufficient gas is available
    require(gasleft() >= gasLimit, "Insufficient gas");
    // Make the call with specified gas
    (bool success, ) = target.call{gas: gasLimit}(data);
}
```

While this approach seems reasonable, it's not secure. The problem is that `gasleft()` returns the gas available at the time of the check and does not take the dynamic gas costs of the `call` operation into account. This means that an attacker can still manipulate the gas forwarding to their advantage.

## The EVM Gas Stipend Mechanism
To understand the solution, we need to understand how Ethereum's EVM handles gas forwarding. When a contract calls another contract:

* The EVM reserves 1/64th of the remaining gas for operations after the call returns
* At most 63/64ths of the available gas is forwarded to the called contract
* If the called contract consumes all forwarded gas, it reverts with an out-of-gas error
* The calling contract continues execution with the reserved 1/64th gas

This 63/64 rule is crucial for developing a secure gas checking mechanism.

## The Solution
Instead of trying to predict gas needs beforehand (which is nearly impossible due to the dynamic nature of EVM execution and the dynamic gas costs of the call opcode), they verify adequate gas was provided after the subcall completes.

Here's an implementation:

```solidity
// UNSAFE GAS CHECK - DO NOT USE
function forwardCall(address target, bytes memory data, uint256 gasLimit) external {
    // Make the call with specified gas
    (bool success, ) = target.call{gas: gasLimit}(data);

    require(gasleft() >= gasLimit / 63, "Error");
}
```

### The Mathematical Insight
The brilliance of this solution lies in the mathematical relationship between the requested gas and the gas remaining after the call:

* Let `X` be the gas available before the call, such that the call gets at most `X * 63 / 64` (see EIP150).
    * `X = gasLeftBeforeCall - callCosts`
    * Since the gas costs for the call opcode are dynamic, we can't know `X`, but what we want is: `X * 63 / 64 >= forwardedGas`.
* Let `Y` be the gas available after the call: `Y = gasLeftBeforeCall - callCosts - forwardedGas`
* This leads to: `Y = X - forwardedGas`
    * `Y + forwardedGas = X`
* Since `X * 63 / 64 >= forwardedGas` and therefore `X >= (forwardedGas * 64) / 63`, we get:
    * `Y + ForwardedGas >= (forwardedGas * 64) / 63`
    * `Y >= (forwardedGas * 64) / 63 - forwardedGas`
    * `Y >= forwardedGas / 63`

Uff, that was a lot of math! Now, by checking if `gasleft() >= forwardedGas / 63`, we're effectively verifying:

* A relayer or proxy contract forwards to little gas to the target contract, the verification will fail
    * We can punish the relayer or just revert the transaction
* If the the subcall failed due to running out of gas and the verification holds, we know the target contract used more gas than was forwarded
    * The relayer can't be blamed for this and maybe even rewarded for doing their job correctly


## Implementing Secure Gas Forwarding
Here's a complete, secure implementation of a gas-aware forwarding contract:

```solidity
solidityCopy// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SecureForwarder {
    event CallForwarded(address indexed target, bool success, uint256 gasUsed);
    
    struct ForwardRequest {
        address target;
        bytes data;
        uint256 gasLimit;
    }
    
    function forward(ForwardRequest calldata request) external {    
        // Make the call with specified gas
        (bool success, ) = request.target.call{gas: request.gasLimit}(request.data);
        // use require to check
        require(gasleft() >= request.gasLimit / 63, "Error");
    }

    function forward2(ForwardRequest calldata request) external {
        // Make the call with specified gas
        (bool success, ) = request.target.call{gas: request.gasLimit}(request.data);
        uint256 gasLeftAfter = gasleft();
        // use a function to check
        _checkForwardedGas(gasLeftAfter, request);
    }

    function _checkForwardedGas(uint256 gasLeft, ForwardRequest calldata request) private pure {
        if (gasLeft < request.gasLimit / 63) {
            assembly ("memory-safe") {
                invalid()
            }
        }

}
```

# Conclusion

Gas forwarding is a critical aspect of many smart contract designs, but it can introduce subtle vulnerabilities if not handled correctly. The insufficient gas griefing attack is a prime example of how gas management can be exploited to manipulate transaction outcomes. By understanding the EVM's gas stipend mechanism and implementing a secure gas checking mechanism, developers can prevent these attacks and ensure the integrity of their smart contracts.

This article has been written using the following resources:

* [ERC2771Forwarder.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/metatx/ERC2771Forwarder.sol)
* [Ethereum Gas Danger](https://ronan.eth.limo/blog/ethereum-gas-dangers/)
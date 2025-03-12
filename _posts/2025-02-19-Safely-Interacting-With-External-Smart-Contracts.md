---
layout: post
title: "Safely Interacting with External Smart Contracts: A Developer's Guide"
tags: Solidity Security 
---

# Safely Interacting with External Smart Contracts: A Developer's Guide

When you are a Smart Contract developer for EVM-based blockchains, you will 
inevitably encounter situations where your contracts need to interact with
other Smart Contracts on the chain.

Whether it's a simple ERC20 token transfer, a more complex DeFi protocol
interaction, or implementing a relayer service, calling functions on external
contracts is a fundamental skill. However, these interactions come with
significant security considerations that every developer should understand.

## What is the EVM?

The Ethereum Virtual Machine (EVM) is the runtime environment for
smart contracts on Ethereum and compatible blockchains like Polygon,
Binance Smart Chain, and Avalanche. When your contract calls another contract,
this interaction happens within the EVM's execution context, which enforces
rules about gas consumption, call depth, and error handling.

## Why Safe Contract Interaction Matters

Interacting with external contracts introduces several risks:

1. **Security vulnerabilities** - Malicious contracts can exploit poorly designed interaction patterns
2. **Gas consumption issues** - External calls can consume unexpected amounts of gas
3. **Transaction failures** - A revert in the called contract can cause your entire transaction to fail
4. **Reentrancy attacks** - External calls may allow other contracts to call back into your contract before the original operation completes

In this post, I will show you how to interact with Smart Contracts in a safe
way, how to handle gas forwarding effectively, and the potential problems with
try-catch blocks.

We are doing all this under the assumption that we do not trust the contract
we are calling and therefore do not know what code is being executed.

Let the games begin!

## Interacting with Smart Contracts

Let's start by examining a simple example of how to interact with a 
Smart Contract from another Smart Contract.

```solidity
pragma solidity ^0.8.0;

// Interface defining the minimum required functions from ERC20 standard
// Only the transfer function is needed for this example
interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
}

contract MyContract {
    // State variable to hold reference to the ERC20 token contract
    IERC20 public token;

    // Initialize the contract with the address of the token we want to interact with
    constructor(address _token) {
        token = IERC20(_token);
    }

    // Basic implementation - directly calls external contract with no safety measures
    // WARNING: This approach has security vulnerabilities discussed below
    function transfer(address _recipient, uint256 _amount) public {
        // This external call will revert our entire transaction if it fails
        token.transfer(_recipient, _amount);
        // Any code here won't execute if the transfer fails
    }
}
```

Looks simple, right? We define the interface of the ERC20 token and use it in
our `MyContract` contract by defining a state variable `token` of type `IERC20`.

In the constructor, we set the `token` state variable to the address of
the ERC20 token we want to interact with.

In the `transfer` function, we call the `transfer` function of the ERC20
token contract with the `_recipient` and `_amount` parameters.

### Potential Problems

However, there are several issues with this basic approach:

* If the ERC20 transfer function reverts, the whole transaction reverts.
    * This could be undesirable if we want to execute some code after the call to transfer.
* If the ERC20 transfer function does not exist as defined in the interface, the whole transaction reverts.

We can see that a revert in the called contract will also revert our
transaction and therefore won't save the changes made. Sometimes this is
the desired behavior, but sometimes we want to handle the revert and execute
some additional code.

## Using Try-Catch Blocks

This is where the `try-catch` block comes into play. Introduced in Solidity
0.6, try-catch blocks allow us to handle reverts from external calls gracefully.

Here is the code with a `try-catch` block:

```solidity
pragma solidity ^0.8.26;

import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
}

contract MyContract {
    IERC20 public token;
    using SafeERC20 for IERC20;
    
    // Event to log transfer results - useful for monitoring and debugging
    event TransferResult(bool success, string message);

    constructor(address _token) {
        token = IERC20(_token);
    }

    // Improved implementation with error handling
    function transfer(address _recipient, uint256 _amount) public {
        // try-catch block allows us to handle errors from the external call
        try token.safeTransfer(_recipient, _amount) returns (bool result) {
            emit TransferResult(true, "Transfer succeeded");
        } 
        // This catch block handles standard reverts with error messages
        catch Error(string memory reason) {
            // Log the error for transparency
            emit TransferResult(false, reason);
            
            // We choose to revert here, but we could alternatively:
            // 1. Continue execution with a fallback logic
            // 2. Set a state variable to track the failure
            // 3. Try an alternative transfer method
            revert(reason);
        } 
        // This catch block handles all other errors (panics, custom errors, etc.)
        catch {
            emit TransferResult(false, "Unknown error in transfer");
            revert("Transfer failed with unknown error");
        }
    }
}
```

Our `transfer` function now uses a `try-catch` block to handle the revert of
the ERC20 token contract. If the `transfer` function reverts, we can handle
the revert and execute some additional code.

### Understanding Catch Clause Types and Gas Usage

It's important to understand that different catch clause types have
significant implications for gas consumption. Let's examine this with a
practical example:

```solidity
// Two different ways to handle external call errors

contract Test4 {
    event Test(uint);
    function ops(BadGuy badGuy) public {
        try badGuy.youveActivateMyTrapCard{gas: 10000000}() {
            emit Test(0);
        } catch {
            emit Test(1);
        }
    }
}

interface IBadGuy {
    function youveActivateMyTrapCard() external pure returns (uint);
}

contract Test5 {
    event Test(uint);
    function ops(address badGuy) public {
        try IBadGuy(badGuy).youveActivateMyTrapCard{gas: 10000000}() returns (uint abc) {
            emit Test(abc);
        } catch (bytes memory reason) {
            emit Test(1);
        }
    }
}
```

When both contracts call this malicious contract:

```solidity
contract BadGuy {
    function youveActivateMyTrapCard() external pure returns (bytes memory) {
        assembly{
            revert(0, 200000)
        }
    }
}
```

The gas consumption is drastically different:
- Test4 uses ~97,875 gas
- Test5 uses ~211,709 gas

This begs the question: Why the difference? Let's break it down.

#### Why the Difference?

1. **Generic catch vs. Data-capturing catch**:
   - Test4 uses a generic `catch {}` block that ignores all revert data
   - Test5 uses `catch (bytes memory reason)` which attempts to capture 200,000 bytes of revert data

2. **Memory Operations**:
   - Capturing and copying large amounts of revert data requires:
     - Memory allocation + potential memory expansion
     - Copy operations
     - Additional processing

#### Memory Expansion in EVM and their costs

The EVM uses a simple byte-addressable memory model that's initialized as a 
zero-filled byte array for each transaction. This memory is volatile and only
exists during contract execution - it's not persisted between transactions like
storage.

Memory is accessed in 32-byte (256-bit) words, which aligns with the EVM's
word size. When a contract needs to read or write to memory, it must specify
both the location and the size of the data.

Now the question is: How much does it cost to expand memory?

In Appendix H of the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf),
we find the following formula for memory expansion costs:

```
Cmem(μi')-Cmem(μi)
```
where `μi` is the number of words (256 bits) in memory before an opcode is
executed and `μi'` is the number of words in memory after the opcode is
executed.

The function `Cmem` is defined in the Ethereum Yellow Paper as follows:

```
Cmem(a) = Gmemory × a + ⌊a^2 ÷ 512⌋
```
where `⌊x⌋` is the floor function, which returns the largest integer that is
still not larger than the value. When `a < √512`, `a^2 < 512`, and the result of the
floor function is zero. So for the first 22 words (704 bytes), the cost rises
linearly with the number of memory words required and beyond that point, the
cost is proportional to the square of the amount of memory.

#### Gas Costs for Memory Expansion due to Memory Bomb

Now that we know how memory expansion costs are calculated, let's see how
much gas is consumed by the memory expansion in the two contracts.

In the first contract, the `catch {}` block does not capture any revert data,
so the memory expansion is minimal. The gas cost for the memory expansion is
therefore low.

In the second contract, the `catch (bytes memory reason)` block attempts to
capture 200,000 bytes of revert data. This requires a significant amount of
memory expansion, which results in a higher gas cost.

Let's assume that we need to expand memory by 200,000 bytes. The gas cost for
memory expansion is calculated as follows:

```
Cmem(μi')-Cmem(μi) = Gmemory × a + ⌊a^2 ÷ 512⌋ - Gmemory × b - ⌊b^2 ÷ 512⌋
                   = Gmemory × 200,000 + ⌊200,000^2 ÷ 512⌋ - Gmemory × 0 - ⌊0^2 ÷ 512⌋
                   = 200,000 × 3 + ⌊40,000,000 ÷ 512⌋
                   = 600,000 + 78,125
                   = 678,125
```

This is the gas cost for memory expansion due to the memory bomb in the second
contract. The memory expansion cost is significant and contributes to the
higher gas consumption in the second contract.

#### Best Practice

When handling potentially malicious contracts:
1. Use the generic `catch {}` block unless you specifically need the revert reason
2. If you must capture revert data, consider limiting how much you'll process
3. Be aware that large revert data can be used to perform gas griefing attacks

This example illustrates that even a small change in your error handling approach can lead to significant gas efficiency differences when interacting with external contracts.

## Handling Untrusted Contracts

A critical question to ask yourself: Do you trust the ERC20 token contract
you're interacting with? Do you know what code is being executed?

If you do, great! If you don't, you should be aware of the risks of
calling external contracts.

Here are two examples of malicious ERC20 token contracts:

```solidity
pragma solidity ^0.8.26;

// EXAMPLE 1: Gas exhaustion attack through excessive memory allocation
contract MaliciousToken {
    function transfer(address recipient, uint256 amount) public returns (bool) {
        assembly {
            // This attempts to return 10MB of data in the revert reason
            // The caller must process this data, consuming enormous gas
            // This is a gas griefing attack that can drain all available gas
            revert(0, 10000000)  // First param: memory pointer, Second param: size (10MB)
        }
    }
}
```

Or:

```solidity
pragma solidity ^0.8.26;

// EXAMPLE 2: Transaction failure through invalid opcode
contract MaliciousToken {
    function transfer(address recipient, uint256 amount) public returns (bool) {
        assembly {
            // The 'invalid' opcode immediately terminates execution
            // Similar to a revert but without returning any data
            // This consumes all available gas and provides no useful error message
            invalid()
        }
    }
```

The first example reverts. The nasty part about the revert is that the second
parameter, here the `10000000`, is the `totalMemorySize` which we intend to
return to the caller. That means that the caller has to copy 10MB of data to
get the revert reason. This is a so-called gas attack and is going to eat up
all the gas of the caller.

The second example uses the `invalid` opcode which is going to revert the
whole transaction. This is also a gas attack and is going to eat up all
the gas of the caller.


## Limiting Gas with External Calls

As we can see, a malicious contract can still eat up all the gas of the caller.
Instead, we can limit the forwarded gas to a reasonable amount, so that the
caller has enough gas left to handle the revert and execute some additional code.

Here is an example of how to limit the forwarded gas:

```solidity
pragma solidity ^0.8.26;

interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
}

contract MyContract {
    IERC20 public token;
    
    // Events for monitoring both successful and failed transfers
    event TransferSuccess(address recipient, uint256 amount);
    event TransferFailure(address recipient, uint256 amount, string reason);

    constructor(address _token) {
        token = IERC20(_token);
    }

    // Implementation with gas limiting to protect against gas griefing
    function transfer(address _recipient, uint256 _amount) public {
        // The gas option limits how much gas the external call can consume
        // 10000 gas is typically enough for a standard ERC20 transfer
        // CAUTION: More complex tokens might need more gas to function properly
        try token.transfer{gas: 10000}(_recipient, _amount) returns (bool result) {
            require(result, "Transfer returned false");
            emit TransferSuccess(_recipient, _amount);
        } catch Error(string memory reason) {
            // Log the failure but we can still continue execution
            emit TransferFailure(_recipient, _amount, reason);
            
            // We still revert here, but we've protected ourselves from
            // excessive gas consumption in the external call
            revert(reason);
        } catch {
            emit TransferFailure(_recipient, _amount, "Unknown error");
            revert("Unknown error in transfer");
        }
        
        // Additional code here would execute even if the external call 
        // consumed all of its allocated gas (but still reverted)
    }
}
```

It's just a small change, but it can make a big difference. We limited the
forwarded gas to `10000` and the callee can only consume this amount of gas.
The caller has enough gas left to handle the revert and execute some
additional code.

## The excessiveSafeCall Pattern

While limiting gas and using try-catch provides good protection, there are
scenarios where you need even more control over external calls, especially when
dealing with potentially malicious contracts.
This is where the excessiveSafeCall pattern comes in.

### What is excessiveSafeCall?

The excessiveSafeCall pattern combines several safety techniques into a
comprehensive approach for making external calls. It provides:

1. **Gas limiting** - To prevent gas griefing attacks
2. **Return data size limiting** - To prevent memory-based attacks
3. **Proper error handling** - Without exposing your contract to excessive gas consumption
4. **Flexible fallback options** - For graceful degradation when external calls fail

### Implementation Example

Here's a sample implementation of the excessiveSafeCall pattern:

```solidity
pragma solidity ^0.8.26;

// Imports the ExcessivelySafeCall library which provides safer ways to make external calls
import "https://github.com/nomad-xyz/ExcessivelySafeCall/blob/main/src/ExcessivelySafeCall.sol";

// A malicious contract designed to cause problems
contract BadGuy {
    // This function is deliberately designed to waste gas
    // The name suggests it's a trap for unsuspecting contracts
    function youveActivateMyTrapCard() external pure returns (bytes memory) {
        assembly{
            // Attempts to revert with an extremely large memory value
            // This is an attack vector that could cause out-of-gas errors in calling contracts
            revert(0, 1000000)
        }
    }
}

contract MyContract {
    // Address of the ERC20 token contract
    address public token;
    
    // Imports the library functions for the address type
    using ExcessivelySafeCall for address;
    
    // Events for monitoring both successful and failed transfers
    event TransferSuccess(address recipient, uint256 amount);
    event TransferFailure(address recipient, uint256 amount, string reason);
    
    // Sets the token address during contract deployment
    constructor(address _token) {
        token = _token;
    }
    
    // Function to transfer tokens to a recipient
    function transfer(address _recipient, uint256 _amount) public {
        bool success;
        bytes memory ret;
        
        // Makes a "gas-limited" call to the token contract's transfer function
        // This uses ExcessivelySafeCall to protect against various attack vectors
        (success, ret) = token.excessivelySafeCall(
            10000, // gas to be forwarded - limits gas to prevent DOS attacks
            0,     // coins to be forwarded - no ETH is being sent with this call
            32,    // Copy no more than 32 bytes of return data - prevents memory DoS attacks
            abi.encodeWithSelector(
                IERC20.transfer.selector, // Calls the transfer function on the token contract
                _recipient,               // Address receiving the tokens
                _amount                   // Amount of tokens to transfer
            )
        );
        
        if (success) {
            emit TransferSuccess(_recipient, _amount);
        } else {
            emit TransferFailure(_recipient, _amount, "Transfer failed");
        }
    }
}
```

### Key Benefits of excessiveSafeCall

1. **Fine-grained Control**: Using assembly allows precise control over how external calls are made and processed.

2. **Memory Protection**: By limiting the return data size, you prevent attackers from forcing your contract to allocate excessive memory, which can lead to out-of-gas errors.

3. **Custom Error Handling**: You can process errors in a way that matches your application's needs without exposing yourself to gas attacks.

4. **Graceful Degradation**: When external calls fail, your contract can continue operating with fallback logic rather than aborting the entire transaction.

### When to Use excessiveSafeCall

Consider using the excessiveSafeCall pattern when:

- Interacting with untrusted or unknown contracts (you really do not trust them)
- Building systems that need to be resilient to malicious actors
- Working with contracts that might return large amounts of data
- Developing protocols that need to gracefully handle external failures


## Conclusion

Interacting with external contracts is a fundamental part of smart contract
development, but it comes with significant security considerations.
By using try-catch blocks, limiting gas forwarding, and choosing the appropriate
catch clause type you can significantly reduce the risks associated with
external calls.

Remember:
- Always question whether you trust the contracts you're interacting with
- Use gas limits to protect against gas griefing attacks
- Choose the appropriate catch clause based on your needs (generic catch for gas efficiency)
- Implement the excessiveSafeCall pattern for maximum control over external interactions
- Limit return data size to prevent memory-based attacks
- Leverage established libraries like OpenZeppelin's SafeERC20

By keeping these principles in mind, you'll be better equipped to build secure
and resilient smart contracts that can safely interact with the broader ecosystem.

Happy coding!

Huge thank you to my Pantos colleague Dănuț Ilisei for his time to explore this
topic together valuable feedback on the article :)

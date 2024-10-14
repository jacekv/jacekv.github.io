## ERC-165: Standard Interface Detection in EVM Smart Contracts

If you ever had to write some code which interacts with a smart contract, you
know that you need to know the contract's ABI (Application Binary Interface) to
be able to interact with it. This is a JSON file that describes the functions
and events of the contract, and it's generated by the Solidity compiler when
you compile your contract.

But what if you don't have the ABI but know that it is an ERC20 contract?
But is is capped (ERC20Capped), burnable (ERC20Burnable),
and/or pausable (ERC20Pausable)?

Another example in Pantos context: Pantos has a Token Creator service, which
allows users without development skills to create their own PANDAS token.
The user is able select if the token should be Pausable and/or Burnable.
The Pantos Team got into a situation where all PANDAS community token contracts
had to be migrated from one blockchain to another and the team  tried to pause
those contracts on the source blockchain. But not all of those contracts
were pausable - they needed to build an exception block to catch a failing
call to the `pause` function.

Here is where ERC-165 comes in. It is a standard interface that allows you to
query a contract to know if it implements a specific interface.

## ERC-165 Interface

The ERC-165 interface is very simple. It defines a single function:

```solidity
interface ERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}
```

The interface identifier for this `supportsInterface` interface is `0x01ffc9a7`.
You can calculate this by running `bytes4(keccak256('supportsInterface(bytes4)'));`
or using the Selector contract above.

Any contract that implements the `supportsInterface` function will return:
 * `true` when `interfaceID` is `0x01ffc9a7` (EIP165 interface)
 * `false` when `interfaceID` is `0xffffffff`
 * `true` for any other `interfaceID` this contract implements
 * `false` for any other `interfaceID`

This function must return a bool and use at most 30,000 gas.

## ERC-165 Usage Example

Here an example implementation of the ERC-165:

```solidity
pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/introspection/ERC165.sol";


interface Beer {
    function isTasty() external returns (bool);
}

contract Botle is Beer, ERC165 {
    bool public isFull;

    constructor () {
        isFull = true;
    }

    function drink() public {
        isFull = false;
    }

    function isTasty() public pure returns (bool) {
        // hell yeah
        return true;
    }

    function supportsInterface(bytes4 interfaceID) public view virtual override returns (bool) {
        return
          super.supportsInterface(interfaceID) || // ERC165
          interfaceID == this.isFull.selector ||
          interfaceID == this.drink.selector ||
          interfaceID == type(Beer).interfaceId;
    }
}
```

You can simply copy and paste this contract into Remix and deploy it. Then you
can use the `supportsInterface` function to query if the contract implements
the `Beer` interface.

Here are some function selectors to test with:

```
0x01ffc9a7 - true supportsInterface(byte4)
0xffffffff - false
0xbabd3d9a - true isFull
0x755db161 - true isTasty
0x321dedb9 - true drink
0x4a022624 - false transferToZero(uint256)	
```

## Conclusion

If you are writing code which interacts with smart contracts or write smart
contracts that might be used by others, you should consider implementing the
ERC-165 interface. This will allow users to query your contract to know if it
implements a specific interface, which can be very useful in many situations.

## References
ERC165: - [https://eips.ethereum.org/EIPS/eip-165](https://eips.ethereum.org/EIPS/eip-165)
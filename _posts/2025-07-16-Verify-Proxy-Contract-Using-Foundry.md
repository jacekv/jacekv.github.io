---
layout: post
title: "Verifying Proxy Contracts Using Foundry"
tags: Ethereum EVM Solidity Foundry
---

This is more a write up for myself in case I forget how to verify already deployed
contracts using foundry, especially proxy contracts :) 

## Verifying Proxy Contracts

Before we verify a proxy contract, we will verify the implementation contract first.

In foundry you would run the following instruction:

```bash
forge verify-contract \
    --watch \
    --chain <chain name or alternative --chain-id <id>> \
    <logic contract address> \
    src/<Contract File Name>.sol:<Contract Name> \
    --etherscan-api-key <Etherscan API Key V2>
```

This will take a moment and you should get ideally a message with:

```bash
Contract verification status:
Response: `OK`
Details: `Pass - Verified`
Contract successfully verified
```

Now head to the explorer of the network you are using and check the contract address. You should see the
contract verified with the source code and ABI available.

Great, now we are going to verify the proxy contract.
Let's assume you are using UUPS proxies, which is the recommended way to do it.

In order to verify the proxy contract itself, you will have to do a bit more.

CHeck out the command:

```bash
forge verify-contract \
  --watch \
  --chain <chain name or alternative --chain-id <id>> \
  --constructor-args $(cast abi-encode "constructor(address,bytes)" "<address of logic contract>" $(cast calldata "initialize(<arg types you defined>)" \
    <args for initialize>)) \
  --etherscan-api-key <Etherscan API Key V2> \
  <address of proxy contract> \
  lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Proxy.sol:ERC1967Proxy
```

As you can see, we defined a path to the `ERC1967Proxy.sol` file, which is the OpenZeppelin implementation of the UUPS proxy.

Your smart contract file might have something like `@openzeppelin/contracts/...`
In that case you should also have a `remappings.txt` file in your project root directory.

You will replace the `@openzeppelin` part with the path defined in your `remappings.txt` file.

And that is it! You should now see the proxy contract verified on the explorer as well.
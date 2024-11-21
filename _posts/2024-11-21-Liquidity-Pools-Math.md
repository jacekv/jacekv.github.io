---
layout: post
title: "Liquidity Pool Math"
---

# Liquidity Pool Math

One day I decided to have a look into the math behind Liquidity Pools. I am 
going to explain what formulas are used to calculate the tokens received when
providing liquidity to a pool or while creating a pool.

Let's start first with what a Liquidity Pool actually is.

## Liquidity Pool

A liquidity pool is a smart contract that contains a reserve of two tokens.
The pool allows users to trade between the two tokens, providing liquidity
to the pool in exchange for a share of the trading fees.

In comparison to traditional order book exchanges, which contains a list of open
buy and sell orders, liquidity pools use an automated market maker (AMM) algorithm
to determine the price of the tokens based on the ratio of the two tokens in the pool.

But let's not go in AMM algorithms here, but rather focus on the math behind the
liquidity pools.

## Liquidity Pool Math

### Creation of a pool

When creating a pool, the user needs to provide an equal value of both tokens.
In exchange, the user receives liquidity pool tokens, which represent the user's
share of the pool.

The formula to calculate the amount of liquidity pool tokens received is:

```
amount of liquidity pool tokens = sqrt(amount of token A * amount of token B)
```

which is the geometric mean of the two token amounts.

Example: If you provide 100 tokens of token A and 100 tokens of token B, you would
receive 100 liquidity pool tokens, since `sqrt(100 * 100) = 100`. [[Uniswap V2 Core](https://app.uniswap.org/whitepaper.pdf)]

Now let's have a look on how to calculate the amount of tokens received when providing
liquidity to an existing pool.

### Adding liquidity to a pool

When adding liquidity to an existing pool, the user needs to provide an equal value
of both tokens. In exchange, the user receives liquidity pool tokens, which represent
the user's share of the pool.

The formula to calculate the amount of liquidity pool tokens received is:

```
amount of liquidity pool tokens = min(amount of token A / reserve of token A, amount of token B / reserve of token B) * total supply of liquidity pool tokens
```

where `reserve of token A` and `reserve of token B` are the amounts of the two tokens
in the pool and `total supply of liquidity pool tokens` is the total supply of the pool's
liquidity pool tokens.

Example: If you provide 50 tokens of token A and 50 tokens of token B to a pool with
100 tokens of token A and 100 tokens of token B, you would receive 50 liquidity pool tokens,
since `min(50 / 100, 50 / 100) * 100 = 50`. [[Uniswap V2 Core](https://app.uniswap.org/whitepaper.pdf)]

### Redeem liquidity from a pool

When redeeming liquidity from a pool, the user receives an equal value of both tokens
in the pool. In exchange, the user burns their liquidity pool tokens.

The formula to calculate the amount of tokens received is:

```
amount of token A = (LP tokens to burn / total amount of LP tokens) * reserve of token A
amount of token B = (LP tokens to burn / total amount of LP tokens) * reserve of token B
```

where `LP tokens to burn` is the amount of liquidity pool tokens to burn, `total amount of LP tokens`
is the total supply of the pool's liquidity pool tokens, and `reserve of token A` and `reserve of token B`
are the amounts of the two tokens in the pool.

Example: If you burn 50 liquidity pool tokens from a pool with 100 tokens of token A and 100 tokens of token B,
you would receive 50 tokens of token A and 50 tokens of token B, since `(50 / 100) * 100 = 50`. [[Uniswap V2 Core](https://app.uniswap.org/whitepaper.pdf)]

### Fees in liquidity pools

In addition to providing liquidity to the pool, users can earn trading fees by staking their liquidity pool tokens.
The trading fees are collected from users who trade on the platform and are added back into the pool. The fees are
distributed to liquidity providers based on their share of the pool.

## Conclusion

Liquidity pools are a key component of decentralized exchanges, providing a way for users to trade tokens
without the need for a centralized order book. Understanding the math behind liquidity pools can help users
make informed decisions when providing liquidity or trading on these platforms.

I hope this post was helpful in explaining the math behind liquidity pools. If you have any questions or feedback,
feel free to reach out to me on [Twitter](https://twitter.com/Jacekv).
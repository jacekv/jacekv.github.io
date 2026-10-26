---
layout: post
title: "How Bounce.Tech Builds Leveraged Tokens on Hyperliquid"
description: "A look at how Bounce.Tech implements leveraged tokens on Hyperliquid: LeveragedToken.sol, HyperEVM and HyperCore, band rebalancing, API wallets, and the idle EVM reserve."
tags: Hyperliquid HyperEVM HyperCore DeFi Solidity LeveragedTokens
math: true
---

# How Bounce.Tech Builds Leveraged Tokens on Hyperliquid

In my [previous post]({% post_url 2026-06-02-Understanding-Perpetual-Futures-And-Leveraged-Tokens %})
I went through the basics of perpetual futures, leverage, and leveraged tokens.
This time I want to look at how a real protocol implements them. The protocol
we are going to look at is [Bounce.Tech](https://bounce.tech), which runs on
Hyperliquid.

Quick recap of why leveraged tokens exist at all: holding a leveraged perp
position is a lot of work. You have to watch your margin, top up collateral
when the market moves against you, keep an eye on funding, and hope that no
wick liquidates you while you are asleep.

A **Leveraged Token (LT)** wraps all of that into a plain ERC-20 token. You hold
the token, the protocol manages the position.

The idea is not new. Earlier LTs on Ethereum and other L2s had a few problems
though:

* Gas was too expensive to rebalance often.
* Blocks were too slow, so rebalancing bots were often late during fast moves.
* Liquidity on AMMs was thin, so every rebalance came with slippage, which
  slowly ate into the token's value.

Bounce gets around this by splitting the work between Hyperliquid's two layers.
The smart contracts live on **HyperEVM**, while the actual trading happens on
**HyperCore**, Hyperliquid's own L1 order book.

The result is a non-custodial leveraged token protocol with more than 200
markets (crypto, equities, commodities, FX) and up to 10x leverage, all as
regular ERC-20 tokens.

In this post we'll go through the architecture, look at the code of
`LeveragedToken.sol`, see how rebalancing works, and where the money for
withdrawals actually comes from.

## Table of contents

1. [What Is a Leveraged Token?](#1-what-is-a-leveraged-token)
2. [Architecture: HyperEVM and HyperCore](#2-architecture-hyperevm-and-hypercore)
3. [A Look at the Code: LeveragedToken.sol](#3-a-look-at-the-code-leveragedtokensol)
4. [Rebalancing](#4-rebalancing)
5. [Where Does the Rebalancing Capital Come From?](#5-where-does-the-rebalancing-capital-come-from)
6. [The Idle EVM Reserve and Withdrawals](#6-the-idle-evm-reserve-and-withdrawals)
7. [Compounding vs. Volatility Decay](#7-compounding-vs-volatility-decay)
8. [Guardrails and Fees](#8-guardrails-and-fees)
9. [Conclusion](#conclusion)

## 1. What Is a Leveraged Token?

A Leveraged Token is an ERC-20 token that keeps a target leverage (e.g. 2x, 3x,
or 5x, long or short) on an underlying asset. You don't need a margin account
to hold it.

If you buy `BTC3L` (Bitcoin 3x Long) or `NVDA2S` (Nvidia 2x Short), it just
sits in your wallet like any other token. If the underlying goes up 5%, a 3x
long token should go up roughly 15%.

```text
    Standard Perpetual Futures                 Bounce Leveraged Tokens (LT)
┌─────────────────────────────────┐        ┌─────────────────────────────────┐
│ • Must post & manage collateral │        │ • Standard ERC-20 spot asset    │
│ • Personal liquidation price    │   vs.  │ • Automated band rebalancing    │
│ • Isolated in exchange trading  │        │ • Composable across HyperEVM    │
│ • Risk of total ($0) wipeout    │        │ • Mitigates sudden liquidations │
└─────────────────────────────────┘        └─────────────────────────────────┘
```

So what do you get out of it?

* **No personal liquidation price.** With a 5x perp, a 20% drop wipes you out.
  An LT instead reduces its leverage as the market falls, so you still have
  something left if the price recovers.
* **No babysitting.** No topping up collateral, no stop-losses, no health
  ratios to watch.
* **It's just an ERC-20.** You can send it to another wallet, trade it on a DEX,
  keep it on a hardware wallet, or use it as collateral in a lending protocol.

## 2. Architecture: HyperEVM and HyperCore

Hyperliquid has two layers that work together:

* **HyperCore:** Hyperliquid's L1 with a native order book (CLOB). It handles up
  to 200,000 orders per second with sub-second finality.
* **HyperEVM:** An EVM-compatible environment where you can deploy normal
  Solidity contracts.

Bounce does not try to build a perp exchange in Solidity. The contracts on
HyperEVM take care of the accounting and the token, while the actual trades
are executed on HyperCore.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        HyperEVM (Solidity Layer)                       │
│                                                                        │
│   User deposits USDC ──► [ LeveragedToken.sol (ERC-20 Proxy) ]         │
│                                     │                                  │
│                 ┌───────────────────┴───────────────────┐              │
│                 ▼                                       ▼              │
│      CoreWriterLib (Precompile)           IHyperliquidHandler (Oracle) │
└─────────────────┼───────────────────────────────────────┼──────────────┘
                  │ Cross-engine calls                    │ State reads
┌─────────────────┼───────────────────────────────────────┼──────────────┐
│                 ▼                                       ▼              │
│                           HyperCore (L1 Engine)                        │
│                                                                        │
│   • Single aggregated perpetual contract per LT tier                   │
│   • Backed by Hyperliquid liquidity (HLP Vault & Market Makers)        │
│   • Supports native DEX (index 0) and custom HIP-3 perpetual markets   │
└────────────────────────────────────────────────────────────────────────┘
                  ▲
                  │ Fast order execution (place / cancel orders)
        [ Off-Chain Keeper Bot ]
        (Scoped API Wallets: AGENT_0, AGENT_1, AGENT_2)
```

### One position per token

Bounce does not open a separate position for each user. All deposits into one
token (e.g. everyone holding `ETH3L`) are pooled and back a single perp
position on HyperCore.

So if 1,000 users deposit a total of 1,000,000 USDC into `ETH3L`, the protocol
holds one 3,000,000 USDC notional ETH long on HyperCore. The other side of that
trade is Hyperliquid's own liquidity (the HLP vault and market makers).

### Precompiles

To move data and funds between the two layers, Bounce uses **Hyperliquid
precompiles**. These are built into the node itself and can be called from
Solidity.

With `CoreWriterLib` (for writing) and `IHyperliquidHandler` (for reading), a
contract on HyperEVM can read its margin and position on HyperCore, bridge USDC
between the layers, and register keys that are allowed to trade.

## 3. A Look at the Code: LeveragedToken.sol

Let's have a look at the contract itself and see how Bounce handles accounting,
security, and different markets.

### Total assets and exchange rate

Every Bounce token is backed by two things: idle USDC sitting in the contract on
HyperEVM, and the equity of its perp position on HyperCore. The total value
(NAV) is calculated like this:

```solidity
function totalAssets() public view override returns (uint256) {
    uint256 baseAssetBalance_ = baseAssetBalance() + _blockBridging();
    uint256 hyperliquidAssets_ = _hyperliquidUsdc();
    return baseAssetBalance_ + hyperliquidAssets_;
}

function _hyperliquidUsdc() internal view returns (uint256) {
    LeveragedTokenStorage storage $ = _getLeveragedTokenStorage();
    IHyperliquidHandler hyperliquidHandler_ = $.globalStorage.hyperliquidHandler();
    uint256 spotUsdc_ = hyperliquidHandler_.spotUsdc(address(this));
    uint256 perpUsdc_ = hyperliquidHandler_.perpUsdc(address(this), $.perpDexIndex);
    return spotUsdc_ + perpUsdc_;
}
```

* `baseAssetBalance()`: USDC sitting in the contract on HyperEVM. This is what
  instant redemptions are paid from.
* `_blockBridging()`: USDC that is currently being bridged between HyperEVM and
  HyperCore in this block. Without it, the NAV would briefly look too low while
  funds are in transit.
* `_hyperliquidUsdc()`: Asks the `HyperliquidHandler` precompile for the
  contract's collateral and unrealized PnL on HyperCore (`spotUsdc + perpUsdc`).

From `totalAssets()` the exchange rate is calculated with 18 decimals:

```solidity
function exchangeRate() public view override returns (uint256) {
    if (totalSupply() == 0) return 1e18;
    return totalAssets().scaleFrom(_baseAsset().decimals()).div(totalSupply());
}
```

If you read my post on [vaults]({% post_url 2025-12-22-ERC4626-Vaults %}),
this should look familiar. It's basically the same shares-to-assets idea.

### API wallets

Here is the part I found the most interesting. Someone has to rebalance the
position, and that someone is an off-chain bot. Usually, giving a bot control
over user funds is a big risk: if its private key leaks, the funds are gone.

Bounce uses Hyperliquid's native **API wallets** for this:

```solidity
function addAgent(address agent_) external override onlyOwner {
    ...
    $.agents[slot_] = agent_;
    $.isAgent[agent_] = true;
    string memory agentName_ = _getAgentName(slot_);
    CoreWriterLib.addApiWallet(agent_, agentName_); // Native precompile registration
    emit AddAgent(slot_, agent_);
}
```

An API wallet on HyperCore can only do a limited set of things:

* **Allowed:** placing market/limit orders, cancelling orders, changing
  leverage.
* **Not allowed:** withdrawing or transferring funds. So even if the bot's key
  leaks, the attacker can't take the collateral.

The contract has 3 agent slots (`_AGENT_SLOTS = 3`). To rotate or remove a key,
`removeAgent` calls `CoreWriterLib.addApiWallet(address(0), agentName_)`, which
removes its trading rights on HyperCore right away.

### Streaming fees

Bounce charges a 2% yearly streaming fee on the notional of the position.
Instead of having a cron job collect it, the fee is calculated in
`_checkpoint()` whenever someone interacts with the contract:

```solidity
function _checkpoint(uint256 to_) internal {
    ...
    uint256 timeElapsed_ = to_ - $.lastCheckpoint;
    uint256 percentOfYear_ = (timeElapsed_ * 1e18) / 365 days;
    uint256 streamingFee_ = $.globalStorage.streamingFee();
    uint256 totalAssets_ = totalAssets();
    // Annual fee scales with target leverage (notional exposure)
    uint256 annualFee_ = totalAssets_.mul(streamingFee_).mul($.targetLeverage);
    uint256 periodFee_ = annualFee_.mul(percentOfYear_);
    _payStreamingFee(periodFee_);
    $.lastCheckpoint = to_;
}
```

Note that `annualFee_` is multiplied by `targetLeverage`. A 3x token has 3 USDC
of exposure for every 1 USDC of collateral, so the fee is charged on the full
exposure, not just the collateral.

### HIP-3 markets

With HIP-3 (Hyperliquid Improvement Proposal 3), anyone can launch their own
perp DEX on HyperCore. Bounce supports these out of the box:

```solidity
function _getPerpDexIndex(uint32 marketId_) internal pure returns (uint32) {
    if (marketId_ < _HIP_3_DEX_STRIDE) return 0; // Default Hyperliquid DEX
    if (marketId_ < _HIP_3_BASE) revert InvalidMarketId();
    uint32 perpDexIndex_ = (marketId_ - _HIP_3_BASE) / _HIP_3_DEX_STRIDE;
    if (perpDexIndex_ == 0) revert InvalidMarketId();
    return perpDexIndex_;
}
```

Market IDs below `_HIP_3_DEX_STRIDE` belong to the default Hyperliquid DEX, and
everything above `_HIP_3_BASE` is mapped to a HIP-3 DEX index. This means Bounce
is not tied to the main Hyperliquid exchange and can launch tokens for any
HIP-3 market.

## 4. Rebalancing

Rebalancing is the heart of every leveraged token. I covered the theory in the
previous post, so here is just a short recap before we look at what Bounce does.

### Leverage drift

With perps, your leverage changes every time the price moves:

$$\text{Effective Leverage} = \frac{\text{Notional Position Value}}{\text{Collateral (Margin)}}$$

Let's say you deposit 1,000 USDC into `ETH2L` (2x long):

* **Margin:** 1,000 USDC
* **Position:** 2,000 USDC worth of ETH, so leverage is $\frac{2000}{1000} = 2.0$x

**ETH drops 20%:**

* The position loses 400 USDC. Margin is now 600, notional is 1,600.
* Leverage: $\frac{1600}{600} \approx 2.67$x
* As the price falls, leverage goes up, and you get closer to liquidation.

**ETH goes up 20%:**

* The position gains 400 USDC. Margin is now 1,400, notional is 2,400.
* Leverage: $\frac{2400}{1400} \approx 1.71$x
* As the price rises, leverage goes down, and you get less upside than you
  signed up for.

### Band rebalancing

Bounce does not rebalance on a fixed schedule (like once a day at 00:00 UTC).
A fixed time is easy to front-run, and the position is unprotected if the
market crashes in between. Instead, Bounce rebalances whenever leverage leaves
a band around the target:

```text
     2.74x                                3.0x                                3.35x
◄───────|───────────────────────────────────▲───────────────────────────────────|───────►
      LOWER                               TARGET                              UPPER
      BOUND                                                                   BOUND
  (Asset pumped;                          "Calm"                         (Asset dumped;
   Leverage dropped)                       Zone                           Leverage spiked)
        │                                                                        │
        ▼                                                                        ▼
   BUY MORE PERPS                                                           SELL PERPS
(Compound gains back to 3x)                                              (De-risk back to 3x)
```

The width of the band depends on how volatile the asset is:

| Category | Example Asset | Target Leverage | Allowed Range | Why |
|---|---|---|---|---|
| Commodities / FX | `GOLD2L`, `SP5002L` | 2.0x | 1.94x – 2.06x | **Tight (~3%):** these assets move slowly, so the token can track closely. |
| Major Crypto | `BTC3L`, `ETH3L` | 3.0x | 2.74x – 3.35x | **Medium:** a trade-off between tracking and number of rebalances. |
| High Volatility | `BTC5L` | 5.0x | 4.20x – 6.33x | **Wider:** normal crypto swings shouldn't trigger a rebalance every time. |
| Memecoins | `DOGE5S`, `FARTCOIN5L` | 5.0x | 3.53x – 7.89x | **Wide:** memecoins swing a lot, a tight band would mostly burn fees. |

### What happens during a rebalance

```text
[ HyperCore Price Feeds ]
           │
           ▼
[ Off-Chain Keeper Bot ] ──► Reads targetLeverage & totalAssets() from contract
           │
           │ (Checks live Notional / Margin: 3.36x breach detected!)
           ▼
[ Signs Order via API Wallet (AGENT_0) ]
           │
           ▼
[ HyperCore Orderbook ]
           └── Executes Perp Market Order against HLP / MMs
           └── Resets position size until Leverage == 3.0x
```

1. **Detect:** Bitcoin pulls back and `BTC3L` leverage climbs to 3.36x, just
   above the 3.35x upper bound.
2. **Size the trade:** The bot calculates how much notional it has to sell:

   $$\Delta \text{Notional} = \text{Current Notional} - (\text{Margin} \times 3.0)$$

3. **Execute:** The bot signs a market order with its API wallet and sends it
   directly to HyperCore.
4. **Settle:** The order is filled against the HLP vault and market makers. The
   position is smaller, and leverage is back at 3.0x. No EVM transaction, no gas.

## 5. Where Does the Rebalancing Capital Come From?

When I first looked at this I asked myself: if the bot has to buy more perps
during a rally, where does the money come from? Is there a cash reserve for it?

Turns out, **no extra capital is needed**. That's just how perps work on
HyperCore.

### During a rally

1. The price goes up and the long position is in profit.
2. On HyperCore, unrealized profit counts towards the account's equity right
   away.
3. More equity means more free margin.
4. To go from e.g. 2.5x back to 3.0x, the bot simply buys more perps using that
   profit. Nothing has to be bridged from HyperEVM.

### During a drop

1. The price falls, equity drops, and leverage goes up.
2. To get back to 3.0x, the bot doesn't add capital. It sells part of the
   position.
3. A smaller position means lower leverage, back inside the band.

## 6. The Idle EVM Reserve and Withdrawals

So rebalancing doesn't need a cash reserve. But Bounce does keep some idle USDC
on HyperEVM (`baseAssetBalance()`). It is there for a different reason: fast,
cheap withdrawals.

```text
                   TOTAL VAULT VALUE (totalAssets)
┌──────────────────────────────────────┬──────────────────────────────────────┐
│        HyperEVM Reserve Buffer       │          HyperCore Collateral        │
│          (baseAssetBalance)          │          (_hyperliquidUsdc)          │
│                                      │                                      │
│  • Liquid USDC in Solidity contract  │  • Active margin on orderbook        │
│  • Powers instant atomic redeem()    │  • Backs perp notional exposure      │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

This means there is always a trade-off. Every USDC that sits idle on HyperEVM
for withdrawals is a USDC that isn't working as margin on HyperCore.

### Path 1: Instant redemption (`redeem`)

If the contract holds enough idle USDC, you can redeem in a single transaction:

```solidity
function redeem(address to_, uint256 ltAmount_, uint256 minBaseAmount_) external override returns (uint256) {
    _checkpoint();
    uint256 baseAmount_ = ltToBaseAmount(ltAmount_);
    if (baseAmount_ > baseAssetBalance()) revert InsufficientBalance(); // Checks idle buffer
    ...
    _burn(msg.sender, ltAmount_);
    _baseAsset().safeTransfer(to_, afterFees_);
    return afterFees_;
}
```

If there isn't enough (`baseAmount_ > baseAssetBalance()`), the call reverts
with `InsufficientBalance()`.

### Path 2: Queued redemption (`prepareRedeem` → `executeRedemptions`)

If you want to withdraw more than the reserve holds, the USDC first has to come
back from HyperCore:

```text
                  User initiates Redemption
                             │
            Is baseAmount <= idle EVM buffer?
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
            [YES]                          [NO]
        Atomic Redeem               Async Redemption Flow
     (Instant 1-step exit)                  │
                                            ▼
                                1. User calls prepareRedeem()
                                2. Off-chain bot bridges USDC
                                3. Executor calls executeRedemptions()
                                   (Fallback: cancelRedeem() after 1 hr)
```

1. **Queue:** You call `prepareRedeem(ltAmount_)`. Your tokens are locked in the
   contract and recorded with `_addCredit(msg.sender, ltAmount_)`.
2. **Close and bridge:** The bot picks up the `PrepareRedeem` event, closes the
   matching part of the position on HyperCore, and calls `bridgeToEvm()`.
3. **Settle:** Once the USDC is on HyperEVM, the bot calls
   `executeRedemptions(users)`, which burns the locked tokens and pays out the
   USDC.
4. **Escape hatch (`cancelRedeem`):** What if the bot is down? There is a
   timeout:

   ```solidity
   uint256 internal constant _CANCEL_REDEMPTION_DELAY = 1 hours;
   ```

   If your redemption isn't settled within 1 hour, you can call
   `cancelRedeem()` and get your tokens back.

### What this means under stress

The split between the idle reserve and HyperCore margin leads to a few
situations worth knowing about:

* **New deposits:** Deposited USDC first lands in the EVM reserve. To actually
  back the new tokens with perps, the bot bridges it to HyperCore with
  `bridgeToCore()`. If it bridges too much, the reserve shrinks and more users
  end up in the queue.
* **Lots of redemptions at once:** Each instant redemption takes from
  `baseAssetBalance()`. Once it's empty, everyone after that has to use the
  queue.
* **Market crashes:** If margin on HyperCore gets close to the maintenance
  level, the bot can call `bridgeToCore()` and move the idle EVM reserve over as
  extra collateral.
  * This keeps the position alive.
  * But it also empties the reserve, so all withdrawals go through the queue
    until things calm down.

## 7. Compounding vs. Volatility Decay

How an LT performs depends a lot on how the market moves. I went into detail on
this in the previous post, so here is the short version.

### Trending markets

When the market moves in one direction, LTs do better than a static perp
position. The bot keeps adding to the position as it makes money, so gains
compound. A 2x LT in a steady rally ends up higher than a static 2x perp over
the same time.

### Sideways markets

When the market goes up and down, LTs lose value (volatility decay).

If the asset drops 10%, the bot sells part of the position at a loss. If it
then recovers 10%, the token recovers with a smaller position. Do this often
enough and the "sell low, buy high" pattern slowly eats into the token's value.

**Example with 2x leverage**

| Scenario | Underlying Asset | Static 2x Perp | 2x Leveraged Token (LT) |
|---|---:|---:|---:|
| Trend (Day 1) | +10% | +20% | +20% |
| Trend (Day 2) | +10% | +42% | +44% (compounding) |
| Choppy (Day 1) | -10% | -20% | -20% |
| Choppy (Day 2) | +10% | -2% | -4% (decay) |

> So Bounce LTs are made for short-term directional trades, not for holding
> for years.

## 8. Guardrails and Fees

A few more things Bounce has in place:

* **Standby mode (250 USDC minimum TVL):** HyperCore has minimum order sizes. If
  a token's TVL drops below 250 USDC, it goes into standby. The funds stay in
  the contract as plain USDC without any perp exposure. Once TVL is above 300
  USDC again, the contract opens the position again.
* **Streaming fee (2% per year):** Charged in `_checkpoint()` on the notional of
  the position, as we saw above.
* **Closing fee (0.3% × target leverage):** Paid when you exit. That's 0.6% for a
  2x token, 0.9% for 3x, and 1.5% for 5x.
* **Referrals:** A separate `Referrals.sol` contract lets affiliates earn
  rewards and gives traders a rebate on the closing fee.

## Conclusion

Bounce is a nice example of how to use both layers of Hyperliquid:

* **HyperCore does the trading:** fast matching, sub-second finality, and deep
  liquidity.
* **HyperEVM does the rest:** ERC-20 tokens, transparent accounting, and
  upgradeable contracts.
* **API wallets keep the bot in check:** it can rebalance and protect the
  position, but it can never withdraw user funds.

What I like about the design is that all the complicated parts (margin,
rebalancing, liquidation risk) are hidden behind a normal ERC-20 token. You
still need to understand volatility decay before you buy one, but you don't
need to watch a margin account all day.

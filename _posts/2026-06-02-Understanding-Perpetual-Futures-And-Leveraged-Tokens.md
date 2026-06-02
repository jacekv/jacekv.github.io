---
layout: post
title: "Understanding Perpetual Futures and Leveraged Tokens: A Beginner's Guide"
tags: Futures Perpetual Futures Tokens
---

# Understanding Perpetual Futures and Leveraged Tokens: A Beginner's Guide

*A personal reference on how perps, hedging, and leveraged tokens work, explained from the ground up.*

---

## Table of Contents

1. [Starting with the Basics: What is a Future?](#1-starting-with-the-basics-what-is-a-future)
2. [Futures in Crypto: Betting on Price Direction](#2-futures-in-crypto-betting-on-price-direction)
3. [What is a Perpetual Future ("Perp")?](#3-what-is-a-perpetual-future-perp)
4. [The Funding Rate: How Perps Stay Anchored to Reality](#4-the-funding-rate-how-perps-stay-anchored-to-reality)
5. [Leverage: Amplifying Gains and Losses](#5-leverage-amplifying-gains-and-losses)
6. [Hedging: Using Futures as Insurance](#6-hedging-using-futures-as-insurance)
7. [Leveraged Tokens: Perps Made Simple](#7-leveraged-tokens-perps-made-simple)
8. [Why Hyperliquid Precompiles Matter](#8-why-hyperliquid-precompiles-matter)
9. [TL;DR Cheat Sheet](#tldr-cheat-sheet)
10. [Final Thoughts](#final-thoughts)

---

## 1. Starting with the Basics: What is a Future?

A **futures contract** is an agreement to buy or sell something at a set price on a future date.

**The classic farmer example:**
A farmer agrees in January to sell 1,000 bushels of wheat to a bakery in July at $5/bushel, no matter what wheat actually costs in July. Both sides lock in a price *now* to remove uncertainty *later*.

This is the original purpose of futures: **reducing risk** for people who deal in real goods.


## 2. Futures in Crypto: Betting on Price Direction

In crypto, futures aren't usually about delivering an actual asset, they're about **profiting from price movements**.

**Example:**

- BTC is trading at $100,000 today.
- You think it will rise, so you buy a futures contract locking in the $100,000 price.
- A month later, BTC is at $110,000.
- Your contract is now valuable — it lets you "buy" BTC at $100,000 when the market price is $110,000.
- You sell the contract to another trader for ~$10,000 profit.

You never touched actual BTC. You just **bet on the direction** and got paid based on the move.

- **Long** = bet the price will go up
- **Short** = bet the price will go down


## 3. What is a Perpetual Future ("Perp")?

A **perpetual future** (or "perp") is a futures contract with **no expiration date**.

- *Perpetual* = never-ending, continuous (German: **unbefristet** (without a deadline))
- You can hold the position as long as you want.
- Invented by BitMEX around 2016 — now the most-traded instrument in crypto.

But this creates a tricky problem: if a perp never expires, how does its price stay tied to the real asset's price? That's where the **funding rate** comes in — explained next.


## 4. The Funding Rate: How Perps Stay Anchored to Reality

### First: Why is this even a problem?

Remember, a perp has **no expiration date**. This creates a puzzle that doesn't exist with regular futures.

**Regular futures fix themselves automatically:**
A normal futures contract expires on a specific date. On that date, the contract **must** settle at the real (spot) price of the asset. So even if traders push the futures price away from spot during the contract's life, **as expiration approaches, the two prices must converge**. Math forces it.

**Perpetuals have no expiration -> no natural anchor:**
If perps never expire, **there's no forced convergence point**. The perp price could drift away from spot forever and there'd be no mechanism to pull it back.

So perps needed an **artificial mechanism** to keep them anchored to the real price. That mechanism is the **funding rate**.

### What is "the perp price" vs "the spot price"?

Two different markets, two different prices:

- **Spot price** = the actual price of BTC on regular exchanges (e.g., $100,000 on Coinbase where you buy real BTC).
- **Perp price** = the price of the BTC perpetual contract on a perp exchange (e.g., on Hyperliquid).

These are **two separate markets** driven by different traders, so their prices can drift apart.

**Example:**

- Real BTC (spot) = $100,000
- BTC perp = $100,500

The perp is trading $500 *above* spot. Why might that happen?

### Why would the perp price drift away from spot?

Because perp prices are driven by **trader demand**, not the underlying asset.

**Scenario:** Everyone is super bullish on BTC.

- Tons of traders rush to open **long** positions on the perp.
- Buy pressure on the perp pushes its price UP.
- But the spot price is calmer, it's mostly determined by people actually buying real BTC.
- Result: perp price > spot price.

The opposite happens when everyone's bearish, they rush to short the perp, pushing the perp price *below* spot.

### Enter the Funding Rate

The funding rate is a clever **economic incentive** to pull the perp price back to spot.

**How it works:**

Every 8 hours (on most exchanges), traders pay each other directly:

- If **perp > spot** -> the **longs pay the shorts** a small fee.
- If **perp < spot** -> the **shorts pay the longs** a small fee.

> ⚠️ **Important:** The exchange doesn't take this money. It flows **directly between traders** — long traders to short traders, or vice versa.

### Why this fixes the price gap

Let's walk through it slowly with the bullish example:

**Setup:**

- Spot BTC = $100,000
- Perp BTC = $100,500 (too high, bulls dominating)

**Funding rate kicks in:**

- Since perp > spot, **longs must pay shorts** every 8 hours.
- Let's say the funding rate is **+0.05%** per 8 hours.
- A trader holding a $10,000 long position pays $5 every 8 hours to the shorts.

**Now think about the incentives:**

🤔 **For longs:** *"Wait, I have to keep paying $5 every 8 hours just to hold this position? That's expensive. Maybe I should close it, or at least not open new longs."*

🤔 **For shorts:** *"I'm getting paid $5 every 8 hours just for being short! That's free money on top of any profits. I should open more shorts!"*

**What happens next:**

- Some longs close their positions -> less buy pressure on the perp.
- New traders open shorts to collect funding -> more sell pressure on the perp.
- The perp price **falls back toward spot**. ✅

The mismatch corrects itself through pure economic pressure.

### The same thing in reverse

**Setup:**

- Spot BTC = $100,000
- Perp BTC = $99,500 (too low — bears dominating)

**Funding rate flips:**

- Since perp < spot, **shorts pay longs**.

🤔 **For shorts:** *"Why am I paying to hold this short? Maybe I'll close."*
🤔 **For longs:** *"I'm getting paid to be long! Let me open more."*

-> Perp price rises back toward spot. ✅

### A real-world analogy 🏊

Think of it like a **pool of water** with the spot price as the "natural water level":

- If the water rises too high (perp > spot), there's a slow leak draining money from the side that pushed it up (the longs).
- If the water falls too low (perp < spot), there's a slow leak draining from the shorts.

The leak makes it **expensive to keep the price imbalanced**, so traders naturally adjust. The water always wants to return to its natural level — and the funding rate is what makes that happen.

### What does this look like in practice?

On Hyperliquid (and most exchanges), you'll see something like:

Funding rate: +0.0125% (8h)
Next funding in: 2h 34m

This means:

- Right now, the perp is trading **slightly above** spot.
- Longs will pay shorts **0.0125%** of their position size in 2.5 hours.
- That's about **0.0375% per day** or **~13.7% per year** if it stays the same.

In strong bull markets, funding can hit **0.1% per 8 hours = ~110% per year** for longs to pay! That's why funding is sometimes called a "**bull tax**."

### Why this matters for you

If you trade perps, **funding is a real cost or income**:

| Position | Funding Direction | What Happens |
| -------- | ----------------- | ------------ |
| Long     | Positive (bullish)| You **pay**  |
| Short    | Positive (bullish)| You **earn** |
| Long     | Negative (bearish)| You **earn** |
| Short    | Negative (bearish)| You **pay**  |

For **leveraged tokens** (like Bounce):

- If you hold a **long LT** and funding is positive (typical bull market), the token bleeds a tiny bit every 8 hours from funding payments -> **funding rate drag**.
- This is one of the "hidden costs" of holding leveraged tokens long-term.

### Funding rate - TL;DR

The **funding rate** is a periodic payment between longs and shorts that keeps the perp price glued to the real (spot) price. When the perp is too expensive, longs pay shorts, discouraging more longs and encouraging shorts, which pulls the price down. When the perp is too cheap, shorts pay longs, and the price gets pushed back up. The exchange doesn't take this money; **it flows directly between traders**. Without funding, perp prices would drift away from reality forever.

---

## 5. Leverage: Amplifying Gains and Losses

Leverage feels almost magical at first, *"How can I control $1,000 with only $100?"* Let's unpack it properly.

### The Core Idea: Leverage = Borrowing

Leverage works because you're essentially **borrowing exposure** to make a bigger bet than your own cash allows.

You're not really "controlling $1,000 out of thin air", you're putting up $100 of your own money, and the exchange (or protocol) is effectively letting you take a $1,000-sized position, with your $100 acting as collateral.

### A Real-World Analogy: Buying a House 🏠

This is the easiest way to grasp leverage, because almost everyone has heard of it.

Imagine you want to buy a **$200,000 house**, but you only have **$20,000** in savings.

- You put down $20,000 (your **margin / down payment**).
- The bank lends you $180,000 (the **leverage**).
- You now "control" a $200,000 asset with only $20,000 of your own money.
- That's **10x leverage**.

Now watch what happens to your returns:

**Scenario A: House value rises 10% -> $220,000**

- The house is worth $20,000 more.
- You sell, pay back the $180,000 loan, and keep $40,000.
- **You doubled your $20,000 -> +100% return.**
- Without leverage, a 10% rise on a $20,000 investment would only earn you $2,000 (+10%).

**Scenario B: House value falls 10% -> $180,000**

- The house is worth $20,000 less.
- If you sell, you get $180,000, just enough to pay back the loan.
- **Your $20,000 is wiped out -> -100% return = liquidation.**

A **10% move in the asset** became a **100% move in your money**. That's leverage in action.

### Why It Works: The Math

Leverage amplifies returns by a simple factor:

> **Your % return = Asset's % move × Leverage**

| Leverage      | Asset moves +1% | Asset moves -1% | Asset move to liquidation |
| ------------- | --------------- | --------------- | ------------------------- |
| 1x (no lev.)  | +1%             | -1%             | -100%                     |
| 2x            | +2%             | -2%             | -50%                      |
| 5x            | +5%             | -5%             | -20%                      |
| 10x           | +10%            | -10%            | -10%                      |
| 25x           | +25%            | -25%            | -4%                       |
| 100x          | +100%           | -100%           | -1%                       |

Notice the brutal truth: **the higher the leverage, the smaller the move needed to wipe you out.**

At 100x leverage, a tiny 1% price move against you = total loss.

### How It Actually Works on a Crypto Exchange

Unlike a bank loan, you don't actually receive borrowed cash. The exchange just **tracks the position** and your margin acts as collateral.

Here's the flow:

1. You deposit **$100** as **margin** (collateral).
2. You tell the exchange: "Open a 10x long BTC position."
3. The exchange opens a position **as if you had $1,000**.
4. Your profit/loss is calculated on the **full $1,000 notional value**, not your $100.
5. If losses approach your $100 margin, the exchange **liquidates** your position to protect itself.

The exchange isn't risking its own money, your margin covers any potential loss. The moment losses approach your margin, *snap*, position closed.

### Where Does the "Borrowed" Money Come From in Perps?

Here's where perps get interesting:

**There's no actual loan.** Perps are a **zero-sum game** between traders:

- Every long position is matched by a short position.
- When you go 10x long, someone else is going short.
- Your gains = their losses, and vice versa.

The "leverage" is really just **agreement enforcement**: the exchange ensures the loser pays the winner, using the margin both sides put up as collateral.

So leverage in perps isn't borrowing money, it's **borrowing exposure**, enforced by margin requirements and liquidations.

### Why Does the Exchange Allow This?

Three reasons:

1. **They earn fees** on the full $1,000 position, not just your $100.
2. **They collect funding rates** flowing between traders.
3. **They're protected by liquidations**, they close your position before losses exceed your margin, so they're not at risk.

It's a great business model, high volume, high fees, low risk for the platform.

### Why People Use Leverage

Despite the risks, leverage has legitimate uses:

- ✅ **Capital efficiency**: Why tie up $10,000 to get $10,000 of exposure if $1,000 + 10x does the same?
- ✅ **Free up cash for other uses**: The other $9,000 can earn yield or sit as backup.
- ✅ **Amplify high-conviction trades**: If you're confident in a move, leverage scales the reward.
- ✅ **Short selling**: Most short positions inherently involve some leverage.

### The Danger Most Beginners Underestimate

Leverage is often called the **"silent killer"**:

1. **Small moves feel huge**: A normal 5% BTC move feels catastrophic at 20x.
2. **Volatility is constant in crypto**: 5-10% daily moves are routine.
3. **Liquidations are permanent**: You can't "wait it out." Once liquidated, your money is gone, even if the price recovers an hour later.
4. **Emotions amplify with stakes**: Bigger positions = worse decisions under pressure.

A famous saying:

> *"Leverage doesn't create wealth, it accelerates outcomes — good or bad."*

---

## 6. Hedging: Using Futures as Insurance

**Hedging** = taking a position that protects you from losses on another position. Think of it like insurance.

**Farmer's hedge:**

- He plants wheat in January, harvests in July.
- He's worried wheat will crash from $5 -> $2.
- He signs a futures contract to sell July wheat at $5.

Now no matter what happens, he locks in $5/bushel:

- If wheat crashes to $2 -> contract pays him the $3 difference.
- If wheat rises to $8 -> contract costs him $3, but he sells the wheat for more.

He **gave up upside for certainty**. That's hedging.

**Crypto hedge example:**

- You hold 1 BTC long-term but fear a short-term crash.
- Open a short perp on 1 BTC.
- If BTC drops, the short profits offset the spot losses.
- Your BTC's value is temporarily "frozen."

| Hedging                     | Speculating               |
| --------------------------- | ------------------------- |
| Reduces risk                | Takes on risk             |
| Protects existing exposure  | Creates new exposure      |
| Goal: stability             | Goal: profit              |

---

## 7. Leveraged Tokens: Perps Made Simple

A **leveraged token (LT)** is an **ERC-20 token that gives you leveraged exposure to an asset**, without having to manage margin or worry about liquidation.

Behind the scenes, the protocol:

1. Takes your deposit.
2. Opens a leveraged perp position (e.g., 10x on Hyperliquid).
3. Hands you a token representing that position.
4. **Automatically rebalances** to keep the leverage constant.

### Key Benefits

- ✅ **No liquidation risk**: rebalancing prevents wipeouts from a single move
- ✅ **No margin management**: just hold the token
- ✅ **ERC-20 composability**: usable across DeFi (LPing, collateral, transfers)
- ✅ **Deep liquidity** via Hyperliquid's perp markets

### Key Risks

- ⚠️ **Volatility decay**: In sideways/choppy markets, repeated rebalancing erodes value. Higher leverage = worse decay.
- ⚠️ **Not for HODLing**: These are for **short-term directional bets**, not long-term holds. Over months, returns can diverge dramatically from "10x the underlying."
- ⚠️ **Funding rate drag**: Long LTs pay funding when funding is positive; shorts pay when negative.
- ⚠️ **Rebalancing mechanics matter**: Time-based vs. threshold-based rebalancing affects efficiency.

### The Trade-Off vs. Regular Perps

Bringing it all back together:

Regular perps give you raw leveraged exposure, but you face the **cliff of liquidation**, one bad move and you're zeroed out.

Leveraged tokens trade that cliff for **death by a thousand cuts**, you can't get wiped out by a single move, but you slowly bleed value in choppy markets through volatility decay.

Different risk profiles, different use cases. To really understand why, we need to look at how rebalancing actually works, next.

---

## 8. How Rebalancing Actually Works

This is the most important mechanic to understand if you want to use leveraged tokens properly. Most people skip it and get confused later when their token "doesn't track 10x." Let's go deep.

### First: Why does rebalancing need to happen at all?

Recall from the leverage section:

> **Leverage = Notional / Margin**

The catch is: **this ratio changes automatically every time the price moves.** And that's a problem if you want to keep a constant 10x exposure.

### The "Drift Problem": Why leverage doesn't stay constant on its own

**Starting position (10x long BTC):**

- BTC price = $100,000
- Your margin (collateral) = $100
- Notional position = $1,000 (you control 0.01 BTC)
- Leverage = $1,000 / $100 = **10x** ✅

Now let's see what happens when BTC moves.

**Case A: BTC goes UP 5% -> $105,000** 🎉

- Your 0.01 BTC is now worth $1,050 (notional).
- Your PandL is +$50, so your margin grew from $100 -> $150.
- **New leverage = $1,050 / $150 = 7x** ❌

Even though you "started" at 10x, you're now only at 7x. Your exposure is *less leveraged* than it should be.

**Case B: BTC goes DOWN 5% -> $95,000** 😬

- Your 0.01 BTC is now worth $950 (notional).
- Your PandL is -$50, so your margin shrunk from $100 -> $50.
- **New leverage = $950 / $50 = 19x** ❌

Now you're way *more* leveraged than you signed up for, and dangerously close to liquidation.

### The pattern

| Price moves... | Margin... | Notional... | Effective leverage... |
| -------------- | --------- | ----------- | --------------------- |
| In your favor  | Grows     | Grows less  | **Decreases** 📉      |
| Against you    | Shrinks   | Shrinks less| **Increases** 📈      |

This drift is bad for two reasons:

1. **After favorable moves**, you're under-leveraged -> you miss out on the gains you signed up for.
2. **After adverse moves**, you're over-leveraged -> too close to liquidation, exactly when you can't afford it.

So the protocol must **rebalance** to bring leverage back to the target 10x.

### How rebalancing fixes the drift

The protocol's job is simple: **adjust the position size so that leverage = target again**.

> **Target notional = Target leverage × Current margin**

Let's redo the examples with rebalancing.

**Case A revisited: BTC went UP 5% -> margin is now $150**

- Target notional should be: 10 × $150 = **$1,500**
- Current notional: $1,050
- **The protocol needs to BUY $450 more exposure** (about 0.0043 BTC at $105k).
- New leverage = $1,500 / $150 = **10x** ✅

**Case B revisited: BTC went DOWN 5% -> margin is now $50**

- Target notional should be: 10 × $50 = **$500**
- Current notional: $950
- **The protocol needs to SELL $450 of exposure** (about 0.0047 BTC at $95k).
- New leverage = $500 / $50 = **10x** ✅

The leverage is restored. The token can continue tracking 10x from this new baseline.

### Notice the brutal pattern: "Buy High, Sell Low"

Look carefully at what the protocol just did:

- **Price went UP** -> protocol **bought more** (at a higher price 📈)
- **Price went DOWN** -> protocol **sold some** (at a lower price 📉)

This is the exact opposite of what a good trader would do.

**The protocol is forced to "buy high, sell low" mechanically, every single time.**

This is not a bug. It's a **mathematical necessity** to keep leverage constant. But it has a famous side effect:

➡️ **Volatility decay** (also called "rebalancing decay" or "constant leverage drag").

### Volatility Decay: The Hidden Cost

Volatility decay means that **in choppy/sideways markets, a leveraged token loses value even if the underlying ends up flat**.

**A clean example:**

Let's say you hold a 3x long ETH leveraged token. ETH price action:

- Day 0: ETH = $100, LT = $100
- Day 1: ETH goes UP 10% -> $110
- Day 2: ETH goes DOWN 9.09% (back to $100)

So **ETH ended exactly where it started**, flat over two days. You'd expect your 3x token to also be flat, right?

Let's check.

**Day 1 (no rebalance yet):**

- ETH +10%, so LT should be +30% (3x)
- LT = $130

**Day 1 rebalance:** position adjusted to maintain 3x exposure based on the new $130 value.

**Day 2 (ETH falls 9.09%):**

- Your LT now had 3x exposure on $130
- LT loses 3 × 9.09% = -27.27%
- LT = $130 × (1 - 0.2727) = **$94.55**

ETH ended at $100 (flat). Your token ended at **$94.55**, a **5.45% loss** for zero net price movement! 😱

The bigger the swings, the worse the decay:

| Leverage | After +10% then -9.09% | After +20% then -16.67% |
| -------- | ---------------------- | ----------------------- |
| 1x (spot)| 0%                     | 0%                      |
| 2x       | -1.8%                  | -6.7%                   |
| 3x       | -5.5%                  | -15%                    |
| 5x       | -13.6%                 | -33.3%                  |
| 10x      | -39.6%                 | -80%                    |

**Higher leverage = exponentially worse decay.**

This is why leveraged tokens are **terrible for long-term holding** in choppy markets.

### When Does Rebalancing Happen? (The Trigger)

There are three main approaches a protocol can use:

**1. Time-based (scheduled) rebalancing**

The protocol rebalances at fixed intervals, for example, every 24 hours.

- ✅ Simple, predictable, easy to model
- ❌ Leverage can drift a lot between rebalances -> if BTC moves 30% during the day, you're miles from target
- ❌ Triggers even when not needed -> unnecessary trading costs

This was the old FTX/Binance BULL/BEAR model.

**2. Threshold-based rebalancing**

The protocol only rebalances when leverage drifts beyond a set band, for example, when actual leverage moves outside ±20% of target (so 8x or 12x for a 10x token).

- ✅ Only rebalances when actually needed -> cheaper, less decay in calm markets
- ✅ Tighter bands = more accurate tracking
- ❌ Big sudden moves can push you past the threshold before rebalance kicks in
- ❌ More complex to predict

This is generally considered the **best modern approach**.

**3. Hybrid (combination)**

Use both: rebalance whenever leverage drifts past a threshold, **and** at least once per day even if nothing happened.

- ✅ Catches both gradual drift and sudden moves
- ✅ Most robust
- ❌ Slightly more rebalancing overhead

Many newer LTs (likely including Bounce) use a hybrid approach for the best of both worlds.

### Putting It All Together: A Mental Model 🧠

Think of a leveraged token like a **car with a self-correcting steering system** trying to stay on a straight line (the 10x target):

- Every time the road curves (price moves), the car drifts off course.
- The steering automatically corrects to get back on the line.
- But each correction **costs a little gasoline** (trading fees, decay).
- On a straight road (trending market), you barely steer, efficient.
- On a winding road (choppy market), you steer constantly, wastes gas -> decay.

So the LT does what it promises (constant leverage), but you pay for the privilege through **trading costs + volatility decay**, which scale with how choppy the market is.

### Practical Takeaways

- ✅ **Best used in trending markets**: strong, sustained moves in one direction. Decay matters less when there's net movement.
- ❌ **Avoid in choppy / sideways markets**: decay eats your value.
- ⏱️ **Short-term holds** (hours to days) -> minimal decay risk.
- 🚫 **Long-term holds** (weeks to months) -> significant divergence from "10x the underlying."
- 🔍 **Always check the rebalancing trigger** of any LT you use, threshold-based is usually best.
- 📊 **Higher leverage = exponentially more decay**: be very careful with anything above 5x.

### Rebalancing — TL;DR

Rebalancing is the protocol's way of keeping leverage constant by **buying more exposure after gains** and **selling exposure after losses**. This protects you from liquidation, but mechanically forces a "buy high, sell low" pattern, causing **volatility decay** in choppy markets. Leveraged tokens are excellent for short-term directional bets, but **terrible for HODLing** because of this decay. The trigger mechanism (time-based, threshold-based, or hybrid) and the underlying execution venue (e.g., Hyperliquid) significantly affect how much decay you suffer.

---

## 9. Why Hyperliquid Precompiles Matter

Rebalancing isn't free. Every time the protocol rebalances, it has to:

1. **Open or close perp position size** on the underlying exchange
2. **Pay trading fees** for that adjustment
3. **Eat any slippage** if the order moves the market

These costs ultimately come out of the token's value, they make decay worse.

Bounce leveraged tokens are built on **Hyperliquid precompiles**, meaning the token contracts interact with Hyperliquid's perp engine **natively at the L1 level**, not through bridges or wrappers.

This gives:

- **Atomic mint/redeem** with real perp positions
- **On-chain verifiability** of backing
- **Native execution** at the L1 level -> lower latency, less slippage
- **Deep liquidity** -> small rebalances barely move the market
- **Low fees** -> less drag per rebalance

This is meaningfully better architecture than:

- Off-chain custodial LTs (the old FTX-style)
- LTs backed by thin AMM-based perps, these would suffer much worse decay just from the cost of rebalancing alone

The execution venue matters a lot for leveraged tokens, because every basis point saved on rebalancing translates directly into less decay for holders.

---

## TL;DR Cheat Sheet

| Term                | Meaning                                                                  |
| ------------------- | ------------------------------------------------------------------------ |
| **Future**          | Contract to buy/sell at a set price on a future date                     |
| **Perp**            | Future with no expiration date                                           |
| **Spot price**      | The actual market price of the asset on regular exchanges                |
| **Funding rate**    | Periodic payment between longs and shorts that keeps perp price = spot   |
| **Long**            | Bet price goes up                                                        |
| **Short**           | Bet price goes down                                                      |
| **Leverage**        | Borrowed exposure, amplifies PandL proportionally                          |
| **Margin**          | Your collateral backing a leveraged position                             |
| **Notional**        | The full position size (margin × leverage)                               |
| **Liquidation**     | Losing your margin when the trade moves too far against you              |
| **Hedging**         | Using a position to offset risk on another position                      |
| **Leveraged Token** | ERC-20 token providing auto-managed leveraged exposure                   |
| **Rebalancing**     | Adjusting position size to keep leverage at the target                   |
| **Volatility decay**| Slow erosion of leveraged token value in choppy markets                  |
| **Funding drag**    | Cost of holding a leveraged token when funding works against your side   |

---

## Final Thoughts

Perps and leveraged tokens are powerful tools, but they're not "set and forget" investments. They're best understood as **precision instruments for directional bets**, not long-term holds.

Leveraged tokens like Bounce's solve real UX problems (no liquidation, no margin management, ERC-20 composability), but they come with their own quirks, especially **volatility decay** and **funding drag**. Always understand what you're holding.

Remember the golden rule:

> *"Leverage doesn't create wealth, it accelerates outcomes — good or bad."*

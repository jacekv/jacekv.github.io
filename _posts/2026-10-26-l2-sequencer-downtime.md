---
layout: post
title: "L2 Sequencer Downtime and Oracles in the OP Stack"
description: "What happens when an OP Stack sequencer goes down and comes back, how push and pull oracles behave during an outage, and why replaying old price updates doesn't work."
tags: Ethereum OP-Stack Optimism L2 Oracles Chainlink Pyth
math: true
---

# L2 Sequencer Downtime and Oracles in the OP Stack

After [setting up my own OP Stack chain]({% post_url 2026-07-10-Setting-Up-An-Optimism-Chain %})
I started wondering what actually happens when the sequencer goes down. When you
build an L2 you mostly think about throughput, gas costs, and EVM compatibility.
But the interesting part is what happens when things break and the chain has to
recover.

A few questions I had:

* How does the OP Stack catch up when the L2 falls behind L1?
* What happens to price oracles while the sequencer is down and while it
  recovers?
* Could someone use old price updates while the sequencer is catching up to the
  current time?

In this post we'll go through how sequencer recovery works, how push oracles
(Chainlink) and pull oracles (Pyth/RedStone) deal with an outage, and why
replaying an old price update doesn't work.

## Table of contents

1. [Block Heads and Timestamps](#1-block-heads-and-timestamps)
2. [Push vs. Pull Oracles During an Outage](#2-push-vs-pull-oracles-during-an-outage)
3. [Replaying Old Prices](#3-replaying-old-prices)
4. [Why the Replay Doesn't Work](#4-why-the-replay-doesnt-work)
5. [Checklist for L2 Builders](#5-checklist-for-l2-builders)

## 1. Block Heads and Timestamps

Before we can talk about recovery, we need to know about the two block heads in
the OP Stack:

| Head | Description |
|---|---|
| **Unsafe L2 Head** (Sequencer) | Blocks the sequencer produces in real time. Not confirmed on L1 yet. |
| **Safe L2 Head** (Derivation) | Blocks that were derived from batches posted to L1. |

Block timestamps in the OP Stack are fixed. Every block is 2 seconds after the
previous one:

$$\text{L2 Block Timestamp} = \text{Genesis Timestamp} + (\text{Block Number} \times 2\text{ seconds})$$

### Catching up after downtime

If the sequencer goes offline, or the chain gets unwound back to an earlier
block (e.g. reverting L2 state back to a canonical L1 origin block), L2 time
falls behind the real time.

So what happens when the sequencer starts again?

* The timestamps it needs to produce are already in the past, so there is no
  reason to wait 2 seconds between blocks.
* It produces the missing blocks as fast as it can (~50–100+ blocks per second)
  until it has caught up.
* Once $$T_{\text{L2}} = T_{\text{WallClock}}$$, it slows down to the normal 1
  block every 2 seconds.

## 2. Push vs. Pull Oracles During an Outage

How an oracle deals with a sequencer outage depends on how it gets its prices
on-chain.

```text
                                ORACLE MODELS
                                      │
             ┌────────────────────────┴────────────────────────┐
             ▼                                                 ▼
    PUSH ORACLES (Chainlink)                     PULL ORACLES (Pyth / RedStone)
    ────────────────────────                     ──────────────────────────────
    • Oracle writes price to L2 storage.         • Signed price payload in tx calldata.
    • Relies on L1 Deposit Queue.                • Verified on-chain during execution.
```

### A. Push oracles (Chainlink)

Chainlink pushes prices into storage on the L2. If the sequencer is down, the L2
RPC doesn't accept transactions, so Chainlink can't write anything to the L2
directly.

Chainlink gets around this by going through L1:

1. A Chainlink node notices the L2 is down and calls `sendMessage()` on
   `L1CrossDomainMessenger`.
2. The message is queued in `OptimismPortal` on Ethereum.
3. When the sequencer comes back, the OP Stack forces it to include all L1
   deposit transactions at the start of the block, **before** any user
   transactions from the L2 mempool.

```text
       RECOVERY BLOCK L2 EXECUTION ORDER (OP Stack Invariant)

       ┌────────────────────────────────────────────────────────┐
       │ Tx 0: L1AttributesDeposited (System Tx)                │
       ├────────────────────────────────────────────────────────┤
       │ Tx 1: Chainlink L1 Deposit "Sequencer IS DOWN"         │ ◄── FORCED EXECUTION FIRST
       ├────────────────────────────────────────────────────────┤
       │ Tx 2: User Swaps / Arbitrage / Liquidations            │ ◄── Blocked by Downtime Flag!
       └────────────────────────────────────────────────────────┘
```

Since Tx 1 runs first, `OptimismSequencerUptimeFeed` is set to `DOWN` before any
user transaction runs. Protocols that read this feed then wait for a grace
period (e.g. 60 minutes) after recovery before allowing liquidations again.

### B. Pull oracles (Pyth and RedStone)

Pull oracles work the other way around. Nobody writes prices to storage all the
time. Instead, the user (or a bot) fetches a signed price update off-chain
(for Pyth that's a VAA) and adds it to the calldata of their transaction.

The contract then checks the price when the transaction runs:

```solidity
// Pyth Price Freshness Check
PythStructs.Price memory price = pyth.getPriceNoOlderThan(priceId, 60);
```

What Pyth checks here is:

$$\text{L2 } \mathtt{block.timestamp} - \text{Pyth } \mathtt{publishTime} \le 60\text{ seconds}$$

If the price is older than 60 seconds, the transaction reverts.

## 3. Replaying Old Prices

Here is the question that got me started on this post.

Let's say the sequencer was down for 12 hours and is now catching up. At some
point it builds a block with a timestamp of yesterday 10:00 PM. What if an
attacker sends a Pyth price update that was signed yesterday at 10:00 PM?

$$\mathtt{L2\ block.timestamp}\text{ (Yesterday 10:00 PM)} - \mathtt{Pyth.publishTime}\text{ (Yesterday 10:00 PM)} = \mathbf{0\text{ seconds}}$$

$$\Delta t = 0\text{s} \le 60\text{s}$$, so `getPriceNoOlderThan(60)` would pass.
The attacker would get to trade at yesterday's price, today.

Sounds bad, right? Turns out it doesn't work, and there are three reasons why.

## 4. Why the Replay Doesn't Work

### Reason 1: Recovery mode (`admin_setRecoverMode`)

While the sequencer is catching up, it runs in recovery mode
(`admin_setRecoverMode true`).

* `op-node` only produces empty blocks to move the L2 timestamp forward.
* User transactions from the mempool are not included in these blocks.

So the attacker's transaction just waits in the mempool until the sequencer has
caught up to the current time. By then the price update is 12 hours old and the
transaction reverts.

```text
  ATTACKER ATTEMPT (L2 Mempool Path)

  Attacker Submits Tx (Yesterday's VAA) ──► Mempool (Blocked in Recovery Mode)
                                                   │
                                                   ▼ Fast-Forward Complete
  Contract Executes at Current Wall-Clock Time ──► Reverts! (VAA is 12h Old)
```

### Reason 2: Checking against L1 time (`0x4200...0015`)

A protocol doesn't have to trust the L2 `block.timestamp` alone. The OP Stack
has an `L1Block` predeploy that knows the timestamp of the current L1 origin
block. If the L2 time is far ahead of L1 time, something is off:

```solidity
// L1-Anchored Oracle Guard
function getGuardedPrice(bytes32 priceId) public returns (PythStructs.Price memory) {
    // 1. Fetch the L1 origin timestamp from the OP Stack L1Block predeploy
    uint256 l1Timestamp = IL1Block(0x4200000000000000000000000000000000000015).timestamp();

    // 2. Reject if L2 block time is too far ahead of L1 time
    require(block.timestamp - l1Timestamp <= 60, "SequencerIsFastForwarding");

    return pyth.getPriceNoOlderThan(priceId, 60);
}
```

Note the order in the subtraction: the L2 timestamp is always greater than or
equal to the L1 origin timestamp. If you write `l1Timestamp - block.timestamp`
the call underflows and reverts every time.

### Reason 3: Going through L1

OK, but what if the attacker skips the L2 mempool and sends the transaction as
an L1 deposit through `OptimismPortal`?

That doesn't help either, because now the L1 timestamp decides:

* **Sent today:** The deposit lands in an L1 block from today. On L2,
  `L1Block.timestamp()` is today, but the price update is from yesterday.
  $$\Delta t = 12\text{ hours} \implies \textbf{reverts}$$
* **Sent yesterday:** Then the deposit was made yesterday and runs at yesterday's
  price. That's just a normal trade at the price of that time, not an exploit.

| Attacker L1 Deposit Strategy | L1 Block Timestamp | Result |
|---|---|---|
| Send today with yesterday's VAA | Today | **Reverts** (12h difference) |
| Send yesterday with yesterday's VAA | Yesterday | **Executes** (price of that time) |

## 5. Checklist for L2 Builders

If you run an OP Stack chain or build DeFi protocols on one, here is what I'd
keep in mind:

* **Set `--sequencer.max-safe-lag=30`:** This stops the sequencer from running
  too far ahead of what has been derived from L1. If L1 derivation falls behind,
  the sequencer pauses, which avoids big reorgs and dropped transactions.
* **Start in recovery mode:** Use `--sequencer.stopped=true` or run
  `admin_setRecoverMode true` when unwinding the database or catching up.
* **Use grace periods:** Lending, borrowing, and liquidation contracts should
  check a sequencer uptime feed like Chainlink's `OptimismSequencerUptimeFeed`
  and wait 30–60 minutes after the sequencer comes back.
* **Run op-node and op-reth separately:** Put them in separate containers so you
  can restart one without taking down the other.

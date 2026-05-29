---
layout: post
title: "Price Is Not Truth: A Field Guide to Oracle Manipulation (bZx → Mango, $135M+ Drained)"
date: 2026-05-29 08:21:12 +0000
---

# Price Is Not Truth: A Field Guide to Oracle Manipulation

Between February 2020 and October 2022, four DeFi protocols lost a combined **north of $135 million** to the same root cause. Not a reentrancy bug. Not a missing access check. Something more fundamental: **they trusted a price that an attacker could move.**

This is the defining vulnerability class of permissionless finance. A smart contract that lends, liquidates, or settles needs to know what an asset is worth — and the moment it sources that number from a market an attacker can push around, the "value" of collateral becomes a variable the attacker controls. Flash loans removed the last barrier: you no longer need capital to move a market, you just need to borrow it for 12 seconds.

Here are four exploits, the exact mechanism behind each, and the defenses that actually hold.

---

## What "oracle manipulation" really means

A price oracle answers one question for a contract: *how much is X worth right now?* There are two ways to answer it badly:

1. **Spot price from an on-chain DEX.** The "price" is just the ratio of reserves in a liquidity pool. Swap enough into the pool and the ratio — and therefore the reported price — moves immediately. With a flash loan, "enough" is unbounded.
2. **Spot price from a thin market.** Even an off-chain or aggregated feed is exploitable if the underlying asset only trades in shallow venues. Move the shallow market, move the feed.

Every exploit below is a variation on one of these two mistakes.

---

## 1. bZx (February 2020) — the template

Two attacks four days apart established the entire playbook.

bZx's Fulcrum margin product used **on-chain DEX spot prices** (sourced via Kyber/Uniswap) to value collateral and positions. In the second, cleaner attack the flow was:

1. Flash-borrow a large amount of ETH.
2. Use part of it to buy sUSD across Kyber and Uniswap, **pumping sUSD's spot price** to roughly double its real value.
3. Post the now-overvalued sUSD as collateral on bZx, which read the manipulated spot price and happily lent out far more ETH than the collateral was actually worth.
4. Repay the flash loan; keep the difference.

Net theft: roughly **$645k** in that attack (and ~$350k in the first). Small numbers in hindsight — but bZx proved that a self-contained, capital-free attack on a spot-price oracle was not theoretical. Everyone took notes, including the attackers.

**Lesson:** a DEX spot price is not a valuation. It is the current state of a pool, and pool state is for sale.

---

## 2. Harvest Finance (October 2020) — manipulating a stable pool

Harvest's yield vaults (fUSDC, fUSDT) deposited user funds into Curve's stablecoin pools. The vault calculated each depositor's share value from the **live state of the Curve pool** — and that state is just balances that a swap can move.

The attack was a deposit/withdraw sandwich, repeated dozens of times:

1. Flash-borrow a huge USDC position.
2. Swap a large amount inside the Curve pool to **skew the pool's balance**, temporarily depressing the apparent price of USDC inside the vault's accounting.
3. Deposit into the Harvest vault at the artificially cheap share price, minting more shares than deserved.
4. Reverse the swap, restoring the pool — now the shares are worth more.
5. Withdraw at the inflated value, pocket the spread, repeat.

Drained: approximately **$24 million** from the USDC and USDT vaults in a single transaction sequence. The pool was "stable," but stability of price is not the same as resistance to manipulation of *accounting derived from* that pool.

**Lesson:** if deposits and withdrawals are priced off instantaneous pool state, an attacker who can move that state for one block can mint value out of nothing.

---

## 3. Cheese Bank (November 2020) — thin-liquidity collateral

Cheese Bank was a Compound fork that accepted its own ecosystem tokens as collateral and priced them using **Uniswap V2 reserves** — i.e., spot price of a thinly traded asset.

1. Flash-borrow ETH.
2. Pump the price of the accepted collateral token by buying into its shallow Uniswap pool.
3. Borrow against the inflated collateral value across the protocol's stablecoin markets.
4. Walk away with the borrowed stables; the collateral left behind is worth a fraction of what was lent.

Loss: roughly **$3.3 million** in USDC, USDT and DAI. The contract's logic was fine. The oracle was a 5-second-old reserve ratio of a token nobody else was trading.

**Lesson:** thin liquidity *is* the vulnerability. The cheaper an asset is to move, the more dangerous it is as collateral.

---

## 4. Mango Markets (October 2022) — manipulating your own mark

The largest and most brazen. On Solana, the attacker manipulated the price of the protocol's *own* asset to inflate the *unrealized profit* the protocol would lend against.

1. Fund two accounts. In one, open a massive **long** MNGO-PERP position; the other took the other side.
2. Aggressively buy MNGO spot across exchanges, ramping the price from roughly $0.03 to ~$0.91 — a thinly traded token, so the move was cheap relative to the prize.
3. Mango's oracle marked the perp position's unrealized PnL at this inflated price, crediting the long account with enormous "equity."
4. Borrow against that paper equity, withdrawing approximately **$110 million** of real assets (USDC, BTC, SOL, and more) — draining the protocol.

The position was never closed profitably; the "profit" was an oracle artifact. The protocol let unrealized gains on an illiquid asset back fully fungible withdrawals.

**Lesson:** never let the manipulable price of an illiquid asset collateralize claims on liquid assets. Especially not your own governance token.

---

## The anatomy, generalized

Strip away the specifics and every one of these is the same five steps:

1. **Acquire capital cheaply** (flash loan, or simply enough working capital for a thin market).
2. **Find the manipulable price input** the protocol trusts — DEX spot, pool state, or a thin-market mark.
3. **Move it** in the direction that inflates collateral / PnL or deflates debt.
4. **Extract** value while the oracle is wrong — borrow, mint, withdraw, settle.
5. **Restore and repay**, keeping the difference.

If your protocol reads a number that step 3 can change, you are exposed.

---

## Defenses that actually work

**Use manipulation-resistant oracles.**
- Aggregated, multi-source feeds (e.g. Chainlink-style) with **staleness checks** and **deviation bounds**. Reject prices that are too old or that jump beyond a sane band.
- **TWAPs** over a meaningful window raise the cost of manipulation from one block to many — but a TWAP is only as safe as the depth of the venue it averages. A TWAP of a thin pool is still cheap to push.

**Price LP and pool assets by fundamentals, not spot.**
- For LP-token or pool-derived collateral, compute **fair value from underlying reserves and external prices** (the "fair LP pricing" approach) instead of trusting instantaneous pool state. This kills the Harvest-style deposit/withdraw sandwich.

**Constrain the collateral itself.**
- Cap or refuse **illiquid and self-referential collateral** (your own token). Mango and Cheese Bank both die here.
- Set conservative LTVs and per-asset borrow caps so that even a successful manipulation can't drain more than a bounded amount.

**Add circuit breakers.**
- Deviation-triggered pauses, price-change rate limits, and oracle-disagreement halts turn a catastrophic drain into a contained, recoverable event.

**Separate paper gains from real withdrawals.**
- Don't let unrealized PnL on a volatile/illiquid mark back fungible withdrawals without haircuts and liquidity-aware limits.

**Defenses that *don't* work:** a single DEX spot price; a TWAP over a shallow pool; "no one would spend that much" assumptions (flash loans make capital free); and trusting that your token is "different."

---

## A 9-point checklist for teams

Before mainnet, confirm every "yes":

1. Does every price input come from a source an attacker **cannot move within a transaction**?
2. Are there **staleness** and **deviation** checks on every feed, with a safe fallback?
3. Is any **pool/LP value** computed from fundamentals rather than spot state?
4. Could a **flash loan** change any number your accounting reads? Test it explicitly.
5. Is **illiquid or self-referential collateral** capped, haircut, or rejected?
6. Are there **per-asset borrow caps** bounding worst-case loss?
7. Do you have **circuit breakers** on oracle deviation and price-change rate?
8. Can **unrealized PnL** on a thin asset back withdrawals of liquid assets? (It shouldn't.)
9. Have you **simulated the five-step attack** against your own contracts in a fork test?

If you can't answer all nine with confidence, the price your protocol trusts is not truth — it's an attack surface.

---

## Get a second set of eyes

We audit DeFi protocols for exactly this class of bug — oracle design, flash-loan attack surface, collateral and liquidation logic. If you're shipping anything that reads a price, **email [info@boogiemanmarketing.com](mailto:info@boogiemanmarketing.com) for a free 30-minute oracle-safety review.** We'll walk your price inputs against the checklist above and tell you straight where you're exposed.

Price is not truth. Make sure your contracts know the difference.

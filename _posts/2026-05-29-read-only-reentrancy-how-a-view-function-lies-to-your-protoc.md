---
layout: post
title: "Read-Only Reentrancy: How a View Function Lies to Your Protocol"
date: 2026-05-29 08:26:36 +0000
---

# Read-Only Reentrancy: How a View Function Lies to Your Protocol

Most engineers learn reentrancy as a withdrawal bug: an attacker re-enters your contract mid-call and drains it, like The DAO in 2016. So they add a `nonReentrant` modifier to every state-changing function, check the box, and move on.

Then their protocol gets drained anyway — through a function that has **no `nonReentrant` modifier because it changes no state at all.** It's a `view`. And during a reentrancy window in *another* contract, it returns a value that is temporarily, catastrophically wrong. Your protocol reads that value, believes it, and acts on it.

This is **read-only reentrancy**, and it has cost protocols millions precisely because it hides where defenders aren't looking.

---

## The setup: a view that depends on mid-flight state

Consider a Curve-style liquidity pool. Its `get_virtual_price()` view computes a price-per-LP-token from the pool's balances and total LP supply:

```
virtual_price ≈ total_pool_value / lp_token_supply
```

That formula is only correct when **balances and supply are consistent.** The danger is that some pool operations briefly make them inconsistent.

In several Curve pool versions, `remove_liquidity` does this:

1. Burns the user's LP tokens (supply drops), **and/or**
2. Sends the underlying assets to the user via a **raw external call** (for native ETH), **before** all internal accounting is finalized.

That external transfer hands control to the user's contract **while the pool's state is half-updated** — supply already reduced, balances not yet reflecting the final state (or vice versa). For the duration of that callback, `get_virtual_price()` returns a value computed from inconsistent numbers. It is wildly off.

The pool's *state-changing* functions are protected by a reentrancy lock. But `get_virtual_price()` is a harmless-looking `view` — no lock, no modifier. It will happily answer mid-reentrancy.

---

## The exploit: borrow against a lie

Now suppose a lending protocol accepts that pool's LP token as collateral and prices it with `get_virtual_price()`. The attack:

1. Attacker calls the pool's `remove_liquidity`, which begins updating state and then makes the external ETH callback to the attacker's contract.
2. **Inside that callback**, while the pool's view is returning a manipulated price, the attacker calls the lending protocol — deposits LP collateral and **borrows against the inflated valuation**, or triggers a liquidation, or mints shares at the wrong price.
3. The callback returns, `remove_liquidity` finishes, state becomes consistent again — but the attacker has already extracted value against a price that never really existed.

The attacker never called a single state-changing function on the *victim* protocol out of order. They never re-entered the victim at all in the classic sense. They re-entered the *pool*, and read a *view* that lied. The victim's `nonReentrant` modifiers were all present and all useless — they guarded the wrong thing.

Real protocols have lost funds to exactly this. The 2022 disclosures around Curve LP pricing flagged a wide swath of integrators as vulnerable, and subsequent exploits (e.g., the Conic Finance incident in mid-2023, on the order of a few million dollars) showed it was not theoretical. Any protocol pricing an LP/derivative token via a view that can be read mid-callback is a candidate.

---

## Why the usual reentrancy mental model fails here

The classic model says: "protect functions that move money." Read-only reentrancy breaks three assumptions baked into that model:

- **The vulnerable function changes no state**, so it gets no guard.
- **The reentrancy happens in a different contract** (the pool), not yours — your code looks "safe in isolation."
- **The damage is done by reading**, not writing. A pure data dependency on a manipulable view is enough.

If your security review only enumerates "functions that write state," you will miss this every time.

---

## The defenses that actually work

**1. Detect the locked state before trusting a price.**
The pool's state-changing functions are reentrancy-locked. The fix for integrators is to **refuse to read the price while that lock is engaged** — route your price read through a call that *reverts if the pool is mid-reentrancy*. Curve's own guidance is to invoke a reentrancy-lock-protected function (so the call reverts inside a reentrant context) before relying on `get_virtual_price()`. If you can't prove you're outside a reentrant call, don't trust the number.

**2. Don't price assets off instantaneous, callback-exposed views.**
Prefer **manipulation-resistant valuation**: compute LP/derivative fair value from underlying-asset prices and reserves using a method that doesn't transiently break during `remove_liquidity`, rather than reading a single mid-flight view. (This is the same lesson as classic oracle manipulation — instantaneous on-chain state is not a valuation.)

**3. Checks-effects-interactions, ruthlessly — including in views you depend on.**
The root cause is an external call made **before** state is fully consistent. Any contract you integrate with that transfers assets before finalizing accounting is a read-only-reentrancy factory. When you control the code, finalize *all* state before *any* external call, so no view can ever observe an inconsistent snapshot.

**4. Extend reentrancy reasoning to consumers, not just producers.**
A reentrancy guard should protect not only your fund-moving functions but the **integrity of any external view your logic reads**. Map every price/valuation input to the worst-case state it could return mid-callback.

---

## The audit checklist for read-only reentrancy

1. Does any pricing, valuation, or accounting logic read a **`view` from an external contract** (pool, vault, oracle adapter)?
2. Can that external contract make an **external call (especially native-asset transfer) before its state is fully consistent**? (Curve-style `remove_liquidity` is the canonical case.)
3. If so, can an attacker **re-enter during that window** and call your protocol while the view returns a manipulated value?
4. Do you **verify the external contract's reentrancy lock is not engaged** before trusting its price?
5. Are you pricing LP/derivative tokens by a **manipulation-resistant method** rather than a single instantaneous view?
6. In your own code, is **every external call made only after all state is finalized** (strict checks-effects-interactions)?
7. Does your review enumerate **read dependencies**, not just state-writing functions?

A "yes" on #1–#3 with a "no" on #4–#6 is an exploit waiting for its transaction.

---

## Get a second set of eyes

Read-only reentrancy is exactly the kind of bug a checklist-driven review misses and an attacker doesn't — it lives in the gap between your contract and the ones you integrate with. If your protocol prices LP tokens, integrates Curve/Balancer-style pools, or reads any external view to value collateral, **email [klashkaan@gmail.com](mailto:klashkaan@gmail.com) for a free 30-minute review.** We'll trace every external read your accounting depends on and tell you which ones can be made to lie.

The dangerous reentrancy isn't always the one that writes. Sometimes it's the one that just answers a question.

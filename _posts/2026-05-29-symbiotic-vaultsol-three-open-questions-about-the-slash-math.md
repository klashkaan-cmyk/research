---
layout: post
title: "Symbiotic Vault.sol: three open questions about the slash math"
date: 2026-05-29 19:48:15 +0000
---

`, `withdrawals[currentEpoch+1]`. Reading the code surfaced three questions worth answering before the next Cantina review cycle.

## 1. Why is the slash applied against `activeStake()` instead of `activeStakeAt(captureTimestamp)`?

```solidity
uint256 activeStake_ = activeStake();   // LATEST checkpoint, not at captureTimestamp
uint256 nextWithdrawals = withdrawals[currentEpoch_ + 1];
```

`captureTimestamp` is the moment the slashable event occurred. The vault validates that `captureEpoch ∈ {currentEpoch - 1, currentEpoch}`, so the gap between event and execution can span up to ~2 epoch durations. With a 1-week epoch, that's a 2-week window during which depositors can join and exit.

The slash burns from the *current* pool, not the pool at `captureTimestamp`. Concretely: if 100 ETH was actively staked at `captureTimestamp` and another 50 ETH deposited before `onSlash` fires, a slash of 10 ETH proportions across 150 ETH — diluting the late-arriving 50 ETH alongside the original 100.

That late-arrival pool wasn't staked at the slashable event. Should it bear loss? Two readings:

- **"Yes, by design"** — slashing is a property of the pool, not the depositor. Depositors accept this risk by entering a pool with a pending slash; they should observe slasher-emitted events and price accordingly.
- **"No"** — the vault should snapshot `activeStakeAt(captureTimestamp)` and burn proportionally from the historical accounting, leaving fresh deposits untouched.

The current code chooses (1). Symbiotic's docs should state this explicitly. If the answer is (2), then `activeStake_` should be replaced with `activeStakeAt(captureTimestamp, hint)`. The checkpoint infrastructure already exists; it's used in `activeBalanceOfAt`.

## 2. The fixup branch shifts loss asymmetrically. Is that intended?

```solidity
uint256 activeSlashed       = slashedAmount.mulDiv(activeStake_,     slashableStake);  // floor
uint256 nextWithdrawalsSlashed = slashedAmount.mulDiv(nextWithdrawals, slashableStake); // floor
uint256 withdrawalsSlashed  = slashedAmount - activeSlashed - nextWithdrawalsSlashed;   // residual

if (withdrawals_ < withdrawalsSlashed) {
    nextWithdrawalsSlashed += withdrawalsSlashed - withdrawals_;
    withdrawalsSlashed = withdrawals_;
}
```

Two `mulDiv` calls round down. The residual lands on `withdrawalsSlashed` — so the **current-epoch withdrawal queue** absorbs up to ~2 wei of double-floor error. Negligible at wei scale.

The more interesting bit is the fixup. When `withdrawals_` (current-epoch matured queue) is too small to cover its allocated share, the excess is pushed to **nextWithdrawals** (the deeper queue), never to active stake.

Walk the imbalance: `nextWithdrawalsSlashed_after = nextWithdrawalsSlashed_initial + (withdrawalsSlashed - withdrawals_)`. Substituting `withdrawalsSlashed = slashedAmount - activeSlashed - nextWithdrawalsSlashed_initial`:

```
nextWithdrawalsSlashed_after = slashedAmount - activeSlashed - withdrawals_
```

So when `withdrawals_` is small, queued exiters (nextWithdrawals) take a haircut that, proportionally, exceeds their fair share, while active stake is unchanged from the initial mulDiv. The design choice protects active stakers at the expense of users one epoch deeper in the withdrawal queue.

Is there a scenario where an actor controlling deposit + withdraw timing can engineer a small `withdrawals_` to shift loss onto other users' nextWithdrawals balance? Probably bounded by `slashedAmount` itself, but worth a formal property test: `nextWithdrawalsSlashed_after ≤ nextWithdrawals` should hold algebraically (it does, since `slashedAmount ≤ slashableStake` and `activeSlashed ≤ activeStake_`), but the proportional fairness invariant doesn't.

## 3. Two checkpoint pushes at the same block timestamp — last write wins

```solidity
_activeStake.push(Time.timestamp(), activeStake_ - activeSlashed);
```

OpenZeppelin's `Checkpoints.Trace256` overwrites the latest entry when pushed at the same key. So if `deposit()` and `onSlash()` execute in the same block, the second push wins. `upperLookupRecent(Time.timestamp())` returns the post-second-push value.

Most of the time this is irrelevant — both writers update activeStake monotonically within the block. But a slasher controlling block ordering (a validator, a builder taking the bundle) can pick whether the slash applies to the pre-deposit or post-deposit pool. The difference is whatever was deposited in the same block.

This is a small attack surface — slasher is already trusted, the manipulation is bounded by per-block deposit volume — but the property "slash result is independent of intra-block tx order" doesn't hold, and it should.

## Why I'm posting these as questions

I read the contracts cold. The verification I'd need before claiming any of these as bugs requires Slasher.sol, VetoSlasher.sol, and a property-test harness against `Checkpoints.Trace256` same-block semantics. If anyone reading this has run those checks — or if the design rationale is documented somewhere I missed — I'd appreciate a pointer.

If you're a protocol team thinking about your own slashing accounting: the three failure modes above (snapshot lag, asymmetric fixup, intra-block ordering) generalize. Pool-based loss accounting against a delayed event is a recurring source of unfair-loss bugs.

— K. Vance, BoogieMan Marketing
"""]

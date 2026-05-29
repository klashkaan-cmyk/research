---
layout: post
title: "Why Ajna's Scaled Fenwick Tree Is Rounding-Safe: A ≤13-Wei Dust Bound on LUP Search"
date: 2026-05-29 18:44:53 +0000
---

# Why Ajna's Scaled Fenwick Tree Is Rounding-Safe: A ≤13-Wei Dust Bound on LUP Search

Ajna's `Deposits.sol` is a binary-indexed (Fenwick) tree of bucket deposits with per-node `scaling` factors, used to find the LUP (lowest utilized price) in O(log n). `findIndexOfSum` walks the tree MSB→LSB, folding each scaled subtree sum into a running scale until the cumulative reaches `targetSum_`. Because each step composes `wmul` (half-up fixed-point multiply) and `floorWmul`, the natural question is: can rounding push the returned index off by a whole bucket — and thus shift the LUP?

The answer is no. The error is bounded at **≤ ~13 wei** of cumulative sum — never a bucket. Here is the line of reasoning.

## The three claims that matter

I focused on three properties. If all three hold, the search is sound:

1. The walk is a correct scaled binary search.
2. Rounding cannot move the crossing point by a whole bucket.
3. `Deposits.update` cannot resurrect a zero-valued node via independent rounding drift.

## (1) The walk is correct

Starting from the highest bit (i = 4096) down to i = 1:

- `runningScale` is folded **only** when the candidate step does **not** cross `targetSum_` — i.e. when the bit is cleared. That fold uses `floorWmul`, conservatively shrinking the carried scale.
- The range guard gates only the `sumIndex_` advance, not the scale fold.
- `sumIndex_` ends as the **largest index whose cumulative is strictly less than `targetSum_`**. The crossing bucket is therefore `sumIndex_ + 1` (the first index with cumulative ≥ target — ties stop, so the test is "reach or exceed", not strict).

The off-by-one return is the intended caller convention, not a bug. `sumIndexSum_` and `sumIndexScale_` are captured at the stop node so the caller can finish the accounting.

## (2) Rounding cannot move the crossing by a whole bucket

This is the key result.

`wmul` rounds half-up. `floorWmul` rounds down. At every evaluated node, the two differ by at most **1 unit** of fixed-point precision. The search visits at most **13 nodes** (`log2(4096) + 1`), so the worst-case cumulative discrepancy between the half-up walk and an exact-arithmetic walk is **≤ ~13 wei**.

Because `wmul` half-up overestimates, the only way rounding can flip the decision is when `targetSum_` lies within ≤13 wei of an exact bucket boundary. In that razor-thin window, the walk may cross one node early. The resulting accounting error is bounded at ≤13 wei — pure dust, not a whole bucket. Under any economic deposit size this is invisible.

This is a general pattern worth internalizing: **rounding in a scaled prefix-sum structure of depth d compounds as O(d), not O(bucket)**. A 4096-leaf Fenwick tree therefore tolerates rounding with a constant tens-of-wei dust envelope. The bucket itself never shifts.

## (3) `update` does not resurrect zero nodes

`Deposits.update` walks the tree applying a delta computed as `wmul(newValue, scaling) - wmul(value, scaling)`. Both terms share the **same** node `scaling`, so the two `wmul` calls round identically against the same multiplier — the subtraction telescopes to the **exact** change in this node's stored scaled value, with no residual rounding drift.

Consequence: a node currently storing `0` can become nonzero only through a real add (a deposit or move-in), never through accumulated rounding noise on unrelated operations. This closes off a class of "phantom liquidity" exploits that scaled accumulators are sometimes vulnerable to.

## Security takeaway

Three patterns to carry forward when reviewing scaled prefix-sum structures:

1. **Audit the rounding mode at each compose step.** Mixing half-up (`wmul`) and floor (`floorWmul`) is fine when the half-up direction is the conservative one for the comparison being made. Here `wmul` overestimates the prefix sum, which can only cause an early crossing — bounded by depth.
2. **Bound the cumulative error by depth, not by bucket.** If the depth is O(log n) and per-step error is O(1) unit, total error is O(log n) units. Verify nothing in the search amplifies it.
3. **For per-node updates, verify the rounding cancels.** When `delta = f(new) - f(old)` shares all rounding parameters between the two terms, the delta is exact even if `f` itself is lossy. Any divergence in those parameters reopens the resurrection vector.

Ajna's implementation gets all three right. No high-severity finding here — what's interesting is the technique itself: a 4096-leaf scaled Fenwick tree with a ≤13-wei rounding envelope is a solid building block.

---

*Research note. This analysis verified `findIndexOfSum` and the `update` delta of `Deposits.sol` against the three properties above. It does not extend to the rest of the library, nor to the protocol logic that consumes the LUP. If you maintain a scaled-accumulator library and want this depth of review, get in touch.*

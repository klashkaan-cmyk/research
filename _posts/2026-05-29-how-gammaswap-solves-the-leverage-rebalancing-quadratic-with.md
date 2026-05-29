---
layout: post
title: "How GammaSwap Solves the Leverage-Rebalancing Quadratic Without Overflowing"
date: 2026-05-29 18:23:34 +0000
---

# How GammaSwap Solves the Leverage-Rebalancing Quadratic Without Overflowing

*Part of an ongoing series reading the hard parts of GammaSwap's on-chain math. Source: [`gammaswap/v1-implementations`](https://github.com/gammaswap/v1-implementations), `contracts/libraries/cpmm/CPMMMath.sol` and `contracts/libraries/FullMath.sol`. Independently reviewed in-house.*

When a leveraged LP position needs to rebalance its collateral to a target ratio against a constant-product pool, the amount to trade isn't a simple swap — it's the root of a **quadratic** that accounts for the trade's own market impact and fees. GammaSwap's `CPMMMath` library solves `a·x² − b·x + c = 0` on-chain. That sounds routine until you notice the coefficients are products of pool reserves and collateral balances, and the discriminant `b² − 4ac` overflows 256 bits long before any realistic pool does.

This post walks through three things the library gets right, and one boundary you must respect if you build on it.

## 1. The phantom-overflow problem

Take the discriminant `b² − 4ac`. In `calcDeltasForMaxLP`, `b` is on the order of `reserve · reserve` (after fee scaling), so `b` can already approach `2^160`–`2^180` for a deep pool. Squaring it pushes `b²` past `2^256`. The product `4ac` is similarly a triple product of reserves and collateral. A naive `uint256` implementation doesn't just lose precision here — it **wraps**, and the square root of a wrapped value is a meaningless number that the protocol would then treat as a trade size.

This is the classic *phantom overflow*: the inputs and the final answer both fit in 256 bits, but an intermediate value does not.

## 2. The 512-bit discriminant, branched on demand

GammaSwap computes the entire discriminant in 512-bit space using a full-precision math library (`FullMath`, whose `mulDiv512`, `mul256x256`, `mul512x256`, and `sqrt512` primitives are the well-known Remco Bloemen / "mathemagic" / Karatsuba-sqrt reference implementations — all flooring, all reverting on true overflow).

The neat part is that it only pays for 512-bit work when it has to. Computing `c`:

```solidity
(c, det) = FullMath.mulDiv512(c, (reserve0 * fee1), (10**decimals0) * fee2);

if (det > 0) {
    det = calcDeterminant512(a, b, c, det, cIsNeg); // c overflowed 256 bits → full 512-bit path
} else {
    det = calcDeterminant(a, b, c, cIsNeg);         // c fits in 256 bits → cheaper path
}
```

`mulDiv512` returns the result as a `(low, high)` pair. The `high` word (`det` here, reused as a flag) is non-zero *exactly when* `c` itself exceeds 256 bits. The code reads that high word as a signal and routes to the 512-bit-aware discriminant routine. Both routines compute `b²` via `mul256x256` (→ 512 bits) and subtract/add `4ac`, then take a 512-bit square root that contracts back to a correct 256-bit value:

```solidity
// calcDeterminant (a > 0):
if (cIsNeg) {                          // c < 0  ⇒  −4ac > 0
    (l0,l1) = add512x512(b², 4a|c|);
    det = sqrt512(l0, l1);             // sqrt(b² + 4a|c|)
} else {
    if (lt512(b², 4ac)) revert ComplexNumber();   // imaginary root → refuse
    (l0,l1) = sub512x512(b², 4ac);
    det = sqrt512(l0, l1);             // sqrt(b² − 4ac)
}
```

No intermediate ever wraps, and a genuinely imaginary discriminant reverts with `ComplexNumber()` rather than returning garbage.

## 3. Unsigned sign-tracking

The 512-bit primitives are unsigned-only — there is no `int512`. So the library carries the sign of `c` in a separate boolean, `cIsNeg`, computed when `c` is first formed:

```solidity
c = tokensHeld1 * reserve0;
uint256 rightVal = tokensHeld0 * reserve1;
(cIsNeg, c) = c > rightVal ? (false, c - rightVal) : (true, rightVal - c);
```

`c` is always stored as a magnitude; `cIsNeg` remembers the sign. The discriminant branch above then becomes pure case analysis: `c < 0` flips `b² − 4ac` into `b² + 4a|c|` (which can never be imaginary), while `c ≥ 0` requires `b² ≥ 4ac` or reverts. The same discipline applies to `b`, which is stored as a positive magnitude even though the true coefficient is negative.

## 4. Root selection — and the one boundary

With `a > 0`, `b ≥ 0` (stored magnitude), `det ≥ 0`, the roots are `x = (b ± det)/(2a)`. The library returns:

```solidity
deltas[0] = (b + det) / (2a);                 // larger root
deltas[1] = (b > det) ?  (b - det)/(2a)        // smaller root, ≥ 0
                      : -(det - b)/(2a);        // smaller root, < 0
```

I verified this selection by hand and via independent in-house review:

- **`deltas[0] = (b + det)/(2a)` is non-negative unconditionally** — sum of non-negatives over a positive denominator. It is the feasible root the interface documents as "index 0," in every input case.
- **The sign branch for `deltas[1]` is correct in all cases.** When `c ≥ 0`, the `require(b² ≥ 4ac)` guarantees `det ≤ b`, so the smaller root is `≥ 0`. When `c < 0`, `det = √(b² + 4a|c|) > b`, so the else-branch correctly returns a negative value. The roots have opposite signs precisely when `c < 0` (product of roots `= c/a < 0`), and index-0 is then the unique positive root.
- **The `b == det` edge is handled:** `b > det` is false, so it returns `−0/(2a) = 0`, matching a true root of exactly zero.

One honest nuance: because `det` is floored, a smaller root whose true value is a tiny negative (when `b² < b²+4a|c| < (b+1)²`) gets reported as exactly `0`. That is a magnitude-rounds-toward-zero artifact — **not** a sign error. The assigned sign is always consistent with the *computed* `det`, and flooring can never invert it.

**The boundary that matters for integrators:** this function guarantees the returned delta is the *mathematically correct positive root* — it does **not** guarantee the trade is *economically feasible*. Nothing here bounds the delta against the reserves actually available or the collateral actually held. That check lives one layer up, in the strategy contract. You can see the pattern in the sibling function `calcCollateralPostTrade`:

```solidity
uint256 soldToken = reserve1 * delta * fee2 / ((reserve0 - delta) * fee1) + 1;
require(soldToken <= tokensHeld1, "SOLD_TOKEN_GT_TOKENS_HELD1");
```

Two defensive touches there reinforce the design philosophy: the `+ 1` rounds the amount sold *up* (the protocol never under-charges itself), and the `require` rejects any delta the collateral can't cover. So even if upstream math produced an aggressive root, the position-level invariant catches it — the failure mode degrades to a **revert**, not a loss of funds.

## Takeaway

The security story here isn't a single clever line — it's layering. The math library's job is to be *arithmetically* bulletproof (no overflow, correct root, refuse imaginary results), and it is. The job of being *economically* safe — that a recommended trade fits the pool and the collateral — is deliberately pushed to the caller's invariants. That separation is exactly what you want to see in audited DeFi math: each layer has one guarantee it never violates, and feasibility is enforced where the balances actually live.

## Scope and what's next

I read and verified `calcDeltasForMaxLP`, the four determinant helpers, and the `FullMath` multiply/divide primitives directly from source. I did **not** re-derive the internals of the Karatsuba `sqrt512` (a well-known INRIA reference algorithm) line-by-line — that, and the deliberately *different* root selection in `calcDeltasForWithdrawal` (it documents index **1**, not index 0, as the feasible root — an asymmetry worth its own writeup), are next in this series.

*If you're shipping leveraged AMM math and want the discriminant, rounding directions, and root selection checked the way this post checks them, that's the kind of work we do.*

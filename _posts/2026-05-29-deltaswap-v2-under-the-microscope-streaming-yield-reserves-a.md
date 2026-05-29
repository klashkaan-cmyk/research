---
layout: post
title: "DeltaSwap V2 under the microscope: streaming-yield reserves are solvent-by-construction, and where the anti-MEV fee leaks"
date: 2026-05-29 18:14:28 +0000
---

*A static review of two custom mechanisms in GammaSwap's DeltaSwap V2 AMM. Source: [github.com/gammaswap/v2-deltaswap](https://github.com/gammaswap/v2-deltaswap), `DeltaSwapV2Pair.sol` + `DSMath.sol`, Solidity 0.8.21. Public code, no engagement — published as audit research.*

## TL;DR

DeltaSwap V2 is a Uniswap-V2-style `x*y=k` pair with two non-standard additions: (1) it **streams** realized fee growth back into the reported reserves over a configurable window instead of crediting it instantly, and (2) it charges a swap fee **only** when a trade is large relative to recent pool activity — a built-in anti-MEV / anti-arb toll rather than a flat 0.30%.

I reviewed both mechanisms line-by-line. Findings:

- **The streaming-reserve math is solvent-by-construction.** Reported reserves can never claim more backing than the pool actually holds — I prove the inequality below. No over-reporting, even transiently. The much-discussed `+1` rounding term is bounded and LP-favorable.
- **The activity-gated fee is best-effort, not a guarantee.** Three evasion paths exist, all inherent to per-trade dynamic fees: liquidity-EMA inflation, cross-block slicing, and a full EMA reset after long inactivity. None lose user funds; they erode *fee revenue / MEV protection*, mostly on low-activity pools. Severity LOW–MEDIUM, design-level.

No high-severity, fund-loss bug in the reviewed surface. The `swap()`, `mint()`, and `burn()` bodies were out of scope, so anything depending on K-invariant enforcement is flagged as an assumption, not a claim.

---

## Mechanism 1 — Streaming-yield reserves

Vanilla Uniswap V2 credits swap fees to LPs instantly: every swap nudges `k` upward, and that value is immediately reflected in the reserves an LP can redeem. DeltaSwap instead tracks two roots of `k`:

- `rootK1 = sqrt(balance0 · balance1)` — the *true* current invariant from real token balances.
- `rootK0` — a *lagging* invariant that climbs toward `rootK1` linearly over `yieldPeriod`.

`getLPReserves()` reports reserves scaled by the lagging ratio:

```solidity
uint256 _rootK1 = DSMath.sqrt(uint256(_reserve0)*_reserve1);
if (timeElapsed > 0 && _rootK0 > 0 && _rootK1 > _rootK0) {
    timeElapsed = uint32(DSMath.min(timeElapsed, _yieldPeriod));
    _rootK0 = _rootK0 + (_rootK1 - _rootK0) * timeElapsed / _yieldPeriod;
}
_reserve0 = uint112(_reserve0 * _rootK0 / _rootK1);
_reserve1 = uint112(_reserve1 * _rootK0 / _rootK1);
```

The point is to smooth yield: a burst of fees raises `rootK1` immediately but only feeds into redeemable reserves gradually, so a just-in-time LP can't deposit, capture a fat fee block, and withdraw the same minute.

### Is it solvent? Can reported reserves ever exceed real backing?

This is the question that matters. If `getLPReserves()` could ever report a product greater than `balance0 · balance1`, the pool would be promising liquidity it doesn't hold. It cannot. Here is the chain, using the state written in `_update`:

```solidity
s.rootK0 = uint112(DSMath.min(DSMath.sqrt(uint256(_reserve0)*_reserve1) + 1, _rootK1)); // _rootK1 = sqrt(balance0*balance1)
s.reserve0 = uint112(balance0);
s.reserve1 = uint112(balance1);
```

1. At write time `s.rootK0 = min(…, rootK1)`, so **`rootK0 ≤ rootK1`** where `rootK1 = sqrt(balance0·balance1)`.
2. Balances (`s.reserve0`, `s.reserve1`) are written in the same call, so the next `getLPReserves()` recomputes `rootK1' = sqrt(s.reserve0·s.reserve1) = sqrt(balance0·balance1)` — identical until the next `_update`.
3. The streaming step only adds `(rootK1' − rootK0)·t/period ≤ (rootK1' − rootK0)`, so the streamed `rootK0` is still **`≤ rootK1'`**.
4. Therefore each reported reserve is `s.reserveᵢ · rootK0/rootK1' ≤ s.reserveᵢ`, and two floor-divisions only push it lower. **The product of reported reserves ≤ `balance0·balance1` — the real backing.** ∎

So the design is conservative in the safe direction: it can under-report (yield not yet streamed) but never over-report.

### The `+1` term

```solidity
DSMath.sqrt(...) + 1   // "+1 because square root formula rounds down"
```

`DSMath.sqrt` is a floor integer square root (verified: Babylonian with the standard `z := sub(z, lt(div(a,z), z))` floor correction). The `+1` compensates that floor so `rootK0` doesn't permanently lag by one unit each update. Its worst case is recognizing **one unit of `rootK` early**, and it is still hard-clamped by `min(…, rootK1)`. Relative to `k` that error is `~2/rootK1`:

| Pool size (balanced, each reserve ≈ R) | rootK1 ≈ R | Max relative reserve distortion from `+1` |
|---|---|---|
| 1e18 wei (1 token, 18-dec) | 1e18 | ~1e-18 |
| 1e12 wei | 1e12 | ~1e-12 |
| 1e6 wei (dust pool) | 1e6 | ~1e-6 |

Even for a dust-sized pool the distortion is sub-ppm and LP-favorable. Not exploitable.

---

## Mechanism 2 — The activity-gated swap fee

Instead of a flat fee, DeltaSwap charges `dsFee` **only** when a trade is large relative to a rolling measure of pool liquidity:

```solidity
function calcTradingFee(uint256 tradeLiquidity, uint256 lastLiquidityTradedEMA, uint256 lastLiquidityEMA) public view returns (uint256) {
    if (s.dsFee > 0 && DSMath.max(tradeLiquidity, lastLiquidityTradedEMA) >= lastLiquidityEMA * s.dsFeeThreshold / 1e8) {
        return s.dsFee;
    }
    return 0;
}
```

Two exponential moving averages feed this:

- **`liquidityEMA`** — EMA of `sqrt(balance0·balance1)`, updated once per block in `_update` with weight `max(blockDiff, 10)` (capped at 100 inside `calcEMA`). This sets the *bar*: fee triggers when a trade's liquidity meets `liquidityEMA · dsFeeThreshold / 1e8`.
- **`tradeLiquidityEMA`** — EMA of recent trade sizes (weight 20), with two reset rules:

```solidity
function _getLastTradeLiquiditySum(uint256 tradeLiquidity, uint256 blockDiff) internal view returns (uint256) {
    if (blockDiff > 0) { return tradeLiquidity; }          // no carry across blocks
    return s.lastTradeLiquiditySum + tradeLiquidity;       // same-block trades sum
}
function _getLastTradeLiquidityEMA(uint256 blockDiff) internal view returns (uint256) {
    if (blockDiff > 50) { return 0; }                       // reset after ~10 min idle
    return s.tradeLiquidityEMA;
}
```

The intent is elegant: small retail swaps trade fee-free, while large swaps — the ones that move price and attract arbitrage/MEV — pay the toll. Same-block slicing is explicitly defeated, because trades within one block are *summed* before the threshold test.

But "fee only on large-vs-recent-activity trades" is a softer guarantee than it looks. Three observations, all design-level:

### 2a. Liquidity-EMA inflation (LOW–MEDIUM, low-activity pools)

The fee bar scales with `liquidityEMA`. Raise the EMA and large trades slip under the bar fee-free. `liquidityEMA` is most malleable exactly when a pool is quiet: with `blockDiff ≥ 100`, `calcEMA`'s weight caps at 100, meaning **the EMA snaps fully to the current `sqrt(balance0·balance1)`** on the next update. An actor can, on a dormant pool: add large liquidity (an update that resets the EMA to the inflated level), execute the otherwise-fee-bearing swap under the now-raised bar, then withdraw. Cost = capital + add/remove friction + block-timing control. On an active pool this is impractical (each block only moves the EMA ≤10% of the gap), so the exposure is concentrated in thin pools.

*Assumption to confirm against `swap()`:* that the fee is evaluated against `s.liquidityEMA` as left by prior/same-tx updates. The public interface strongly implies this; I did not read `swap()`.

### 2b. Cross-block slicing (LOW)

`_getLastTradeLiquiditySum` carries **nothing** across blocks (`blockDiff > 0 ⇒ return tradeLiquidity`), and `tradeLiquidityEMA` carries only 20% weight and zeroes after 50 idle blocks. A whale can split one large position into sub-threshold chunks spread across blocks and each chunk is judged on its own size. Same-block slicing is blocked; cross-block slicing is not. The cost is real, though — multi-block execution means price-impact drift and MEV exposure between chunks — so this is a "pay-with-risk-instead-of-fee" trade, not free money.

### 2c. Post-idle EMA reset window (LOW)

The `blockDiff > 100 ⇒ weight 100` (full reset) and `blockDiff > 50 ⇒ tradeEMA = 0` rules create a predictable window right after low-activity periods where both EMAs are freshest/lowest and most steerable. This is what makes 2a cheap rather than a separate bug, but it's worth the team's attention as the common root: **the anti-MEV fee is weakest precisely on the pools that are quiet enough to be cornered.**

---

## Verdict

| Item | Severity | Notes |
|---|---|---|
| Streaming-reserve over-reporting | None | Proven solvent-by-construction; clamped by `min(…, rootK1)`. |
| `+1` sqrt-rounding term | None | Bounded `~2/rootK1`, LP-favorable. |
| liquidity-EMA inflation to dodge fee | Low–Med | Practical only on low-activity pools; erodes fee/MEV protection, no fund loss. |
| Cross-block trade slicing | Low | Inherent to dynamic fees; costs execution risk. |
| Post-idle EMA reset window | Low | Root cause shared with 2a. |

**No high-severity issue in the reviewed surface.** The streaming-reserve mechanism is a genuinely nice piece of engineering — it solves JIT-LP yield-sniping without ever risking pool solvency. The activity-gated fee is a reasonable anti-MEV heuristic, but it should be understood as *best-effort*: a sufficiently patient or well-capitalized actor can route large flow fee-free on thin pools. If protecting fee revenue on low-liquidity pairs matters, the fix space is small (floor the threshold, carry a decaying trade-sum across blocks, or rate-limit EMA movement) — happy to detail.

*Scope: static review of `DeltaSwapV2Pair.sol` (streaming-reserve + fee functions) and `DSMath.sol` only. `swap()/mint()/burn()` invariant enforcement was not reviewed; claims touching them are labeled as assumptions. This is unsolicited research on public code, not a paid engagement and not a disclosure of an exploited issue.*

---

*We do 24/7 deep smart-contract review — math-heavy AMMs, vaults, and lending especially. If you want this depth on your code before it ships, get in touch.*

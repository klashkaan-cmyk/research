---
layout: post
title: "GammaSwap's ERC-4626 Vault: How It Closes the Inflation Attack — and the One Edge That Remains"
date: 2026-05-29 17:52:17 +0000
---

# GammaSwap's ERC-4626 Vault: How It Closes the Inflation Attack — and the One Edge That Remains

ERC-4626 tokenized vaults have a famous failure mode: the first-depositor "inflation" (or "donation") attack. An attacker deposits 1 wei to mint a single share, then transfers a large amount of the underlying asset directly to the vault. Because naive vaults compute share price from the vault's live token balance, that donation inflates the attacker's one share; the next honest depositor's `assets * supply / totalAssets` rounds down to zero shares — and the attacker, holding 100% of supply, owns the victim's deposit.

GammaSwap's `GammaPoolERC4626` — where the *asset* is a CFMM LP token and the *share* is a GS LP token — defends against this with two independent mechanisms. Here's how each works, why the combination is robust, and the single accounting edge that survives.

## Defense 1: Internal accounting, not `balanceOf`

The crux of the donation attack is that `totalAssets` reads the vault's live token balance, so an unsolicited transfer moves the price. GammaSwap doesn't do that. `totalAssets()` resolves through `_totalAssetsAndSupply()`, documented as "calculated using state global variables" — the pool's stored LP balance and borrowed invariant, updated only inside `deposit`/`withdraw`/`mint`/`redeem`.

A raw ERC-20 `transfer` of CFMM LP tokens to the pool therefore changes nothing the pricing math can see. The donation lever — the entire premise of the attack — is disconnected. This alone neutralizes the canonical exploit.

## Defense 2: MIN_SHARES under-mint on first deposit

GammaSwap adds defense-in-depth for the genuinely-first deposit:

```solidity
uint256 public constant MIN_SHARES = 1e3;
...
if (supply == 0 || _totalAssets == 0) {
    if (assets <= MIN_SHARES) revert MinShares();
    unchecked { return assets - MIN_SHARES; }
}
```

The first depositor must supply more than 1,000 units and receives `assets - 1000` shares. Unlike Uniswap V2, which mints `MINIMUM_LIQUIDITY` to a burn address, GammaSwap simply *under-mints*: 1,000 units of assets back zero shares — a permanent floor. The subtraction is `unchecked`, but the preceding `revert MinShares()` guarantees `assets > MIN_SHARES`, so it cannot underflow.

## The detail forks get wrong: rounding direction

The most common subtle bug in ERC-4626 implementations is rounding the wrong way — toward the user instead of the vault — which leaks value one wei at a time and, in adversarial loops, much faster. GammaSwap's four preview paths all favor the vault:

| Function        | Calls                          | Rounds |
|-----------------|--------------------------------|--------|
| previewDeposit  | _convertToShares(assets,false) | down   |
| previewMint     | _convertToAssets(shares,true)  | up     |
| previewWithdraw | _convertToShares(assets,true)  | up     |
| previewRedeem   | _convertToAssets(shares,false) | down   |

Depositing rounds shares *down*; minting rounds asset cost *up*; withdrawing rounds shares burned *up*; redeeming rounds assets returned *down*. In every case the residual dust accrues to existing LPs — never to the actor. The ceil-division is the standard `(a * b + (denom - 1)) / denom`. Correct, and correctly applied.

## The edge that remains: `totalAssets == 0` with live supply

There is one sharp edge worth flagging. The first-deposit branch triggers on `supply == 0 || _totalAssets == 0`. The second disjunct means: if the pool's accounted assets ever reach exactly zero while GS LP shares are still outstanding, the *next* depositor is priced as if they were the first — receiving `assets - MIN_SHARES` shares regardless of existing supply.

The consequence: that depositor's real assets now back not only their freshly-minted shares but also the pre-existing shares, worthless a moment earlier. The old holders are silently re-capitalized at the new depositor's expense:

```
new fraction = (assets - 1000) / (oldSupply + assets - 1000)  <  (assets - 1000) / assets
```

In practice this is low severity. Reaching `totalAssets == 0` with non-zero supply requires the pool's entire LP balance *and* borrowed invariant to be wiped while shares persist — a near-insolvency state the MIN_SHARES buffer and proportional withdrawal make hard to enter. But "hard to reach" is not "unreachable," and the failure is silent. A defensive implementation would revert in this state, or socialize the re-initialization explicitly rather than implicitly favoring stale shares.

## Takeaway

GammaSwap's ERC-4626 layer does the boring things right: price from internal accounting so donations are inert, keep a first-deposit buffer for defense-in-depth, and round every conversion toward the pool. The inflation attack is closed not by one clever trick but by removing the attacker's lever entirely. The residual `totalAssets == 0` edge is the kind of thing worth a one-line guard — informational here, but exactly where the next bug in a *fork* of this code will live.

*Part of an ongoing series auditing live DeFi math. If you're shipping a CFMM, lending vault, or ERC-4626 derivative and want this depth of review before mainnet, get in touch.*

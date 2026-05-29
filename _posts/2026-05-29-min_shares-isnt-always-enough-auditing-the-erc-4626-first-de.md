---
layout: post
title: "MIN_SHARES Isn't Always Enough: Auditing the ERC-4626 First-Deposit Attack"
date: 2026-05-29 17:19:59 +0000
---

The first-deposit inflation attack on ERC-4626 vaults is old news — and still ships to production. Reviewing live vault code, the interesting question is no longer "is it vulnerable?" but "exactly how much does each mitigation buy you?" Here's the auditor's version.

## The attack, precisely

An empty vault mints shares as `shares = assets * totalSupply / totalAssets`. The attacker is the first depositor:

1. Deposit 1 wei → receive 1 share. `totalSupply = 1`.
2. Donate a large amount `D` of the underlying directly to the vault. Now `totalAssets ≈ D`, `totalSupply = 1`, so one share is worth `D`.
3. The next honest depositor adds `a < D` and receives `a * 1 / D = 0` shares (rounded down). Their deposit is absorbed; the attacker redeems their single share for the inflated pool.

Two distinct properties have to hold to kill this — and auditors routinely check only the first.

## Mitigation 1: a minimum-shares floor (necessary, not sufficient)

Locking a minimum number of shares on the first deposit — e.g. minting `MIN_SHARES` to `address(0)` and crediting the first depositor `assets - MIN_SHARES` — prevents the "1 share for 1 wei" setup. Done correctly the lock is *permanent* (burned to a dead address), not merely held by the contract.

But the floor only bounds the share price the attacker can manufacture from their *own* deposit. With a linear first-deposit (`shares = assets - MIN_SHARES`), depositing just above the floor yields ~1 share at a price of ~`MIN_SHARES`. Honest deposits *below that price* still round to zero. The residual loss is bounded to ~`MIN_SHARES` units — negligible for an 18-decimal LP token, potentially not for a 6-decimal stablecoin with a small floor. **Check the floor's magnitude against the asset's decimals.**

## Mitigation 2: donation resistance (the one that's often missing)

The lethal step is the *donation* — inflating `totalAssets` without minting shares. It only works if `totalAssets()` reads the token's live `balanceOf(vault)`. Vaults that track an **internal accounting variable** (updated only on deposit/withdraw, ignoring direct transfers) are immune to donation regardless of the floor: a donated token simply isn't counted.

So the real audit checklist for a custom 4626 is three lines:
1. Is a minimum-liquidity amount **permanently locked** on first deposit (to a dead address, not the contract)?
2. Is its magnitude meaningful relative to the asset's decimals?
3. Does `totalAssets()` use **internal accounting**, not `balanceOf` — closing the donation vector entirely?

A vault that does (1) and (3) reduces the entire attack class to bounded dust griefing. A vault that does (1) but reads `balanceOf` is still fully exploitable — the floor just raises the attacker's setup cost. The mitigations are not interchangeable, and "we lock minimum shares" is not, by itself, an answer.

---

*Khalid Vance Research audits Solidity/EVM contracts — vault share math, accounting invariants, oracle and reentrancy surfaces. Free 30-minute review; engagements in crypto. klashkaan@gmail.com.*

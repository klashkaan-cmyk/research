---
layout: post
title: "The First-Depositor Inflation Attack: How a 1-Wei Deposit Steals an ERC-4626 Vault"
date: 2026-05-29 08:23:17 +0000
---

# The First-Depositor Inflation Attack: How a 1-Wei Deposit Steals an ERC-4626 Vault

There is a vulnerability that shows up in a startling fraction of tokenized vaults we review. It needs no flash loan, no governance takeover, and no exotic math. It needs an attacker to be **the first person to deposit** — and a victim to deposit after them. The attacker walks away with the victim's money. The vault contract behaves exactly as written. That is the problem.

This is the **inflation attack** (a.k.a. the first-depositor or share-donation attack), and it is endemic to naïve ERC-4626 implementations. Here is exactly how it works, with numbers and code, and the fixes that actually close it.

---

## Background: how share vaults price deposits

An ERC-4626 vault is a tokenized share of a pool of assets. When you deposit, you mint shares:

```
shares = assets * totalSupply / totalAssets
```

and when you withdraw, you burn shares for a proportional slice of `totalAssets`. To protect the vault from rounding in the user's favor, **deposits round shares *down*.** That single, correct-looking rounding rule is the lever the attack pulls.

The second ingredient: many implementations compute `totalAssets()` as the vault's live token balance:

```solidity
function totalAssets() public view returns (uint256) {
    return asset.balanceOf(address(this)); // <-- the flaw
}
```

If `totalAssets` is just `balanceOf(this)`, then **anyone can increase it by sending tokens directly to the vault** — a "donation" that mints no shares but raises the value of every existing share.

Combine "deposits round down" with "share price can be donated upward" and you get theft.

---

## The attack, step by step

Assume a fresh vault, asset has 18 decimals, victim is about to deposit **2,000 tokens**.

1. **Attacker deposits 1 wei.** Vault is empty, so they get `1` share. Now `totalSupply = 1`, `totalAssets = 1 wei`. The attacker owns 100% of the vault.

2. **Attacker donates 2,000 tokens** by transferring them straight to the vault address. No shares minted. Now `totalSupply = 1`, `totalAssets = 2000e18 + 1`. **One share is now worth ~2,000 tokens.**

3. **Victim deposits 2,000 tokens.** Their shares:
   ```
   shares = 2000e18 * 1 / (2000e18 + 1) = 0   (rounds down)
   ```
   The victim receives **zero shares** for a 2,000-token deposit. `totalAssets` is now ~4,000 tokens, `totalSupply` is still `1`, all owned by the attacker.

4. **Attacker redeems their 1 share** for the entire ~4,000 tokens — their 2,000 donation back, plus the victim's 2,000.

The victim paid 2,000 tokens for nothing. No function reverted. No invariant was "violated." The vault did precisely what its arithmetic prescribed.

In practice attackers tune the donation so the victim's shares round down to zero (or near-zero), and they can do the whole thing in one bundled transaction by front-running the victim's deposit in the mempool.

---

## Why "it rounds down" isn't a good enough defense

Two common rebuttals, both wrong:

- *"The victim should just check slippage."* True — and a `minSharesOut` check would save them. But ERC-4626's `deposit()` has no slippage parameter, and most integrators call it raw. A safe vault cannot assume every caller wraps it correctly.
- *"Donations are weird, no one does that."* Donations are a one-line `transfer`. The entire exploit costs the attacker nothing but the (recoverable) donation and gas.

The vulnerability lives in the vault, so the fix belongs in the vault.

---

## The fixes that actually work

**1. Virtual shares and assets (the canonical fix).**
OpenZeppelin's ERC-4626 (v4.9+) adds a *decimals offset* that introduces virtual shares and one virtual asset into the conversion:

```
shares = assets * (totalSupply + 10**offset) / (totalAssets + 1)
```

The virtual offset means the attacker can never own ~100% of effective supply with 1 wei, and the donation needed to round a realistic victim deposit to zero becomes economically absurd (it scales with `10**offset`). This is the recommended baseline — use the audited OZ implementation and set a sensible offset rather than rolling your own.

**2. Internal accounting instead of `balanceOf`.**
Track deposited principal in a storage variable and ignore unsolicited transfers:

```solidity
uint256 internal _totalAssets;
function totalAssets() public view returns (uint256) { return _totalAssets; }
// _totalAssets += amount on deposit; -= amount on withdraw
```

A direct donation no longer changes share price, so step 2 of the attack does nothing. (Caveat: this changes how externally-accrued yield is recognized — design accordingly.)

**3. Dead shares / bootstrap deposit.**
Mint a non-trivial chunk of initial shares to a burn address (or have the deployer seed the vault) so the first real depositor is never minting against an empty, manipulable supply. This is what "seed the vault on deploy" guidance is really protecting against.

**4. Revert on zero-share mints and enforce minimums.**
At minimum: `require(shares > 0)` on deposit, and expose a `minSharesOut` slippage guard so integrators can protect themselves. This doesn't fix the root cause alone, but it turns a silent theft into a revert.

The strongest posture combines (1) or (2) with (4). Dead shares (3) alone can be overwhelmed by a large enough donation — size it deliberately.

---

## The audit checklist for any tokenized vault

1. Does `totalAssets()` read `balanceOf(this)`? If so, can a direct transfer move the share price? **(donation surface)**
2. Is the vault protected against the empty-vault first-depositor case — virtual shares, dead shares, or a seeded deposit?
3. Can a deposit mint **zero** shares? Is that path reverted?
4. Is there a slippage / `minSharesOut` guard for integrators?
5. Which direction does each conversion round, and does every rounding favor the vault (never the user)?
6. Is the OZ ERC-4626 base used as-is, or has the conversion math been re-implemented? (Re-implementations are where this bug reappears.)
7. For yield-bearing vaults: how is externally-accrued yield recognized without reopening the donation surface?

If you can't answer #1 and #2 cleanly, your vault is exploitable by its very first user.

---

## Get a second set of eyes

We audit vaults, lending markets, and any contract that does share-based accounting — the exact place this class of bug hides. If you're shipping an ERC-4626 vault or anything that mints proportional shares, **email [klashkaan@gmail.com](mailto:klashkaan@gmail.com) for a free 30-minute review.** We'll trace your deposit/withdraw math and rounding against the checklist above and show you where a first depositor could walk off with the pool.

The first deposit into your vault should be a milestone — not an exploit.

---
layout: post
title: "The Delegatecall Trap: Uninitialized Proxies, Dead Libraries, and $150M Frozen Forever"
date: 2026-05-29 08:30:41 +0000
---

# The Delegatecall Trap: Uninitialized Proxies, Dead Libraries, and $150M Frozen Forever

Upgradeable contracts are the standard way to ship serious protocols — you deploy a proxy that holds the state and money, and point it at a logic contract you can swap later. It's elegant. It's also the source of some of the most expensive mistakes in the industry, because the mechanism underneath it — `delegatecall` — is a loaded gun pointed at your own storage.

One unprotected initializer froze over **$150 million** of ETH *permanently*. One uninitialized implementation nearly bricked a major bridge and earned the whitehat who found it a **$10 million** bounty — the largest ever paid at the time. Both are the same bug. Here it is.

---

## How proxies actually work

A proxy contract holds the state and the funds. When you call it, its fallback does roughly:

```solidity
// inside the proxy
(bool ok, ) = implementation.delegatecall(msg.data);
```

`delegatecall` runs the **implementation's code** in the **proxy's storage context**, with the proxy's `address(this)` and balance. The logic lives in one contract; the state lives in another. That separation is what makes upgrades possible — and it's exactly what makes the failure modes below catastrophic. The implementation's code can touch the proxy's storage and the proxy's money, and a few specific mistakes let the *wrong person* do the touching.

---

## Pitfall 1: The unprotected initializer (and the uninitialized implementation)

Upgradeable contracts can't use a constructor to set state (constructors run in the implementation's context, not the proxy's), so they use an `initialize()` function instead. Two things must be true: it must run **once**, and it must run **on the proxy**. When neither is enforced, you get the two most famous freezes in Ethereum history.

**Parity multisig freeze (November 2017, ~$150M+ frozen forever).** Parity's multisig wallets were thin proxies that `delegatecall`-ed into a shared library contract. That library had an `initWallet` function with no protection — and the **library itself had never been initialized.** A user (`devops199`) called `initWallet` directly *on the library*, became its owner, and then called its `kill()` — a `selfdestruct`. Because every multisig wallet delegatecalled into that library for its logic, destroying the library **bricked all of them at once.** Roughly 513,000 ETH became permanently inaccessible. Not stolen — frozen. Nobody can ever move it.

**Wormhole uninitialized implementation (September 2022, $10M whitehat bounty).** Wormhole's bridge used a UUPS proxy. The **implementation contract was left uninitialized**, which meant anyone could call `initialize` on the implementation directly, take ownership of it, and — because the implementation retained an upgrade/`selfdestruct` path — destroy it. Destroying the logic contract a proxy points at bricks the proxy. A whitehat reported it through Immunefi before it was exploited and was paid **$10 million**. Had an attacker found it first, the bridge would have been frozen.

**Root cause in both:** an initializer that wasn't locked, on an implementation/library that was never initialized, combined with a reachable `selfdestruct`. The fix is mechanical (below) — which is exactly why it's so painful that it keeps happening.

---

## Pitfall 2: `selfdestruct` reachable through delegatecall

A logic contract should be inert on its own — it's just code other contracts borrow. But if the logic contract contains a `selfdestruct` (or a `delegatecall` to attacker-controlled code that can self-destruct it), and that path is reachable, then **destroying the shared logic destroys every proxy that depends on it.** This is the mechanism that turned both incidents above from "embarrassing" into "irreversible." Logic contracts must never carry a live self-destruct path.

---

## Pitfall 3: Storage layout collisions across upgrades

Because the implementation reads and writes the **proxy's** storage by slot number, the new implementation's storage layout must be a strict, compatible extension of the old one. Reorder variables, change types, or insert a field in the middle, and an upgrade will silently reinterpret existing slots — an `owner` address becomes a balance, a balance becomes a flag. No revert, no warning, just corrupted state and, often, lost funds. Upgrades are a storage-compatibility problem first and a code problem second.

---

## Pitfall 4: Function selector clashes

In some proxy designs, an admin function on the proxy can share a 4-byte selector with a function on the implementation, so a call meant for one hits the other. The transparent-proxy and UUPS patterns exist specifically to resolve this — which is why you should use the audited standard, not a hand-rolled proxy.

---

## The defenses that actually work

**1. Lock the implementation's initializer.**
Use OpenZeppelin's `Initializable` and call `_disableInitializers()` in the implementation's constructor so the **implementation can never be initialized directly** — only the proxy can be initialized, exactly once. This single line is what would have prevented the Wormhole bug.

```solidity
constructor() { _disableInitializers(); }
```

Use the `initializer` / `reinitializer` modifiers on init functions; never leave an init function callable twice or on the logic contract.

**2. Keep `selfdestruct` out of logic contracts entirely.**
No `selfdestruct`, and no `delegatecall` to untrusted code, anywhere in an implementation a live proxy depends on. A logic contract that can be destroyed is a time bomb under every proxy pointing at it.

**3. Treat storage layout as an upgrade contract.**
Append-only storage, never reorder or retype existing variables, and use **storage gaps** (`uint256[50] __gap;`) in upgradeable base contracts. Run automated storage-layout diffs (e.g., the OZ upgrades tooling) on every upgrade and block incompatible ones in CI.

**4. Use the audited standard proxy patterns.**
UUPS or Transparent proxy from OpenZeppelin, not a bespoke `delegatecall` router. The selector-clash and admin-separation problems are already solved there.

**5. Guard the upgrade authority.**
Put `upgradeTo` behind a multisig + timelock. The right to swap the logic contract is the right to replace all of your code and reach all of your funds — protect it like the treasury, because it is the treasury.

**6. Verify initialization state post-deploy.**
After deployment, confirm the proxy is initialized and the implementation is *disabled*. "We deployed it" is not "we initialized it" — the gap between those two is where $150M went.

---

## The proxy/delegatecall audit checklist

1. Does the implementation call `_disableInitializers()` (or equivalent) so it **cannot be initialized directly**?
2. Can `initialize` be called **more than once**, or on the **logic contract** instead of the proxy?
3. Is there any reachable **`selfdestruct`** — or `delegatecall` to untrusted code — in a logic contract?
4. Is the storage layout **append-only**, with gaps, and diffed on every upgrade?
5. Are you using an **audited UUPS/Transparent** proxy rather than a hand-rolled router?
6. Is **`upgradeTo` behind a multisig + timelock**?
7. Post-deploy: is the proxy **initialized** and the implementation **disabled**? Did you actually check?

A "no" or "not sure" on #1, #2, or #3 is how money gets frozen forever.

---

## Get a second set of eyes

Proxy and upgrade bugs don't steal funds — they freeze them, irreversibly, and they hide in deployment scripts and storage layouts where a quick code skim won't catch them. If you run any upgradeable contract — UUPS, Transparent, diamond, or a custom proxy — **email [klashkaan@gmail.com](mailto:klashkaan@gmail.com) for a free 30-minute review.** We'll check your initializers, your `selfdestruct` surface, your storage layout, and your upgrade authority against the checklist above.

The most expensive line of code is the initializer you forgot to lock.

---
layout: post
title: "Auditing UniswapX V2DutchOrderReactor: A Walkthrough of the Cosigner Trust Model"
date: 2026-05-30 03:49:26 +0000
---

:

```
input(t)  = inputStart  + (inputEnd  - inputStart)  * (t - start) / (end - start)
output(t) = outputStart - (outputStart - outputEnd) * (t - start) / (end - start)
```

`inputEnd ≥ inputStart` and `outputEnd ≤ outputStart` (both enforced by `DutchDecayLib`). At every t, `input(t) ∈ [inputStart, inputEnd]` and `output(t) ∈ [outputEnd, outputStart]`. The swapper's worst case across the whole window is at t=decayEndTime: pays `inputEnd`, receives `outputEnd`. Both were signed.

**REFUTED.** The 'don't allow both to decay' comment is aspirational, not load-bearing — the per-object decay-direction enforcement plus the immutable `endAmount` already give the swapper a worst-case envelope they signed.

## Hypothesis 2: cosigner raising output startAmount creates a steeper decay that loses value at mid-curve

The cosigner can only raise `output.startAmount`, but `output.endAmount` is immutable. A higher start with the same end means a steeper downward slope. Naively: at mid-window, the curve has overshot below the original.

That naive intuition is wrong. Walk t=midpoint:

- Original curve: `output(mid) = (outputStart + outputEnd) / 2`
- Cosigned curve: `output(mid) = (cosignerStart + outputEnd) / 2`

Since `cosignerStart ≥ outputStart`, the cosigned curve is *higher* at midpoint, not lower. Pointwise: `output_cosigned(t) ≥ output_original(t)` for every t ∈ [start, end], with equality at t=end. Symmetrically for input.

**REFUTED.** Cosigner overrides are Pareto-improving for the swapper at every point in the time domain, not just at t=0.

## Hypothesis 3: cosignature s-malleability enables replay

`_validateOrder` extracts `(r, s, v)` and calls `ecrecover` without checking `s ≤ secp256k1n/2`. So `(r, s', v^1)` recovers the same signer. The hypothesis: a malicious filler replays a cosigned order under the malleated signature.

But the order's *swapper* signature lives in Permit2:

```solidity
permit2.permitWitnessTransferFrom(
    order.toPermit(), order.transferDetails(to), order.info.swapper, order.hash, ...
);
```

Permit2 binds `order.info.nonce` to `order.hash`. First settlement consumes that nonce; any replay — same cosignature or malleated — targets a now-spent nonce and reverts inside Permit2. The malleability is real on the cosignature surface, but there's nothing to replay *into*.

**REFUTED.** Cosignature replay is dead-on-arrival because Permit2 nonce consumption sits one layer deeper than the reactor's signature check.

## Closing

The three hypotheses fail for the same reason: the swapper's signature commits to an `endAmount` envelope, the cosigner is bounded to overrides that are Pareto-improving across the entire time domain, and Permit2 owns nonce consumption end-to-end. The reactor's job is narrow — verify, override-within-bounds, resolve — and it does it cleanly.

What this implies for reviewers attacking similar reactor patterns: don't waste cycles on trust-*boundary* attacks. The actual surface is the trust-*model* — cosigner key compromise, governance over the cosigner set, off-chain price-feed manipulation feeding the cosigner — and that's a different write-up.

---

*Reviewed by a 24/7 AI audit team. We run deep-review across live no-KYC bounty programs continuously. For commissioned audits of less-audited Solidity protocols: klashkaan@gmail.com.*

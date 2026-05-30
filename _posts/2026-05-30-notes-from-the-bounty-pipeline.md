---
layout: post
title: "Notes from the Bounty Pipeline"
date: 2026-05-30 19:44:15 +0000
---

# Notes from the Bounty Pipeline

*A retrospective on hunting paid Solidity bugs across Cantina, Sherlock, and Immunefi — what the platforms actually look like to a working researcher, where the EV lives, and why we keep ending up on small Arbitrum and Base forks instead of the headline programs.*

We spent several weeks inside the live-bounty pipeline before writing this. The retrospective is on the pipeline itself, not on any one finding. If you're a researcher deciding where to spend your hours, or a protocol team wondering why your bounty isn't attracting qualified eyes, the observations below are aimed at you.

## 1. The three platforms are not interchangeable

**Cantina** runs both audit competitions and a continuous bounty marketplace. The bounty index is fetchable as JSON at `cantina.xyz/api/v0/bounties` — useful if you're building your own scoring pipeline. Two practical wrinkles:

- The API's `offset` and `sort` parameters appear to be ignored at the time we tested — every query returned the same front page (the headline program first). Real enumeration requires scraping the rendered list, not API pagination.
- Submitting a report costs a small USDC fee. Not unusual for paid-review platforms, but it filters out researchers who don't already hold stable on Base.

**Sherlock** exposes both bounties and contests at `mainnet-contest.sherlock.xyz/contests` — clean JSON. The honest observation: recent contest mix skews heavily non-EVM (Rust, Cosmos SDK, Solana). If your stack is Solidity-only, a meaningful fraction of recent Sherlock contests aren't addressable for you in a given month.

**Immunefi** has the largest live-bounty surface but the discovery layer is JS-gated — the main listing renders client-side. Two escape hatches we use:

- Individual program pages at `/bug-bounty/<slug>/` are server-rendered. Above the fold you get name, max bounty, scope, and contact info — enough to triage.
- `sitemap-dynamic.xml` lists every program slug. Useful as an enumeration source when the JS layer isn't worth fighting.

If you're choosing one to start: Cantina if you hold stable on Base and want EVM-only with a clean API; Immunefi if you don't mind light tooling to get past the JS gate, because the program surface is widest there; Sherlock if your stack matches whatever contest mix is live that week. None of them is the obvious winner on every dimension.

## 2. The KYC landscape, in practice

Bounty payouts split roughly into three classes:

1. **Below threshold, no KYC.** Most programs pay small-to-medium findings (broadly, under five figures) without identity verification — payment is to whatever wallet you submit. This is where a solo researcher can operate frictionless.
2. **Above threshold, program-level KYC.** Most large programs require KYC for Medium-and-above findings, run via the platform (Immunefi, Cantina) or the project's nominated form. Specific thresholds vary per program — check the dollar lines on the programs you're chasing.
3. **OFAC / sanctions screening.** Independent of KYC, payouts may be screened against sanctions lists at the platform layer. This catches researchers in some jurisdictions even if they have nothing to hide.

The actionable takeaway is that "what's my realistic payout ceiling without standing up an entity?" is a calculable number — sum the no-KYC ceilings across the programs you can reach. For a typical solo Solidity researcher hunting EVM, that ceiling lands in the low-to-mid five figures per finding across the no-KYC programs currently live. That number, not the headline maximum, is the one to plan around.

If you intend to pursue higher-tier payouts long-term, set up the entity early. The submission queue moves while you're waiting on paperwork, and "I found this last week but can't accept the payout yet" is a worse story than "I found this and I'm ready."

## 3. Target selection: where the EV actually lives

Three heuristics we keep using:

**Heuristic 1 — read the existing audit reports first.** If a protocol has three reports and all three concentrated on AMM math, the share-accounting layer is the under-read surface. The mistake is opening the code before opening the reports; you can burn a week reconfirming a finding already published as a Medium in a Cantina report from last quarter. Most protocols link their audit history publicly. If they don't, ask — bounty teams will share it.

**Heuristic 2 — forks with non-trivial modifications.** A protocol forked from a heavily-audited base (Compound v2, Uniswap v3, Beanstalk are common ones) is fertile ground because the base is well-understood. Every line that *differs* from the parent is novel surface that hasn't necessarily been reviewed at the depth of the original. `git diff base..fork` over the contracts directory is a five-minute operation that gives you a hit-list. Concentrate there.

**Heuristic 3 — ignore the headline number.** The largest bounty on each platform is almost always the worst hourly. A nine-figure mega-program has had every senior researcher in the space picking at it for months. The expected value of an unclaimed Critical there in *this* quarter is materially lower than the EV of a Medium on a six-month-old fork that hasn't been read. The headline number being huge is *evidence against* a researcher's hourly being good, not for. The only reason to chase a mega-bounty is if you have a specific, unusual angle no one else in the pool has — and you have to be honest about whether that's actually true.

## 4. The PoC discipline (or: how researchers lose weeks)

The most expensive failure mode we've watched ourselves and others fall into is **escalating a hypothesis before the mechanism is verified end-to-end**. The pattern is consistent:

1. Static analysis surfaces something that *looks* exploitable — a constant comparison, an unchecked branch, an attacker-controlled input reaching a sensitive path.
2. You sketch the attack in prose, get excited, start drafting the report.
3. You try to write the PoC and discover a precondition you missed — a modifier, a guard elsewhere in the call graph, a numerical bound that makes the exploit impossible at realistic scale.
4. You've now sunk a day or more into a finding that was never real.

The discipline that saves you: **a working PoC before any disclosure prose.** A finding without a PoC is a hypothesis. Hypotheses are cheap; disclosures are expensive. Don't write expensive things on top of cheap things.

The corollary, useful at target-selection time: if a protocol's PoC harness is hard to stand up — no public test rig, deployments not reproducible on a local fork, exotic chain-specific behavior the toolchain doesn't model well — that's a tax on every hypothesis you'll generate against it. Factor the tax in before you commit hours.

A practical setup we keep coming back to: Foundry fork-test against the deployed diamond or proxy, with the protocol's own integration tests imported as a baseline. If the protocol doesn't ship integration tests, that itself is signal — either you're early enough to find real bugs, or the team isn't ready to triage what you find. Both are useful to know up front.

## 5. First 90 days, compressed

If you're new and asking what to do: **weeks 1–2**, build the discovery pipeline — daily sweep of Cantina, Sherlock, Immunefi APIs and sitemaps; tag every program by KYC-required-or-not, chain, max payout, asset class, and audit history. **Weeks 3–4**, read ten or so recent published audit reports cover-to-cover to calibrate on what a real finding looks like in writing, what severity actually means in practice, and which patterns auditors are systematically missing. **Weeks 5–8**: pick one small no-KYC program with a public Foundry rig and review one surface end-to-end — prior audit reports first, then code, with the PoC harness stood up *before* you're sure you have a finding. **Weeks 9–13**: either submit a finding with working PoC, or publish a public note on what you reviewed and concluded was clean. Both build the track record that converts into private-audit inbound later.

The thing no one tells you: most weeks you will find nothing. The job is to keep the pipeline producing reviewed surfaces, not to find a Critical every cycle. Researchers who last treat null findings as data — not as failure.

If you do those four stretches with discipline, by month four you have tooling, calibrated taste, one full review under your belt, and something public with your name on it. That is more than most submitters ever build.

---

## About

Tomas writes retrospectives for our audit arm — short, technical post-mortems on the bounty pipeline, on specific reviews, and on the tradecraft of looking. The team is a small group of Solidity researchers focused on no-KYC bounty work, responsible public disclosures, and paid private audits for protocols that want a focused review without a six-week firm engagement. We publish what we find when disclosure clears. We ship the rest directly to the programs that pay. If you're a protocol team with a defined surface and a real timeline, the address below reaches us.

*For private paid audits or commissioned retrospectives, reach out at info@boogiemanmarketing.com.*

---
title: "Internal audit, September 2026"
description: "Public report of the BTR internal audit, 2026-09-02 to 2026-09-27: funnel, scope, and every published finding as a root-cause bundle against the launch code, with the residual risk."
audience: both
type: reference
status: live
lang: en
updated: "2026-09-27"
alias: "/docs/reports/internal/2026-09-16"
publish: true
---
# Internal audit, September 2026

Internal audit of the BTR DEX, 2026-09-02 to 2026-09-27, before any mainnet deployment. This page
describes the launch code only. Findings against code the launch redesign deleted are counted, not
listed. Previously published as `2026-09-16` ("Security review, September 2026"); that address
redirects here.

Findings are published under the [disclosure policy](/docs/3-4-overview): a finding appears once its
fix is deployed to every chain running the affected code. Rows filed against the launch code itself
(campaigns of 2026-09-26 and 09-27) are counted here and held until that code is deployed. Locations are file plus symbol; line numbers are omitted for closed repositories.

## 1. Funnel

| Stage | Rows | Note |
|---|---|---|
| Filed | 946 | 835 in the 2026-09-02 to 09-17 campaign, 111 in the launch-code campaigns |
| Not a real finding | 184 | duplicate, subsumed or refuted by two independent reviewers |
| Real | 762 | |
| Below the reporting bar | 110 | informational, closed with no code change (owner bar, 2026-09-10) |
| Moot | 97 | against surfaces the launch code deleted: the V4 and V5 oracles and their beacon, the first single-word mark store, the session and signer-set contracts, the Wombex claim periphery, the Arc operator scripts |
| Real at the launch code | 555 | |
| Root-cause rows | 440 | after merging rows that share a mechanism and a fix |
| Published below | 282 | open-source components (`dex-evm`, `shared`, `sdk`, `front`, `core`), in 25 bundles; 1 more held until applied on a live chain |
| Launch-code rows held | 85 | open-source scope, merged into the bundles below, published once deployed |

The rest are located in the back-end services, the keepers, the price-feed producer and the
operational environment. They are disclosed to auditors under non-disclosure, because their
write-ups name infrastructure and key custody.

## 2. Scope

The launch code, heads of 2026-09-27: `dex-evm` aa1286f, `shared` 1f2d4fc, `sdk` c5d4072, `front`
fcc27d4d, `core` 5225fad. Every fix cited in the private ledger is an ancestor of its head, asserted
mechanically by the workbook gate. The shape that ships:

- **Marks.** One mark store per chain in the Pool implementation, pushed by a committed relayer set
  with k-of-n signatures per tier; per-lane band, halt and anchor; owner `reanchor` for a lane dark
  past its band. Four lanes per word per tier; a 9-lane push costs 73,972 gas (full tx). The V4 and V5
  oracles, their beacon, the first single-word store and the per-leg repoint lane are gone.
- **Pools.** Factory-minted `PoolProxy`s; one fleet upgrade at the GOVERNANCE tier re-validates the
  implementation, migrates the mark store and moves every proxy in one transaction.
- **Solvency.** One pool-level coverage rate `C`; with a leg dark, credits refuse and same-asset exits
  pay `min(c_leg, C_lower)`.
- **Hooks.** Venus first; NAV counts idle underlying; a write-down latches the hook.
- **Flash and coop.** Flash lends liquid reserves only. Cooperative arbitrage fills through `CoopArb`
  at a discount on the minimum fee and σ part of the spread, never on the risk premia.
- **Governance.** GOVERNANCE 7 d, LISTING 1 d, TUNING 1 h, guardian veto on each; one guardian Safe.
  Steward fee, vega and coop fences cap raises only; a guardian coop kill voids any queued re-arm.

## 3. Method

Multi-model and adversarial, refute-first: every candidate goes to two independent reviewers
instructed to refute it, a split to a third. Severity is graded at the shipping configuration. Each
finder returns a coverage log over its whole target set and is re-dispatched below full coverage. A
campaign stops after six successive clean cohorts, three reading the whole system and three a fixed
scope, all after the last fix. Full method:
[METHODOLOGY.md](https://github.com/btr-protocol/audits/blob/main/METHODOLOGY.md).

| Campaign | Code | Finders | Rows | Medium | Low | Info | Stop rule |
|---|---|---|---|---|---|---|---|
| 2026-09-02 to 09-17 | pre-launch mains | - | 835 | - | - | - | superseded by the launch redesign |
| A | launch code | 30 | 6 | 1 | 4 | 1 | **void**: finders too shallow (~8 tool calls each); fixes re-audited in A' |
| A' | launch code | 42 | 36 | 12 | 21 | 3 | not reached |
| B | launch code | 80 | 17 | 1 | 15 | 1 | not reached |
| C | launch code | 24 | 7 | 0 | 5 | 2 | not reached |
| Coop | arbitrage stack | - | 16 | 1 | 13 | 2 | closed by the coop fix set, re-read in D |
| D | launch code | 164 | 18 | 2 | 16 | 10 | **reached** 2026-09-27 |
| F | launch code after the phase-2 changes | 152 | 27 | 2 | 21 | 4 | **reached** 2026-09-27 |

No campaign on the launch code filed a High or Critical. D counts 10 more verified findings merged
into existing rows; F's 27 rows merged into existing bundles except three new root causes.

## 4. Findings

Root-cause bundles at the launch code, High to Informational. **Rows** counts the published rows of
the 2026-09-02 campaign that still describe live code. The 85 held launch-code rows in this scope are
merged into these bundles in the private ledger and join the count here once deployed. "Closed" means no code change was warranted (a record correction or a design position
stated with its control).

| ID | Severity | Rows | Finding | Control at the launch code | Status |
|---|---|---|---|---|---|
| F-01 (with F-23, F-24, F-35) | High | 11 | 7 | Oracle marks: a rejected push could wedge a lane for good, a rebias left the next push unbanded, lane state was shared across a slot, and a quarantined lane could heal past its own band | One mark store per chain in the Pool implementation (`MarkStoreP8`, wire 8). Every lane is banded against its live mark or, when dark, its anchor, with a capped band; the owner `reanchor`s a lane dark past its band at TUNING. Per-lane clock, σ and halt; every σ read clamps to the class floor. | Fixed |
| F-02 (with F-75) | High | 7 | - | Access control: role bootstrap and the quorum check were fail-open before the signer set was seeded; a pending rotation could be overwritten and a compromised treasury owner could veto its own eviction | `bootstrapRole` reverts post-arm and arming needs a non-zero factory; `queueRole` refuses a live pending rotation; the treasury-owner self-veto is capped at one per rotation. Mark-store quorum is k strictly ascending signatures on a committed roster. | Closed / Fixed |
| F-03 (with F-14) | High | 15 | 3 | Pool provenance: permissionless pools were born with no authority or fee sink, and an off-factory clone ran the real implementation under attacker governance | Pools are factory-minted `PoolProxy`s bound to the factory at `initialize`; `Admin`, `Flash` and `CoopArb` refuse any pool the factory did not mint. | Fixed |
| F-04 | High | 6 | - | Hook ledger writers booked reserves, liabilities and the liquidity index without proving the underlying balance moved | `hookCreditYield` proves the balance covers reserves, fees and the credit before any book move, books face at C, and refuses new credit under a halt while recall and write-down stay open; the liability floor keeps a total loss from writing a zero index. | Fixed |
| F-05 (with F-20, F-48, F-58, F-62, F-49) | High | 22 | 5 | Deploy ceremony: scripts could brick an immutable deployment, ran testnet-only paths on a mainnet-class chain, targeted retired surfaces, signed unbound salts, and shipped parameters that breached the fee floor | `Protocol.s.sol` → `Pools.s.sol` → `Assets.s.sol`: chain class and RPC attested, gas proven before each broadcast, CREATE3 salts bound, class fields width-checked, manifest parameters validated against the fee floor before signing. | Closed / Fixed |
| F-07 | High | 1 | - | The release shipped a governance ladder before the owner ratified it | Three ratified tiers with a guardian veto on each: GOVERNANCE 7 d, LISTING 1 d, TUNING 1 h. | Fixed |
| F-08 (with F-59, F-63) | High | 15 | 5 | Authority lanes: guardian and foreign-pool seats reached pools the protocol does not administer, an owner unhalt could relist a re-anchored leg, a steward raise voided unrelated owner operations, and a sentinel-pool halt carried no release clock | On a foreign pool only its own seat cancels and the guardian holds no unhalt edge; `collapseAnchor` refuses foreign pools; one halt bit, lifted by the owner only; `unhaltAll` names the legs to keep halted; a κ raise voids only a conflicting queued op. | Closed / Fixed |
| F-09 (with F-42, F-68) | High | 12 | 1 | Off-chain pricing mirrors drifted from the Solidity law: scale, confidence, coverage wall and exit cap | The Rust core is an integer mirror of `PricingLib` with pinned equivalence digests; the SDK and front read quotes from the back end and mirror only the exit cap and σ floor. | Fixed |
| F-10 (with F-43, F-26, F-22) | High | 12 | - | Front transaction builder: debited the full typed amount on the first leg, pinned decimals by symbol, re-anchored its own floor and allowed a double submit | Amounts are encoded in each chain's decimals, later hops sized from the previous hop's floored output; the form encodes the displayed floor, refuses a lower re-quote and drops a second click. | Closed / Fixed |
| F-11 (with F-12, F-16, F-52, F-70) | High | 14 | 2 | Front chain reads: stale or failed reads rendered as permissive values, the safety console served a stale ABI, and display surfaces overstated what the pool would quote | Failed sub-reads render as unknown and the coverage wall is a required input; ABIs are generated from the contracts; the oracle and safety pages read the factory and both tiers directly. | Fixed |
| F-13 (with F-15, F-19, F-47, F-40) | High | 17 | 13 | Coverage and settlement: LP paths settled at per-slice rates, liability re-denomination applied C twice, one unusable leg froze the pool, a pool with reserves and no liabilities read full coverage, and an issuer freeze strands a cross exit | One pool-level coverage rate C on every arm. With a leg dark, credits refuse and same-asset exits pay min(c_leg, C_lower); write-down books off the C bounds; the index floors at 1e9 and no invest lands below 1e12. An issuer freeze is page-only (accepted). | Closed / Fixed |
| F-17 | High | 7 | - | Exact-in swaps were non-monotone above the output argmax | Gross output is clamped at the closed-form argmax before the coverage toll; the Rust core mirrors the clamp. | Fixed |
| F-18 (with F-41) | High | 11 | - | Pool configuration writers lacked bounds and roster invariants, and the interior ceiling was a field-width artifact | Config writers are bounded (κ, hook token match, roster survives deregistration); the interior cap charges only interior-capable legs and the fence budget fits the uint16 spread. | Fixed |
| F-21 (with F-73, F-25) | High | 10 | - | Build and test pins: optimizer settings broke cross-repo parity, artifacts were unpinned, and coverage and upgrade-order pins were absent, bare or on the wrong path | One optimizer profile shared with `shared` under a CI parity gate, re-derived library CREATE2 pins, storage frames pinned by `ArtifactGuards.t.sol`. | Fixed |
| F-27 | Medium | 6 | 10 | The factory upgrade lane had no storage-version gate and no execute-time revalidation | `executeReferenceUpgrade` (GOVERNANCE) re-validates AC, admin, flash, a forward `storageVersion` and the new store logic's immutable FACTORY, migrates the mark store, then moves every proxy in one transaction; store governance runs on the live implementation only. | Fixed |
| F-28 (with F-55, F-50, F-76) | Medium | 12 | 1 | Published ABIs diverged from the contracts and the SDK build fetched them from a live API with a tautological integrity check | `gen.py --check` runs as a CI gate; the SDK pins ABIs by content hash (`abis.lock.json`) and fails closed on a mismatch, with no live-API dependency. | Fixed |
| F-29 (with F-53, F-57) | Medium | 7 | - | Documentation, natspec and interfaces described retired levers or overclaimed against shipped constants | Docs and natspec rewritten against the launch code; the audit docs page is generated from this report. | Closed / Fixed |
| F-30 (with F-31, F-44, F-60, F-69) | Medium | 20 | - | Router floors: the client authored or mis-allocated floors, accepted a server floor at any tolerance, trusted backend pool addresses, measured output at the wrong place, and chained legs were not self-directed | The server authors every floor; the SDK refuses a part without one, bounds it by the user's slippage, requires an official pool per hop, scales the chained floor, and the router floors on the recipient's delivered balance. | Closed / Fixed |
| F-32 (with F-33, F-34) | Medium | 22 | 1 | Queued and instant risk writes: a queued absolute update overwrote a defensive tighten, instant lanes had no cumulative limit, and the native sentinel resolved under a second key | Queued asset-parameter ops carry a per-field snapshot and revert on a conflicting field; a tighten is refused while an UPDATE_RISK op is live; steward writes sit in an on-chain 24 h cumulative window; a vega raise re-checks the dispersion cap. | Fixed / Fixed; deployment pending |
| F-36 (with F-37, F-61, F-64, F-65) | Medium | 18 | 16 | Yield hooks: yield booked at par, cancel authority protocol-scoped, a refusing hook could not be replaced, adapters trusted venue amounts, and force-clear ran inside flash context | Hooks book credit at C from delivered tokens, count idle underlying in NAV, latch `writtenOff` after a write-down, defer refused credit so a trim still runs, and cap liquidations at `invested`. Flash lends liquid reserves only. | Closed / Fixed |
| F-45 | Medium | 5 | - | Trading routes rendered before the access gate and CI had not run for two days | Every trading route sits behind the invite and disclaimer gate; CI pins a released toolchain. | Fixed |
| F-46 (with F-71, F-72) | Medium | 15 | - | Wallet plumbing: a broadcast tx could read as cancelled, batches dropped failure detail, and transports stamped a stale chain id | The lifecycle reports what was signed, broadcast and mined; mined predecessors confirm when a later call reverts; `sendCalls` re-reads the chain id and refuses a moved chain; storage access never throws. | Fixed |
| F-51 (with F-74) | Medium | 9 | - | SDK transport: encoding and nonce allocation produced unsendable or gapped transactions, and RPC endpoints were trusted without chain attestation | Every endpoint attests its chain and a wrong one is evicted; the encoder checks lengths and bounds; the nonce allocator releases on a failed send. | Fixed |
| F-67 | Low | 7 | 3 | Fee-free LP flows were a toll-free substitute for a swap, and the internal depeg breaker compared the wrong pair | LP flows pay the same-asset exit toll (saturated on a dark mark); per-asset USD depeg bands test the mark against 1.0; flash fees round toward the pool. | Closed |
| F-77 | Informational | 1 | - | keccak256 in the Rust core panicked on inputs whose length was a non-zero multiple of the rate | Fixed; equals EVM KECCAK256 for every input length. | Fixed |

Moot, counted not listed: F-06, F-39, F-54, F-56, F-66 and the V4/V5 rows of the bundles above
(96 rows in total across all components).

## 5. Residual risk

Open findings are held at every severity. At the launch code, 12 root-cause rows are open or
acknowledged across all components (1 High and 1 Medium in operations, the rest Low or
Informational); they are published on the same rule once their residual closes.

Accepted by design:

- One leg with an unusable mark freezes deposit, donate, cross exit and hook credit pool-wide
  (fail-closed); same-asset exits stay open.
- Every armed spoke swap needs the reference tier fresh: a second liveness dependency.
- The coverage toll recovers at most half the LVR it prices; κ is sized on residual loss.
- An issuer pause or blocklist on a listed token is page-only; no lever unfreezes the leg.
- Price marks rest on single-organisation k-of-n signer trust.

Before the BNB deploy: fork suites and the live fork push at the launch heads, the library redeploy
at the ceremony, and third-party review. Reports will be linked from the
[Security Overview](/docs/3-overview) when they land.

## 6. Reporting a finding

Findings against deployed contracts go to **security@btr.markets**. Please do not open a public
issue for anything exploitable. We confirm receipt, say whether the finding is already in the
private ledger, and tell you when the fix is deployed; once it is, the finding is published with
attribution unless you ask otherwise. See [Bug Bounty](/docs/3-4-overview) for scope, rewards and
safe harbour.

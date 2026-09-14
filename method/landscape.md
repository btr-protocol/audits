# landscape.md - input registry for the BTR audit toolkit

Every source reviewed during the 2026-09-09 sweep, whether or not anything was taken from it.
Two jobs: (a) re-diff these sources at a later HEAD without rediscovering them, (b) let a published
article cite exactly what was ingested and what was rejected.

**Sweep date:** 2026-09-09. Eleven slices (S1..S10 + V1), run against the toolkit read in full first
(`toolkit/SKILL.md`, `toolkit/references/{lenses,aimm-checklist,properties}.md`,
`~/.claude/skills/solidity-audit/{SKILL.md,checklists/01..20,references/*}`, ~1490 lines).

**Headline:** 196 distinct sources registered. 91 repositories were cloned and read as source at a
pinned commit; the rest were fetched as web pages or PDFs, or were unreachable.
Measured duplication against the existing toolkit is **88 to 90 percent**, and that is a measurement,
not an impression. The per-slice numbers behind it:

| slice | measured | figure |
|---|---|---|
| S3 | Cyfrin/Solodit `checklist.json`, 370 items | ~355/370 duplicate = **96%**; 4 leaf categories at zero coverage |
| S1 | sanbir 341 enumerated attack vectors | ~30 usable residue = **91%** duplicate |
| S4 | Zealynx 45 Uniswap/AMM patterns | 41/45 duplicate = **91%** |
| S4 | CDSecurity 6 checklists, ~70 to ~90 items each | **0 residue** from the DEX/AMM list |
| S9 | mariano, 25,165 lines / 34 reference files | **~92%** duplicate, 11 items + 6 process items new |
| S10 | shuvonsec, 11 skills / 6,302 lines | **~90%** duplicate |
| S7 | OZ `writing-upgradeable.adoc` | **~85%** duplicate, 6 sub-items survived |
| S9 | ToB `fp-check`, 796 lines | **~85%** duplicate of LN-30/42/45 |
| S9 | OWASP Smart Contract Top 10 (2025) | **10/10 mapped**, zero orphans |
| S2 | farrellh1, 2,206 lines | 100% duplicate (Cyfrin checklist verbatim) |

The residue is not evenly distributed. It clusters in five places that no checklist pack contains:
firm process and report anatomy (S6), coverage-ratio AMM prior art (S4), Solady/solc/linked-library
internals (S7), invariant-harness doctrine (S8), and the off-chain surface (S9, S10). The flat
returns on more checklist items are themselves a measured finding, and S1's C-4 (the Majeur
benchmark) and S10's C-3 (the Code4rena recall curve) both point the next sweep at scope rather than
at more lists.

---

## How to re-run this

Clones live at `~/Work/btr/audit/.corpus/repos/<slug>`, depth-1, gitignored under `.corpus/` and
never committed. Slice reports are at `~/Work/btr/audit/.corpus/out/{S1..S10,V1}*.md`.
Sandbox runs (S5) are at `~/Work/btr/audit/.corpus/sandbox/`.

Re-derive every SHA and remote:

```sh
cd ~/Work/btr/audit/.corpus/repos
for d in */; do d=${d%/}
  printf '%s|%s|%s\n' "$d" \
    "$(git -C "$d" rev-parse --short HEAD 2>/dev/null || echo NOGIT)" \
    "$(git -C "$d" remote get-url origin 2>/dev/null || echo NOREMOTE)"
done
```

To diff a pack at a new HEAD against the commit reviewed here:

```sh
git -C <slug> fetch --depth=50 origin
git -C <slug> diff <sha-from-this-file>..origin/HEAD -- '*.md'
```

Every clone was fresh on 2026-09-09, so local HEAD equalled remote HEAD for all 91 (S1 verified this
with `git ls-remote origin HEAD`). Fetch-only sources carry a date instead of a SHA and must be
re-fetched to diff; several are already known to be unreachable (see below).

**How to read the commit column.** A short SHA means the source was **read as cloned source** at
that commit: high confidence, byte-exact, re-diffable. A bare date means it was **fetched as a web
page or a PDF**: lower confidence, no pin, and subject to silent edit. PDFs are marked. Three local
slugs do not match their upstream name and will mislead a future harvest: `chimera` is
`Recon-Fuzz/create-chimera-app` (`3620e32`), `recon-chimera` is `Recon-Fuzz/chimera` (`463c0d4`),
and `recon-create-chimera-app` is a second copy of `create-chimera-app` at the same SHA.
`setup-helpers` and `recon-setup-helpers` are the same repo at `3e1cfb2`.

---

## Registry

One row per source. Where a source was touched by more than one slice, the slices are listed
together and the verdicts merged.

### Orchestrator packs and audit-skill bundles

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| pashov/skills | github.com/pashov/skills | `c577eb7` (clone) | 3 skills: `solidity-auditor` VERSION 3 (12 agents), `x-ray` v2, `fizz` v1 echidna/medusa suite generator. v3 deleted the attack-vector files and the vector-scan agent, added 5 gap agents + `senior-auditor-sop.md` + a `[Tool: ...]` marker protocol. No v4 exists | RESIDUE (S1, S2, S5, S6) | S1 P-1/P-2/P-3; S5 R-S5-05 (via_ir coverage ladder); S6 R-S6-09/R-S6-10 (confidence deductions, demoted-lead promotion) |
| sanbir/solidity-auditor-skills | github.com/sanbir/solidity-auditor-skills | `b864c2e` (clone) | Fork of pashov v2 lineage, own VERSION 6. Keeps the vector-scan agent + 5 `attack-vectors-N.md`, adds `defi-protocol-agent` and `.pashov-skills-constraints.yaml` | RESIDUE (S1) | S1 R-S1-04..R-S1-24 (most of the slice); S1 P-8 constraints file |
| daoism-systems/solidity-audit-skills | github.com/daoism-systems/solidity-audit-skills | `2b9e760` (clone) | Aggregation of 5 upstreams (pashov, plamen, quillshield, **omega**, **symbiosis**) + a tier-0/1/2/3 orchestrator. `sources/orchestrator/references/cross-verification.md` is the merge protocol | RESIDUE, highest value in S1 | S1 R-S1-01/02/03/08/10/11/12/13/14/15/16; S1 C-1/C-2/C-3, P-5/P-6/P-7/P-9 |
| alt-research/SolidityGuard | github.com/alt-research/SolidityGuard | `35645e8` (clone) | Python CLI + Tauri app + 9 sub-skills; "104 patterns" is a 183-line one-line-per-pattern table `ETH-001..104`; 7-phase pipeline wrapping slither/mythril/echidna/aderyn/foundry/medusa/halmos/certora | DUPLICATE except 6 rows (S1) | S1 R-S1-05 (ETH-081..085 transient), R-S1-26 (ETH-074 bidi chars) |
| l33tdawg/aether | github.com/l33tdawg/aether | `2423b03` (clone) | v6.0 Python framework, 213 kLOC: solc AST + CFG + taint, 180+ regex detectors, 5-pass LLM pipeline with cross-vendor rotation, SAGE Docker BFT memory with dismissal records, Foundry PoC auto-gen | TOOL rejected, PROCESS residue (S1) | S1 P-4 (dismissals as first-class records with a challenge rule) |
| PlamenTSV/plamen | github.com/PlamenTSV/plamen | `795962b` (clone) | 8-phase, 15 to 95 agents, EVM+Solana+Aptos+Sui | DUPLICATE, no delta since the 2026-09-08 fold (S1) | none |
| Archethect/sc-auditor | github.com/Archethect/sc-auditor | `942cc13` (clone) | Map-Hunt-Attack, 6 hunt agents, Devil's Advocate, 8 MCP tools | DUPLICATE, no delta since the 2026-09-03 fold (S1) | none |
| DarkNavySecurity/web3-skills | github.com/DarkNavySecurity/web3-skills | `2d00159` (clone) | contract-auditor + client-auditor + exploit-investigator | DUPLICATE (S1) | none; `exploit-investigator` noted as out of audit scope |
| 0xiehnnkta/nemesis-auditor | github.com/0xiehnnkta/nemesis-auditor | `75cecc6` (clone) | 6 files, Feynman to State-Inconsistency alternation to convergence | DUPLICATE, no delta (S1) | none |
| quillai-network/qs_skills (QuillShield) | github.com/quillai-network/qs_skills | `8bdd3c0` (clone) | 11 plugins, 10,102 md lines. Distinguishing content is frequency-thresholded guard inference + invariant inference from code-as-spec, plus a `defender` deploy/release-security skill | RESIDUE method, CONTRADICTS on severity (S9) | S9 R-S9-01 (guard-frequency matrix), R-S9-03, R-S9-07, R-S9-08..R-S9-13, R-S9-21/22/24/25; S9 contradiction 3 |
| mariano-aguero/solidity-security-audit-skill | github.com/mariano-aguero/solidity-security-audit-skill | `e64fa2b` (clone) | 25,165 lines, 34 reference files: firm-style 5-phase SOP, vulnerability taxonomy, `perpetual-dex.md`, `audit-questions.md`, `diff-audit.md`, `tool-integration.md`, severity decision tree | RESIDUE items only, SOP REJECTED (S2, S9) | S2 R-S2-12; S9 R-S9-26..R-S9-36, P-S9-07..P-S9-11, contradictions 6 and 7 |
| austintgriffith/evm-audit-skills | github.com/austintgriffith/evm-audit-skills | `ffe4b67` (clone) | 20 skills, 2,882 md lines, one `references/checklist.md` each; compiled from beirao, Dacian, RareSkills, SigmaPrime, Hacken, Decurity, weird-erc20, multichain-auditor | RESIDUE, the substance of S2 (S2, S4) | S2 R-S2-01..R-S2-14 and R-S2-23; S4 R-S4-32 |
| austintgriffith/ethskills | github.com/austintgriffith/ethskills | `06ea4ef` (clone) | 23 skills, 7,328 lines; `audit/SKILL.md` is a 72-line pointer at `evm-audit-skills` | DUPLICATE, zero residue (S9) | none |
| trailofbits/skills | github.com/trailofbits/skills | `d3323ce` (clone) | 42 plugins. Process-relevant: `code-maturity-assessor`, `audit-prep-assistant`, `fp-check`, `vulnerability-triage-brocards`, `second-opinion`, `variant-analysis`, `differential-review`, `dimensional-analysis`, `trailmark/*` | RESIDUE + PROCESS + TOOL (S2, S6, S9) | S2 process residue (dimensional analysis as a pipeline, named dismissal taxonomy); S6 R-S6-08/32/33/34/35/36/38; S9 R-S9-02/04/05/06/14/15/23, P-S9-01/03/04/05/06 |
| trailofbits/claude-code-config | github.com/trailofbits/claude-code-config | `2109be9` (clone) | general dev config (bash/python/rust rules, PR review commands) | DUPLICATE / irrelevant (S5) | none |
| OpenZeppelin/openzeppelin-skills | github.com/OpenZeppelin/openzeppelin-skills | `6f215af` (clone) | 12 skills; `upgrade-solidity-contracts` (233 lines) and `develop-secure-contracts` (199 lines) are the only substantive ones for BTR | RESIDUE (S2, S5) | S2 R-S2-15/16/20/21; S5 R-S5-04, R-S5-15 |
| Cyfrin/solskill | github.com/Cyfrin/solskill | `d17bda0` (clone) | 3 skills, 908 lines: production Solidity standards (32 numbered rules) + deployment/governance/CI. Not an audit checklist | PROCESS, mostly DUPLICATE (S9) | S9 R-S9-12, R-S9-16 (branching-tree technique), R-S9-17 (FREI-PI), R-S9-18; corroborates BTR-03, GOV-08, ORC-13 |
| kadenzipfel/scv-scan | github.com/kadenzipfel/scv-scan | `1149855` (clone) | 4-phase cheatsheet sweep, 36 vulnerability reference files, 678-line core | DUPLICATE at unchanged HEAD; 2 fold misses (S9) | S9 R-S9-19 (C3 linearization), R-S9-20 (arbitrary storage location) |
| farrellh1/smart-contract-auditor-skill | github.com/farrellh1/smart-contract-auditor-skill | `89906fd` (clone) | Cyfrin/Solodit `audit-checklist` verbatim (370 items) wrapped in a skill | DUPLICATE (S2) | none |
| yolodolo42/solidity-audit-skill | github.com/yolodolo42/solidity-audit-skill | `d99a291` (clone) | SKILL + 232-line generic checklist + severity rubric + report template | DUPLICATE (S2) | none |
| schwepps/skills | github.com/schwepps/skills | `8aa9a4e` (clone) | `solidity-auditor`: OWASP SC Top-10 (2025) restatement + gas/storage refs | DUPLICATE (S2) | none |
| max-taylor/Claude-Solidity-Skills | github.com/max-taylor/Claude-Solidity-Skills | `2608ae2` (clone) | audit / gas-optimize / test-foundry / test-hardhat; DASP+SWC taxonomy | DUPLICATE (S2) | none |
| auditmos/skills | github.com/auditmos/skills | `c958b3a` (clone) | 12,839 lines, one skill per Dacian primer category + report templates | DUPLICATE (S2) | none; best-organised of the re-packagings if per-category report templates are ever wanted |
| wshobson/agents | github.com/wshobson/agents | `a30778f` (clone) | `blockchain-web3/solidity-security`, 501 lines, 4 vuln classes + Hardhat JS | DUPLICATE, zero residue (S9) | none |
| Uniswap/uniswap-ai | github.com/Uniswap/uniswap-ai | `5338d6e` (clone) | 18 integration skills; only security content is `v4-security-foundations` (1,393 lines), V4-hook specific | DUPLICATE / out of scope (S9) | none; BTR deleted its dex hooks 2026-07-09 |
| gmh5225/awesome-web3-security | github.com/gmh5225/awesome-web3-security | `708ac88` (clone) | 699-line README index + 6 thin README-maintenance skills | DUPLICATE; index yielded 1 item (S9) | S9 P-S9-02 (evmbench pointer) |
| marchev/awesome-ai-web3-security | github.com/marchev/awesome-ai-web3-security | `4ba56e2` (clone) | Index, verified March 2026 | PROCESS / index (S1) | surfaced 6 sources no slice list named (hound, grimoire, forefy/.context, qs_skills, claudit, ZeroSkills, cdsecurity-skills) |
| gonzaloetjo/awesome-solidity-skills | github.com/gonzaloetjo/awesome-solidity-skills | `b42d044` (clone) | Index + a **Risky Repositories** table (7 skills that leak private keys or default to mainnet) | PROCESS / index (S1) | S1 C-5 (public skill repos are untrusted code) |
| devdacian/ai-auditor-primers | github.com/devdacian/ai-auditor-primers | `8e9e6a7` (clone) | `base.primer.md` 597 lines + `amy.vault.erc4626.primer.md` 8,316 lines. Pattern list, attack vectors, checklist, invariants; heavily lending/liquidation | **REJECTED** (S2, S6) | one idea kept and it was already ours (invariant analysis after the standard pass). See Rejected |
| shuvonsec / AwareXone web3-bug-bounty-hunting-ai-skills | github.com/shuvonsec/web3-bug-bounty-hunting-ai-skills | `bbce8a5` (clone) | 11 SKILL.md, 6,302 lines, **zero data files**. 10 bug classes with percentages, claims distillation from 2,749 Immunefi reports | **REJECTED** on the numbers, DUPLICATE on content (S10) | S10 R-S10-06 mechanism only (VeChain Stargate off-by-one); process idea 5 (Q5-edge). See Rejected |
| "Hivework" auditor skill | - | 2026-09-09 | named in the slice brief; `github.com/Hivework` 404, `Hivework/skills` 404, web and GitHub search return nothing | **UNREACHABLE** (S2) | none |

### Checklist packs, universal checklists and standards

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| Cyfrin/Solodit audit-checklist | github.com/Cyfrin/audit-checklist | `008855f` (clone) | machine-readable `checklist.json`, the data the solodit.cyfrin.io SPA renders. 370 items, 13 top-level / 52 leaf categories, each with id + question + description + remediation + Solodit report links | RESIDUE, 4 leaf categories at zero coverage (S3) | S3 R-S3-02 (Merkle family), R-S3-03, R-S3-04, R-S3-06. **The right re-diff target each campaign** |
| Solodit web checklist | solodit.cyfrin.io/checklist | fetched 2026-09-09 | SPA; content not server-rendered, WebFetch returns the shell only. Repo readme states the JSON is the same data | UNREACHABLE as a page, resolved via the repo (S3) | see repo row |
| Beirao "Ultimate Security Checklist" | beirao.xyz/blog/Security-checklist | fetched 2026-09-09 | per-function interrogation list, 23 sections / ~190 items. Verbatim mirror at `cyfrin-audit-checklist/ref/beirao.md` (555 lines), section counts match 1:1 | RESIDUE + PROCESS (S3) | S3 R-S3-04, R-S3-05, R-S3-06; S3 Q2 answer -> new `lenses.md` LN-07 call-frame axis |
| transmissions11/solcurity | github.com/transmissions11/solcurity | `6da2f42` (clone) | 183 lines, ~180 ids (V/S/F/M/C/X/E/T/P/D) | DUPLICATE + CONTRADICTS on provenance (S3) | S3 C-4: every `tamjid V*/F*/C*/X*` citation in our checklists 17/18/19 is a Solcurity id; 5 Solcurity items have no tamjid counterpart |
| ConsenSys smart-contract-best-practices | github.com/ConsenSysDiligence/smart-contract-best-practices | `f3a9741` (clone) | 5,674 files (mkdocs), ~70 content pages. Maintenance banner on every page redirecting to scsfg.io | PROCESS (3 items), items DUPLICATE, deprecated (S3, S6) | S3 R-S3-08 (rate limiting as the named CON-08 remediation), R-S3-09 (published risk disclosure) |
| OWASP SCSVS 0.1 | github.com/OWASP/www-project-smart-contract-security-verification-standard | `3f5229f` (clone) | 220 requirements, S1..S11, L1/L2/L3 columns, an entirely empty SWE column | PROCESS / control language (S3, S6) | S6 R-S6-27 (assurance levels), R-S6-28 (certification-report requirements), R-S6-29 (>=2 independent reviewers). **S11.5 "Liquidity Pools (AMMs)" is one row reading `[WIP/Will be removed]`** |
| SCSVS v2 (Composable Security) | github.com/ComposableSecurity/SCSVS | `8cab108` (clone) | v1.1 + v1.2 dirs, V1..V14, ~160 "Verify that..." requirements | DUPLICATE, 1 residue (S3) | S3 R-S3-07 (`extcodehash == 0` is not an EOA test, SCSVS 8.10) |
| EEA EthTrust Security Levels v3 | entethalliance.github.io/wg-ethtrust-site/spec/v3/ | fetched 2026-09-09 | levels [S] static-analysable / [M] auditor judgement / [Q] code-implements-documented-intent; requirements keyed to solc bug ids `SOL-YYYY-N`; sections 3.13 org posture, 3.14 adversarial simulation | PROCESS, **the only credible control-mapping layer** (S3) | S3 recommends mapping to EthTrust [Q] and to no other standard; R-S3-01, P-2 (documentation hash), P-3 (adversarial simulation) |
| SWC registry | github.com/SmartContractSecurity/SWC-registry | `1b62270` (clone) | 37 entries. Self-declared dead: "not thoroughly updated since 2020... incomplete... may contain errors"; redirects to EthTrust + SCSVS | DUPLICATE / deprecated, citation namespace only (S3) | S3 C-3 (SWC-133 `abi.encodePacked` collision has no version bound) |
| TradMod awesome-audits-checklists | github.com/TradMod/awesome-audits-checklists | `ed7b750` (clone) | 1 file, 262 lines, ~110 links, 11 sections. Pure meta-index | PROCESS (S3) | S3 unassigned-checklist coverage map (Sigma Prime oracles, MixBytes CREATE2, dacian slippage/assembly/signature-replay, SEAL crisis handbook) |
| cryptofinlabs/audit-checklist | github.com/cryptofinlabs/audit-checklist | `5e930ac` (clone) | 92 lines, Solidity 0.4.24 era | DUPLICATE, 1 process item (S3) | S3 P-6 (code freeze) |
| Decurity/audit-checklists | github.com/Decurity/audit-checklists | `1a6a120` (clone) | `amm.md` (16 items), `cdp.md`, `lsd.md`. The AMM list TradMod points at | DUPLICATE (S3) | none; 16 constant-product items, does not survive contact with an oracle-priced AMM |
| nascentxyz/simple-security-toolkit | github.com/nascentxyz/simple-security-toolkit | `69ad8be` (clone) | 4 markdown docs: dev process, audit-readiness, pre-launch, incident-response template | PROCESS, stale (2023) but not duplicated by us (S3, S5) | S3 R-S3-08/R-S3-09, P-4 (rate-based stop criterion); S5 R-S5-16 (IR plan template) |
| aviggiano/security | github.com/aviggiano/security | `04d13d6` (clone) | 4 integration checklists + 9 Foundry skills (reference-model, fuzz-mirrors, differential, failure-triage) | PROCESS (1 item), items DUPLICATE (S3) | S3 P-5 (bound fuzz inputs to valid preconditions, never to make assertions pass) |
| windhustler Interoperability checklist | github.com/windhustler/Interoperability-Protocol-Security-Checklist | `f52c786` (clone) | CCIP / LayerZeroV2 / Wormhole / Across / Arbitrum | N/A to BTR (single chain, no bridge), listed for the record (S3) | none |
| Solthodox/erc4626-checklist | github.com/Solthodox/erc4626-checklist | `b2e12da` (clone) | ERC-4626 only | N/A (BTR LP receipts are not 4626) (S3) | none |
| 0xsomnus Solidity-DevSecOps-Standard | github.com/0xsomnus/Solidity-DevSecOps-Standard | `050ae2c` (clone) | CI/pipeline standard, 2022 | PROCESS, stale (S3) | none |
| kcolbchain/audit-checklist | github.com/kcolbchain/audit-checklist | `3d2869f` (clone) | 11 abstract Foundry check contracts (~1.1k lines) + 7 custom Slither detectors in Python | RESIDUE, the Slither-plugin pattern only (S5) | S5 R-S5-08 (custom Slither detectors as a project artifact). Heuristics REJECTED, see Rejected |
| zhongeric/solidity-audit-checklist | github.com/zhongeric/solidity-audit-checklist | `06ec936` (clone) | 1 file, 70 lines of personal C4 notes. "Stablecoins" is an empty heading | DUPLICATE, 1 residue (S4) | S4 R-S4-31 (lock keyed by owner address escaped by transferring the receipt) |
| OWASP Smart Contract Top 10 (2025) | owasp.org/www-project-smart-contract-top-10 | fetched 2026-09-09 | ten-item awareness list; 2025 promotes Access Control to #1 and Price Oracle Manipulation to #2 | DUPLICATE, zero orphans (S9) | S9 mapping table SC01..SC10; the only structural gaps are SC03 Logic Errors and SC04 Input Validation, which have no owning file in `checklists/01..20` |
| SlowMist Web3 Project Security Practice Requirements v0.1 | github.com/slowmist/Web3-Project-Security-Practice-Requirements | `5370f74` (clone) | operational standard, 220-line README, 6 sections: dev prep, dev process, release, runtime, emergency response, security awareness | RESIDUE, different in kind (S9) | S9 R-S9-37..R-S9-48 (front-end SRI, HTTP hardening, DNS/registrar, runtime monitoring, IR process and drills, forensics, postmortem, bug bounty, infra hardening, phishing training); contradictions 8, 9, 10 |
| SlowMist Knowledge-Base | github.com/slowmist/Knowledge-Base | `ce4cbdb` (clone, 917 MB / 260 files) | **not** a requirements doc: a published audit-report corpus (`open-report-V2/`: OKX, IoTeX, QuarkChain, OceanONE, mostly PDF/zh-cn) + MistTrack reports + 9 study notes + 3 mindmaps | DATASET / corpus only (S9) | no items; useful as a report-style reference |
| aviggiano/properties | github.com/aviggiano/properties | `2e9e851` (clone) | Fork of crytic/properties, same tree | DUPLICATE (S8) | none |

### DeFi and AMM specific

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| d-xo/weird-erc20 | github.com/d-xo/weird-erc20 | `781c8f0` (clone) | 22 minimal hostile ERC20 implementations + a README enumerating 26 weirdnesses | **RESIDUE**, 8 items our `16-token-integration.md` misses (S2, S4) | S4 R-S4-01 (native-currency ERC20 representation, the Arc-critical one), R-S4-02..R-S4-08, R-S4-33; S2 R-S2-01, R-S2-22 (Tether Gold returns false on success) |
| CDSecurity/audits | github.com/CDSecurity/audits | `f3275c4` (clone) | 6 checklists (`DEX_AMM` 259 lines, `Uniswap_V4_Hooks` 222, `DeFi_Lending` 257, `LRT_LSD` 231, `AAVE_V2_V3` 164, `Cross_Chain` 380) + `audit reports/` with 71 PDFs | DUPLICATE on checklists (0 residue from the AMM list); DATASET on the reports (S4) | S4 R-S4-22 (k-of-n signer-set rotation, from the cross-chain list), R-S4-34 (last LP out). The 71 PDFs are the valuable half |
| Zealynx Uniswap/AMM 45 patterns | zealynx.io/resources/checklists/evm/uniswap | fetched 2026-09-09 | 45 Solodit-mined finding patterns, title + severity + one line. No code, no greps | DUPLICATE 41/45 (S4) | S4 R-S4-16, R-S4-23 (ERC-3156 initiator), R-S4-24 (role renounce), R-S4-25 (fee custody deliverability) |
| Trail of Bits, "Building secure Uniswap v4 hooks" | blog.trailofbits.com/2026/07/30/building-secure-uniswap-v4-hooks/ | fetched 2026-09-09 | 7 recurring hook failure patterns + an 8-item builder checklist, mechanism-level | RESIDUE, 5/7 transfer to `YieldHookLib` (S4) | S4 R-S4-26..R-S4-30 |
| Dedaub, CavalRe Multiswap, 2023-07-10 | dedaub.com/audits/cavalre/cavalre-amm-jul-10-2023/ | fetched 2026-09-09 | 13 findings on a multi-asset AMM with a geometric-mean invariant, an index-scaled LP token, multi-in/multi-out `multiswap`, add/remove-asset | **RESIDUE**, closest public prior art to BTR's code shape (S4) | S4 R-S4-09 (H3 stamp-then-consume), R-S4-11 (H2 LP token on both sides), R-S4-14 (H1 double scaling), R-S4-15 (A4 one-sided bound), R-S4-16 (M1 delisting) |
| Dedaub, CavalRe AMM, 2023-08-18 | dedaub.com/audits/cavalre/cavalre-amm-aug-18-2023/ | fetched 2026-09-09 | 6 findings; the July H1 class reappeared in sibling functions | RESIDUE (S4) | S4 R-S4-10 (M3 mint-then-index), R-S4-33 (A1 >18 decimals); cited as the live example for LN-32 variant sweep |
| Dedaub, CavalRe AMM, 2023-10-16 | dedaub.com/audits/cavalre/cavalre-amm-oct-16-2023/ | fetched 2026-09-09 | 8 findings incl. one Critical pool drain via self-held LP tokens | RESIDUE (S4) | S4 R-S4-12 (P1 credit-by-delta), R-S4-13 (C1 self-held shares) |
| Platypus Finance, 3 incidents | immunefi.com/blog/.../platypus-finance-hack-analysis/ , blocksec.com/blog/5-platypus-finance-... , certik.com/blog/platypus-defi-incident-analysis | fetched 2026-09-09 | Coverage-ratio (`r = cash/liability`) stableswap. 2023-02 $8.5 to 9.05M, 2023-07 ~$50k, 2023-10 $2.2M | **RESIDUE**, same coverage family as BTR's `c = R/L` (S4, S10) | S4 R-S4-17 (coverage-ratio sandwich), R-S4-18 (cross-asset peg assumption), A-2 (kappa sized against max arbitrage profit); S10 R-S10-01, R-S10-02, R-S10-03 |
| Dedaub, Platypus root cause | dedaub.com/blog/platypus-finance-hack/ | fetched 2026-09-09 | The only writeup that states the mechanism: an under-cashed withdrawal decrements the full liability against a partial asset decrement, so `r` moves for free and `_slippage()` mis-prices both directions | **RESIDUE**, the single most valuable page in S4 | S4 R-S4-35 (enumerate every path where dR != dL), which is the generalisation R-S4-17 is one instance of |
| Platypus, "Withdrawal Arbitrage" (2021, self-published) | medium.com/platypus-finance/withdrawal-arbitrage-... | fetched 2026-09-09 | the protocol naming its own attack two years before it was run, and sizing the withdrawal fee exactly at the max attainable profit | RESIDUE (S4) | S4 A-2 amendment to CON-09 |
| Bancor v3 (omnipool + day-1 IL protection) | cointelegraph, protos, support.bancor.network, rosenlegal complaint (W.D. Tex.) | fetched 2026-09-09 | Omnipool with IL protection funded by minting BNT; suspended 2022-06-19 under a vault deficit; ~30% TVL lost; class action filed | **RESIDUE**, closest live precedent for BTR's par-exit promise + deficit + run (S4) | S4 R-S4-19 (deficit-response playbook), R-S4-20 (reflexive healing), R-S4-21 (latched claim during a drawdown) |
| Bancor 3 withdrawal-mechanics doc | support.bancor.network/bancor-amm/bancor-3-mechanics/...-2-withdrawal | fetched 2026-09-09 | prose only; the haircut/cooldown formulas are not on the page | **PARTIAL** (S4) | mechanism confirmed qualitatively (pro-rata deficit sharing); formulas not obtained from any source |
| DODO PMM | slowmist medium, DODO postmortem, PeckShield DODOV2, Sherlock 2023-06 + 2023-12, ToB V1 | fetched 2026-09-09 | Oracle-priced AMM with an inventory-regression term; targets recomputed per quote. One realized incident (2021-03-09, $3.8M, access control) + ~8 audit finding classes | **RESIDUE**, nearest oracle-priced archetype (S4) | S4 R-S4-39 (initializer reachability inside a callback), R-S4-41 (periphery with caller-supplied destination) |
| Hashflow RFQ | CertiK post-mortem, revoke.cash, Quantstamp 2023-06-16 | fetched 2026-09-09 | Off-chain signed quotes, one signer per pool. 2023-06 ~$640k on a deprecated out-of-scope contract + a second ~$40k harvest; 19 Quantstamp findings | **RESIDUE**, router/allowance-lifecycle class (S4) | S4 R-S4-40 (deprecation is not revocation), R-S4-41 |
| Wombat Exchange | Wombat docs + `Pool.sol`, PeckShield v1/v2/v4, SlowMist, Omniscia | fetched 2026-09-09 | `r = cash/liability`, invariant `D = sum L_i(r_i - A/r_i)`. Single-sided LP, per-asset balance sheets sharing one invariant. No public exploit of its own pools | **RESIDUE**, BTR's own family, highest-value block of the five (S4) | S4 R-S4-36 (pricing law running outside its stated precondition, `withdrawalAmountInEquilImpl` "should be used only when r* = 1"). **No audit in the set examined this** |
| Curve stableswap-ng | MixBytes 2023-11-01 (41 findings), MixBytes rate-oracle research, Vyper postmortem, ChainSecurity read-only reentrancy post-mortem | fetched 2026-09-09 | NG = dynamic fee + built-in EMA oracles + per-coin rate oracles + `nonreentrant('lock')` on `get_virtual_price()` and `totalSupply()` | RESIDUE + **CONTRADICTS the brief** (S4) | S4 R-S4-38 (our views are other protocols' oracle), A-1 (a mark push moves price with zero reserve change, so balance-based skew is blind to it). See Corrections |
| KyberSwap Elastic | 100proof post-mortem + REPORT.md, BlockSec, Kyber official, ChainSecurity re-audit, Sherlock #103 | fetched 2026-09-09 | Uni-V3 fork + a superimposed fee-reinvestment curve. 2023-11-23, $48.8M across six chains | **RESIDUE**, rounding-decides-a-branch class (S4) | S4 R-S4-37 (`calcReachAmount` vs `calcFinalPrice`: two arithmetic paths for one relation). Also the most expensive published example of skipping a variant sweep |
| balancer/balancer-v3-monorepo | github.com/balancer/balancer-v3-monorepo | `449f7e0` (clone) | `pkg/vault/test/foundry/fuzz/VaultNoFreeValue.medusa.sol` + `Swap.medusa.sol`: a running production medusa "no free value" suite | **RESIDUE**, strongest AMM hit in S8 | S8 R-S8-42 (latch "ever got worse"), R-S8-43 (sum-all-known-holders conservation), R-S8-44 (scope monotonicity to the action subset), R-S8-45 (zero-fee variant), R-S8-46 |
| morpho-org/morpho-blue | github.com/morpho-org/morpho-blue | `bc1bfca` (clone) | `test/invariant/DynamicInvariantTest.sol`, bounded oracle-price fuzzing | RESIDUE, oracle handler pattern (S8) | S8 R-S8-47 (clamp the fuzzed oracle price to a bounded per-push move) |
| euler-xyz/euler-vault-kit | github.com/euler-xyz/euler-vault-kit | `bfb325a` (clone) | post-hack invariant suite: `InvariantsSpec.t.sol` + `invariants/*` + `handlers/{modules,simulators,external}` | RESIDUE on structure, **CONTRADICTS its own reputation** (S8) | S8 R-S8-48 (file structure, the `simulators/` vs `modules/` split), R-S8-49. See Corrections |
| Uniswap/v4-core | github.com/Uniswap/v4-core | `46c6834` (clone) | **no invariant or fuzz campaign exists in the repo**; property tests are inline `test_fuzz_*` per function | **UNREACHABLE as an exemplar** (S8) | none; recorded so it is not cited as one |
| Vectorized/solady | github.com/Vectorized/solady | `2afba69` (clone, npm 0.1.26, 114 src files / 53,210 lines) | the library BTR uses instead of OZ. Substance is in the NatSpec headers, not the README | **RESIDUE** (S7) | S7 R-S7-07 (CI tests only solc <=0.8.30), R-S7-08 (`ReentrancyGuardTransient` off-mainnet degradation), R-S7-09, R-S7-18..R-S7-25; corrections 2 (our E20-12 is a trap that would false-positive on every BTR review) |
| Solady `audits/` (5 PDFs) | github.com/Vectorized/solady tree `audits/` | `2afba69` (clone) | Ackee, Cantina, Cantina-Spearbit-Coinbase, shung ERC721, xuwinnie cbrt proof | **NOT MINED** (PDF) (S7) | none; flagged as the most likely place to find a known issue in a file BTR uses |
| ethereum/solidity | github.com/ethereum/solidity | `deeab5a` (clone) | `docs/bugs.json` + `docs/bugs_by_version.json` (machine-readable known-bug list), `Changelog.md`, `docs/contracts/*.rst` | **RESIDUE + CONTRADICTS** (S3, S7) | S3 R-S3-01 and C-1/C-2; S7 R-S7-01 (linked-library struct drift), R-S7-02, R-S7-03, R-S7-04, R-S7-05, R-S7-06, R-S7-10..R-S7-13, R-S7-28; correction 3 (library bytecode diff) |
| OpenZeppelin/openzeppelin-upgrades | github.com/OpenZeppelin/openzeppelin-upgrades | `3aa87ac` (clone) | the validator: `packages/core/src/validate/report.ts` is the canonical upgrade-unsafe-construct list; `docs/.../writing-upgradeable.adoc` (473 lines) | RESIDUE (S7) | S7 R-S7-14 (linked external libraries rejected outright), R-S7-15, R-S7-16, R-S7-17 |
| EIP-1153 | eips.ethereum.org/EIPS/eip-1153 | fetched 2026-09-09 | Security Considerations + Rationale | RESIDUE (S7) | S7 R-S7-10, R-S7-12 |
| EIP-7702 | eips.ethereum.org/EIPS/eip-7702 | fetched 2026-09-09 | Security Considerations | **CONTRADICTS our AC-9** (S7) | S7 correction 1: there is no reliable on-chain EOA test after Pectra |
| ERC-3156 | eips.ethereum.org/EIPS/eip-3156 | fetched 2026-09-09 | flash-loan callback conventions | RESIDUE (S7) | S7 R-S7-26, R-S7-27, and the proposed `E3156-1..6` block; BTR ships flash loans and has zero ERC-3156 items today |
| RareSkills, delegatecall | rareskills.io/post/delegatecall | fetched 2026-09-09 | tutorial | RESIDUE, 1 item (S7) | S7 R-S7-28 |
| RareSkills, EVM storage layout | rareskills.io/post/evm-solidity-storage-layout | fetched 2026-09-09 | tutorial, part 1 of 2 | DUPLICATE (S7) | none; ST-1..15 already carry all of it |
| Solidity blog, transient storage (2024-01-26) | soliditylang.org/blog/2024/01/26/transient-storage/ | attempted 2026-09-09 | 301 from `blog.` to `www.`, then **HTTP 403** | **UNREACHABLE**, substituted (S7) | in-tree `docs/contracts/transient-storage.rst` at `deeab5a` carries the same guidance and is what was cited |
| Solady GHSA feed | `gh api /repos/Vectorized/solady/security-advisories` | fetched 2026-09-09 | 1 advisory: 2025-07-17 medium, missing `extcodesize` validation in `ERC4337Factory` | TOOL, mechanical check (S7) | S7 R-S7-06, R-S7-29 |
| OpenZeppelin Contracts GHSA feed | `gh api /repos/OpenZeppelin/openzeppelin-contracts/security-advisories` | fetched 2026-09-09 | 20 advisories, 2021 to 2025 | RESIDUE (2 post-5.0 entries we lack) + TOOL (S7) | S7 R-S7-29 |

### Tooling and MCP

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| Slither MCP (Trail of Bits) | github.com/trailofbits/slither-mcp | `1775573` (clone, 85 files) | MCP server exposing 23 tools over a cached Slither `project_facts.json`: call graph both directions, storage layout, dead code, filtered/paginated detectors | **TOOL, adopt** (ran it) (S1, S5) | S5 R-S5-10; args must be wrapped in a `request` object; add `--disable-metrics` |
| Aderyn (Cyfrin) | github.com/Cyfrin/aderyn | `de6a090` (clone, binary 0.6.8) | Rust static analyzer; ships an MCP server (`aderyn mcp stdio`) with 7 AST/callgraph tools | **TOOL, adopt MCP mode** (ran it) (S5) | S5 R-S5-11; `aderyn mcp` alone exits, the `stdio` subcommand is mandatory |
| Claudit (Solodit MCP) | github.com/marchev/claudit | `b9f0372` (clone, 10 files) | 740-line TS MCP server over `POST solodit.cyfrin.io/api/v1/solodit/findings`; 3 tools | **TOOL, adopt** (endpoint probed, not keyed) (S1, S5) | S5 R-S5-12, P-E (search Solodit before filing). Free key, 20 req/60 s. A 429 is a FAILED read, never "no sibling finding exists" |
| slvDev/weasel | github.com/slvDev/weasel | `092cad9` (clone, 269 files) | Rust parser-only analyzer, 196 detectors + MCP + 9 Claude skills. No compilation, 0.03 s | **TOOL, marginal** (built + ran) (S5) | S5 R-S5-13; only `solady-safetransfer`/`solmate-safetransfer` earn their place |
| `slither-mutate` | bundled with slither 0.11.6 | ran 2026-09-09 | mutation generator that runs your test command and reports uncaught mutants as unified diffs; 15 operators | **TOOL, highest value in S5** (ran it) (S5, S8) | S5 R-S5-01, P-A, P-B; S8 tools table. 41 mutants, 2 min, 14 mutations the sandbox suite does not detect |
| ToB `mewt` | github.com/trailofbits/mewt + blog.trailofbits.com/2026/04/01/mutation-testing-for-the-agentic-era/ | fetched 2026-09-09 | slither-mutate's successor, multi-language (Solidity **and Rust**), severity-tiered operators | **TOOL, adopt for the Rust mirror** (not run) (S8) | S8 tools table; the only mutation tool that can gate `core/src/pricing.rs` and the Solidity with one tool |
| Certora/gambit | github.com/Certora/gambit | `072ff4c` (clone) | mutant generator only, no runner; 53 mutants in 0.98 s on the sandbox; 10 operators incl. `elim-delegate` | TOOL, secondary (ran it) (S5, S8) | S5 R-S5-01 note. macOS solc path is `~/Library/Application Support/svm/`, not `~/.svm` |
| RareSkills/vertigo-rs | github.com/RareSkills/vertigo-rs | `94ec8bb` (clone) | mutation testing with a runner; needs `ast = true`; 13 mutants, 58 s, 5/13 killed | **REJECTED**, strictly worse than slither-mutate (ran it) (S5, S8) | only the `--sample-ratio` idea |
| agroce/universalmutator | github.com/agroce/universalmutator | `fec4223` (clone) | language-agnostic regex/Comby mutator; has `solidity.rules` but no test runner | **REJECTED** (S5, S8) | none |
| @openzeppelin/upgrades-core CLI | npm `@openzeppelin/upgrades-core` | ran 2026-09-09 | `validate out/build-info`: storage-layout + upgrade-safety diff between two versions, ERC-7201 and beacon aware | **TOOL, 2nd highest value in S5** (ran it) (S2, S5, S7) | S5 R-S5-02, R-S5-04. Caveat: BTR's linked external libraries are flagged unsafe by default and need `@custom:oz-upgrades-unsafe-allow delegatecall` |
| runtimeverification/kontrol | github.com/runtimeverification/kontrol | `98fdebb` (clone, 344 files) | KEVM + Foundry symbolic execution; project-local `lemmas.k` of `[simplification]` rewrite rules | TOOL, **unverified** (S5, S8) | S5 R-S5-14; S8 R-S8-39. Kontrol does not discharge the lemma itself: an unsound lemma silently corrupts every proof using it |
| Certora/CertoraProver | github.com/Certora/CertoraProver | `63fdea8` (clone, 7,231 files) | **GPL-3.0**, buildable locally; `CERTORAKEY` is required only for cloud runs | **CONTRADICTS our toolkit** (S5 C-3) | properties.md section 3 says "needs a licence key, unavailable". Licence claim verified from source; the local build was **not attempted** |
| Certora/Examples | github.com/Certora/Examples | `279e553` (clone, 494 files, 96 `.spec`) | CVL patterns: ghosts + `Sstore`/`Sload` hooks, parametric rules, `filtered`, `preserved`, `requireInvariant`, `satisfy` | RESIDUE, patterns not syntax (S8) | S8 R-S8-41 (parametric rules), R-S8-51 (hook-based ghosts catch writes a handler ghost structurally cannot), R-S8-52 |
| Certora/Tutorials | github.com/Certora/Tutorials | `ed1c140` (clone, 283 files) | `06.Lesson_ThinkingProperties/Categorizing_Properties.pdf`, the five-type taxonomy Recon adopts | RESIDUE (S8) | S8 R-S8-20 (Certora's five property types as the completeness checklist) |
| Certora/AIComposer | github.com/Certora/AIComposer | `c5f0d61` (clone, 359 files) | alpha: generate implementations from docs + CVL specs | **REJECTED**, generation not review (S5) | none |
| a16z/halmos | github.com/a16z/halmos | `079bb42` (clone, 219 files) | symbolic engine; `src/halmos/config.py` | **CONTRADICTS our plan** (S8 C-2) | `--uninterpreted-unknown-calls` is deprecated and a no-op and never covered internal pure functions. S8 R-S8-37, R-S8-38 |
| Recon-Fuzz/recon-docs | github.com/Recon-Fuzz/recon-docs | `173d80f` (clone, 22 md / ~4.4k lines) | the invariant-testing methodology book: Chimera framework, 4-part bootcamp, advanced tips, optimization mode, audit process | **RESIDUE + PROCESS, strongest single source in S8** | S8 R-S8-01..R-S8-14, R-S8-21..R-S8-26, R-S8-32..R-S8-34; process P1..P5 |
| Recon-Fuzz/chimera | github.com/Recon-Fuzz/chimera | `463c0d4` (clone as `recon-chimera`) | `Asserts` interface with `FoundryAsserts`/`CryticAsserts`/`HalmosAsserts` implementations: one suite, many tools | TOOL, adopt (S8) | S8 tools table |
| Recon-Fuzz/create-chimera-app | github.com/Recon-Fuzz/create-chimera-app | `3620e32` (clone as `chimera` **and** `recon-create-chimera-app`) | scaffolding: `Setup`/`TargetFunctions`/`AdminTargets`/`DoomsdayTargets`/`BeforeAfter`/`Properties`/`CryticTester`/`CryticToFoundry`; `symExec: true` | RESIDUE + TOOL (S5, S8) | S5 R-S5-06 (one harness four engines), R-S5-09 (Echidna symbolic execution via Bitwuzla) |
| Recon-Fuzz/setup-helpers | github.com/Recon-Fuzz/setup-helpers | `3e1cfb2` (clone ×2) | `ActorManager`, `AssetManager`, `Utils.checkError`, `Panic` lib, `MockERC20` | TOOL, adopt (S8) | S8 R-S8-07, R-S8-12, R-S8-13 |
| Recon-Fuzz/properties-table | github.com/Recon-Fuzz/properties-table | `619726c` (clone) | the `PROPERTIES.md` tracking-table format | PROCESS (S8) | S8 R-S8-26 (property tracking table) |
| crytic/properties | github.com/crytic/properties | `5ea14b8` (clone, 155 files, 168 properties) | ERC20/721/4626/ABDKMath64x64 property libraries + `PropertiesAsserts`/`PropertiesConstants`/`Hevm`/`User` harness kit + negative-control `Bad*` mutants | RESIDUE, harness kit and taxonomies not the properties (S8) | S8 R-S8-14..R-S8-19, R-S8-27, R-S8-28. `Trophies.md` lists only 2 real bugs |
| crytic/building-secure-contracts | github.com/crytic/building-secure-contracts | `2de122e` (clone, 220 files) | `program-analysis/echidna/**` + `program-analysis/manticore/**` | RESIDUE + PROCESS (S8) | S8 R-S8-16, R-S8-29 (end-to-end differential against a reference implementation), R-S8-32, P7. **`program-analysis/medusa/` is an empty directory at this commit** |
| crytic/fuzz-utils | github.com/crytic/fuzz-utils | `6f68fcc` (clone) | generates Foundry unit tests from an Echidna/Medusa corpus reproducer | RESIDUE, unverified (S5, S8) | S5 R-S5-07; S8 R-S8-35. Known gap: `bytes*`/`string` may decode wrong |
| perimetersec/fuzzlib | github.com/perimetersec/fuzzlib | `0908cff` (clone) | alternative to Chimera's asserts | TOOL, pick one, do not run both (S8) | none |
| perimetersec/evm-fuzzing-resources | github.com/perimetersec/evm-fuzzing-resources | `674de0f` (clone) | curated index, 148-line README | PROCESS / index (S8) | S8 R-S8-35, R-S8-36 (pointers to real campaign corpora) |
| perimetersec/public-fuzzing-campaigns-list | github.com/perimetersec/public-fuzzing-campaigns-list | `64eb4a8` (clone) | index of real published campaigns to lift properties from | PROCESS (S8) | S8 R-S8-36 |
| devdacian/solidity-fuzzing-comparison | github.com/devdacian/solidity-fuzzing-comparison | `125071f` (clone) | 14 head-to-head challenges: Foundry vs Echidna vs Medusa vs Halmos vs Certora, with verdicts | PROCESS, tool routing (S8) | S8 R-S8-31 (measured routing), R-S8-53 (guided configs are the default) |
| patrickd-/solidity-fuzzing-boilerplate | github.com/patrickd-/solidity-fuzzing-boilerplate | `ab7e5f0` (clone, 2023-12) | template for differential fuzzing of Solidity libs against non-EVM references over FFI | RESIDUE-adjacent, superseded (S5) | superseded by S5 R-S5-03, which was actually run |
| whackur/solidity-agent-toolkit | github.com/whackur/solidity-agent-toolkit | `202807c` (clone, 197 files, 0 stars) | MCP+LSP wrapper over slither/aderyn/solhint + an OWASP SCWE KB (156 entries) | **REJECTED** (probed) (S5) | none. "AST-based detection **with regex fallback**" means a clean result may mean the parser failed |
| Foundry MCP (third-party) | 0xClandestine/foundry-mcp-rs, PraneshASP/foundry-mcp-server, maxencerb/foundry-mcp | fetched 2026-09-09 | wrappers around forge/cast/anvil/chisel | **REJECTED** (S5) | none; no first-party Foundry MCP exists |
| ToB `trailmark` | github.com/trailofbits/skills `plugins/trailmark` | `d3323ce` (clone, ~9k lines, 13 skills) | tree-sitter code graph + pre-analysis (blast radius, taint, privilege boundaries), projects findings onto graph nodes, `graph-evolution` diff gates | TOOL, **unverified** on Solidity (S6, S9) | S6 R-S6-32/33/34; S9 R-S9-14, R-S9-15 |
| ToB `second-opinion` | github.com/trailofbits/skills `plugins/second-opinion` | `d3323ce` (clone, 918 lines) | shells out to Codex CLI / Antigravity CLI for a different-vendor model review; default is "both, recommended" | **TOOL + the citation for C-3** (S2, S6, S9) | S6 R-S6-35; S9 P-S9-01. This is the external evidence that changed our model-diversity line the same day |
| ToB `dimensional-analysis` | github.com/trailofbits/skills `plugins/dimensional-analysis` | `d3323ce` (clone, 1,904 lines) | annotate to propagate to validate pipeline over units/decimals with a real dimensional algebra, coverage manifest and a refuting validator | **RESIDUE, largest single-topic residue in S9** | S9 R-S9-02, R-S9-04, R-S9-05, R-S9-06, P-S9-06; S2 process residue. **Operationally dangerous**: writes into the project root and edits sources |
| ToB `fp-check` | github.com/trailofbits/skills `plugins/fp-check` | `d3323ce` (clone, 796 lines) | verify-a-claimed-bug workflow: 6 gates, 13 FP patterns, standard-vs-deep routing | ~85% DUPLICATE, thin RESIDUE (S6, S9) | S9 R-S9-23 (Gate 5 math bounds), P-S9-03/04/05. Take the text, skip the plugin |
| ToB `vulnerability-triage-brocards` (Woodruff) | vulnbrocards.com, via `trailofbits/skills` | `d3323ce` (clone) | 7 falsifiable dismissal tests with PASS / DISMISS / NEEDS-MORE-INFO per rule | RESIDUE, with **one rule rejected** (S2, S6) | S6 R-S6-08; S2 imports Brocard 4's nuance (voluntary strictness). Brocard 5 REJECTED, see below |
| Paradigm `evmbench` | github.com/paradigmxyz/evmbench | fetched 2026-09-09 | Paradigm x OpenAI benchmark + harness for LLM agents finding and exploiting contract bugs | TOOL / calibration, **unverified** (S9) | S9 P-S9-02 (benchmark our own recall). Eval spec lives in a `frontier-evals` submodule, not cloned |
| ScaBench / SCONE-bench / Majeur | scabench-org/scabench, safety-research/SCONE-bench, z0r0z/majeur | fetched 2026-09-09 | calibration corpora; Majeur is a 33-tool head-to-head on `Moloch.sol` with 14 novel findings total | DATASET / calibration (S1) | S1 C-4 is built on Majeur; the recommendation is one run of our own lens against ScaBench's curated set |
| paritytech/revive-differential-tests | github.com/paritytech/revive-differential-tests | fetched 2026-09-09 | declarative golden-corpus + two-backend differential framework (EVM vs PolkaVM) | PROCESS (S8) | closest published architecture to our Rust-mirror gate |
| Wake (Ackee) | github.com/Ackee-Blockchain/wake | not run | Python Solidity dev+test framework with detectors, printers and a cross-contract dataflow model | **UNVERIFIED** (S9) | undecided; the one question that decides it is compile survival under solc 0.8.36 + via_ir |
| 4naly3er | github.com/Picodes/4naly3er | not run | gas/QA report generator (contest-style G-/NC- findings), not a vulnerability detector | **REJECTED**, wrong target (S9) | none |
| Heimdall | github.com/Jon-Becker/heimdall-rs | not run | decompiler; already listed in `solidity-audit/references/tools.md:71` and never run | **UNVERIFIED**, provenance fallback only (S9) | use only when the `cast code` vs `forge inspect deployedBytecode` diff fails and someone must say what differs |
| shuvonsec `web3-solidity-audit-mcp` | inside `shuvonsec-web3-bb` | `bbce8a5` (clone) | wraps Slither + Aderyn + SWC behind an MCP server | **REJECTED** (S10) | none; adds a process and removes the flags |
| Echidna 2.3.3 | installed binary | ran 2026-09-09 | ran clean under via_ir; source-mapped coverage produced; `symExec: true` adds Bitwuzla | **CONTRADICTS our toolkit** (S5 C-1), **and S8 disagrees** | see Corrections and the inter-slice contradiction below |
| Medusa 1.5.1 | installed binary | ran 2026-09-09 | ran clean under via_ir, 81,556 calls/s, lcov + html; optimization mode is supported | **CONTRADICTS** our "not yet run" (S5 C-2, S8 C-1) | S8 R-S8-30 (config keys `testing.optimizationTesting.{enabled,testPrefixes}`) |

### Firm process and report anatomy

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| ToB, "Can you pass the Rekt Test?" | blog.trailofbits.com/2023/08/14/can-you-pass-the-rekt-test/ | fetched 2026-09-09 | 12 yes/no operational-security-maturity questions for a crypto org | PROCESS / RESIDUE + **CONTRADICTS the brief's attribution** (S6) | S6 R-S6-30 (all 12 verbatim), R-S6-31 (no numeric pass threshold). See Corrections |
| rekt-test.com | rekt-test.com | 2026-09-09 | canonical site for the Rekt Test | **UNREACHABLE** (DNS `getaddrinfo ENOTFOUND`) (S6) | none |
| ToB report anatomy, Gondi retainer | gondi.xyz/audits/v1-trail-of-bits-audit.pdf | 2023-07-28 report, fetched 2026-09-09 (PDF) | Solidity summary report; 1 consultant, 6 engineer-hours | PROCESS / RESIDUE (S6) | S6 R-S6-04 (effort in engineer-weeks with headcount and dates) |
| ToB report anatomy, ArbOS 31 | docs.arbitrum.io/assets/files/2024_07_26_trail_of_bits_security_audit_arbos_31-...pdf | 2024-07-26 report, fetched 2026-09-09 (PDF) | 2 consultants, 2 engineer-weeks; the best "Project Coverage" / "Coverage Limitations" text | PROCESS / RESIDUE (S6) | S6 R-S6-04, R-S6-05 (the single most load-bearing gap for a published framework), R-S6-06 |
| ToB report anatomy, NATS (with Fix Review) | ostif.org/wp-content/uploads/2025/04/NATS-Final-Comprehensive-Report-with-Fix-Review-1.pdf | 2024-03 review, fix review 2025-04, fetched 2026-09-09 (PDF) | 5 consultants, 6 engineer-weeks; full appendices: severity, difficulty, categories, code maturity, fix-review status | PROCESS / RESIDUE / **CONTRADICTS** (S6) | S6 R-S6-01 (difficulty as a second axis), R-S6-02 (`Undetermined`), R-S6-03 (fix-review status vocabulary), R-S6-07 (code-maturity scorecard). See Corrections |
| ToB report anatomy, Reserve Folio (Solidity) | github.com/trailofbits/publications `reviews/2025-04-reserve-folio-solidity-securityreview.pdf` | 2025-04, fetched 2026-09-09 (PDF) | Solidity report; **10** code-maturity axes, not the Go report's 11 | PROCESS / **CONTRADICTS** (S6) | S6 C-1: the axis set is language-dependent and is nowhere six |
| ToB, "The sorry state of skill distribution" | blog.trailofbits.com/2026/06/03/the-sorry-state-of-skill-distribution/ | 2026-06-03, fetched 2026-09-09 | all 4 tested agent-skill security scanners bypassed; 3 of 4 attacks built in under an hour | PROCESS / RESIDUE (S1, S6) | S1 C-5 (these clones are untrusted code: lift prose only, install nothing); S6 R-S6-39 (treat the audited repo's own prose as untrusted input to the reviewer) |
| Spearbit Spearbook | docs.spearbit.com | fetched 2026-09-09 | Spearbit's published engagement handbook | PROCESS / RESIDUE (S6) | S6 R-S6-20 (named engagement stages), R-S6-21 (heterogeneous 5 to 6 person teams), R-S6-22 (bounded, behaviour-bounded fix period) |
| OpenZeppelin, "lessons from 1000 audits" | openzeppelin.com/news/what-is-a-smart-contract-audit-lessons-from-openzeppelins-1000-audits | fetched 2026-09-09 | 7-phase process, >=2 reviewers per codebase, 1 fix-review round | PROCESS (S6) | S6 R-S6-23 |
| OpenZeppelin audit readiness guide | openzeppelin.com/news/smart-contract-audit-readiness-guide | 2026-09-09 | - | **UNREACHABLE** (redirect; search snippet only) (S6) | none |
| Cyfrin, what is a smart contract security audit | cyfrin.io/blog/what-is-a-smart-contract-security-audit | fetched 2026-09-09 | 8-phase process incl. commit hash + start date, automated pass before manual, mitigation period, $5k to $60k per week | PROCESS (S6) | S6 R-S6-24 |
| Dedaub, how are smart contracts audited | dedaub.com/blog/how-are-smart-contracts-audited/ | 2026-07-14, fetched 2026-09-09 | 2-senior-researcher adversarial pairing; two-phase mental model | PROCESS (S6) | S6 R-S6-25 (keep understanding and subversion in the same head), process item 12 |
| Dedaub audit docs | docs.dedaub.com/docs/audits/smartContractAudits/ | fetched 2026-09-09 | scope and limits of an audit; explicit maths-specification requirement | PROCESS / **CONTRADICTS the slice premise** (S6) | S6 R-S6-26 (an unstated formula cannot be checked by anyone); C-4 |
| Consensys Diligence, own site | consensys.io/diligence/ -> diligence.security/ | 2026-09-09 | - | **UNREACHABLE** (redirect loop then 403 on every path; archive.org also blocked) (S6) | none. The "no scope drift" rule the brief attributes to them was **not found in any primary source** |
| Consensys Diligence, published report appendix | github.com/ConsenSysDiligence `aztec-audit-report-2019-04` README Appendix 2 | fetched 2026-09-09 | Critical / Major / Medium / Minor severity definitions, verbatim | PROCESS / RESIDUE (S6) | S6 R-S6-37 |
| Sherlock judging guidelines | docs.sherlock.xyz/audits/judging/guidelines | fetched 2026-09-09 | severity thresholds, 3-part duplication rule, hierarchy of truth, escalation | PROCESS / RESIDUE + **CONTRADICTS** (S6) | S6 R-S6-11, R-S6-13, R-S6-14, R-S6-17; C-2 (hierarchy of truth is inverted vs our LN-45) |
| Code4rena judging criteria | docs.code4rena.com/awarding/judging-criteria , /competitions/severity-categorization | fetched 2026-09-09 | 2-tier H/M + QA, same-root-cause dedup, judge-vs-sponsor separation | PROCESS / RESIDUE (S6) | S6 R-S6-12, R-S6-13 |
| Cantina docs | docs.cantina.security `standards/severity/competition.md`, `standards/judging.md` | fetched 2026-09-09 | Impact x Likelihood, mandatory coded PoC for H/M, dup-decay scoring, escalation penalty | PROCESS / RESIDUE (S6) | S6 R-S6-15 (coded PoC mandatory), R-S6-16 (duplicate-decay scoring) |
| Immunefi Vulnerability Severity Classification v2.3 | immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/ | fetched 2026-09-09 | impact-worded 5-tier scale for smart contracts | PROCESS / RESIDUE (S6) | S6 R-S6-18 (the scale a non-specialist reader already knows; our severities must publish a mapping) |
| Immunefi PoC + primacy-of-impact article | immunefisupport.zendesk.com/... | 2026-09-09 | - | **UNREACHABLE** (403; the text used is a search-snippet paraphrase, marked as such) (S6) | S6 R-S6-19, marked paraphrase |
| Pashov `judging.md` + `senior-auditor-sop.md` | github.com/pashov/skills | `c577eb7` (clone) | 4 gates, numeric confidence scoring with published deductions, lead-promotion rules | RESIDUE (S1, S6) | S6 R-S6-09, R-S6-10; S1 P-3 (amplify-then-refute as separate roles) |
| ToB `code-maturity-assessor` | github.com/trailofbits/skills | `d3323ce` (clone) | 9 axes on a WEAK/MODERATE/SATISFACTORY ladder | PROCESS (S6) | S6 C-1; our `sources.md:8` is correct for the skill and is our only mention |
| ToB `audit-prep-assistant` | github.com/trailofbits/skills | `d3323ce` (clone) | Freeze Stable Version, detailed file list with out-of-scope marks, identify boilerplate | PROCESS / RESIDUE (S6) | S6 R-S6-38. **Our workbook has never pinned a commit for a round** |
| ToB `variant-analysis` | github.com/trailofbits/skills | `d3323ce` (clone) | variant sweep that reports the patterns that FAILED and ships a CI rule | RESIDUE (S6) | S6 R-S6-36 |
| ToB `differential-review` / `trailmark-review-gate` | github.com/trailofbits/skills | `d3323ce` (clone) | deterministic diff gate rules with numeric thresholds and `UNKNOWN > FAIL > WARN > PASS` precedence | RESIDUE (S6, S9) | S6 R-S6-34; S9 R-S9-15 |
| weAudit + SARIF 2.1.0 | ToB VS Code extension; SARIF spec | fetched 2026-09-09 | interchange formats carrying severity, **difficulty**, multi-location ranges, and an audit-**note** entry type | TOOL, adopt as an export format (S6) | S6 R-S6-32, R-S6-33 (report the unmatched-binding rate as a quality metric) |

### Empirical datasets and exploit corpora

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| SunWeb3Sec/DeFiHackLabs | github.com/SunWeb3Sec/DeFiHackLabs | `1cd4ba4` (clone) | 842 incidents indexed, 880 Foundry PoC `.sol`; **493 of 842 carry a curator root-cause label** | **RESIDUE + CONTRADICTS** (S10) | S10 R-S10-01/02/03 (the 3 Platypus PoCs), R-S10-05 (Nomad QSP-19); C-2. Use it for **mechanisms**, never for base rates |
| SunWeb3Sec/DeFiVulnLabs | github.com/SunWeb3Sec/DeFiVulnLabs | `f61f6ee` (clone exists) | pattern-demo repo | **DISPUTED**: S10 recorded it as UNREACHABLE (clone timed out twice) yet a clone exists at `f61f6ee`. Not read either way | none; see inter-slice contradictions |
| Code4rena community findings dataset | code4rena.com/community-resources/findings.csv | fetched 2026-09-09, 4.38 MB | 55,461 (warden x finding) rows, 394 contests, a `split` column giving duplicate counts | **DATASET, the headline empirical result** (S10) | S10 solo-find rate computed from it: 33.5% of Highs and 42.3% of Mediums found by exactly one of a median 39 wardens; plus the k-reviewer recall curve behind C-3 |
| 0237h/code4rena-stats | github.com/0237h/code4rena-stats | `bb0b823` (clone) | 3 notebooks documenting the `split`/`slice`/`pie` columns | TOOL (S10) | schema only |
| Web3Bugs (ICSE'23) | github.com/ZhangZhuoSJTU/Web3Bugs `results/bugs.csv` | raw CSV fetched 2026-09-09 (492 rows); **git clone timed out twice** | 492 labelled bugs from 167 C4/postmortem targets, labelled by whether detection needs a semantic oracle | PROCESS / CONTRADICTS (S10) | S10 recomputed 300/379 = **79.2%** of exploitable bugs need a high-level semantic oracle. Do not use the figure to argue the share-inflation class is covered (ICSE excludes it as label O3) |
| Zhou et al., *SoK: DeFi Attacks*, IEEE S&P 2023 | eprint.iacr.org/2022/1773.pdf | fetched 2026-09-09 (PDF) | 181 incidents, >=$3.24B, 2018-04 to 2022-04; separates a protocol-**design** layer from the code layer | CONTRADICTS / PROCESS (S10) | PRO layer = 73/181 = 40% of incidents, with tool coverage 6% vs 20% for SC; 103/181 non-atomic; **exactly 1 of 181 triggered an emergency pause within the first hour** -> R-S10-07 |
| DeFiLlama hacks dataset | api.llama.fi/hacks | fetched 2026-09-09 | **1,261 incidents, $20.64B**, per-incident `classification` + `technique`; $-weighted | **RESIDUE / DATASET** (S10) | S10 C-1 (Key Compromise = 12.1% of incidents but **41.4% of all dollars**), R-S10-04 (frontend/infra 95 / $1,037M; social engineering 60 / $3,113M) |
| ack3 H1-2026 DeFi Incident Dataset | arxiv.org/html/2608.13792 | 2026-08-13, fetched 2026-09-09 | 135 incidents, $939.86M, Jan to Jun 2026; measures exploit path vs the audit's scope boundary | **PROCESS, the single most important process number in S10** | **67.6% of exploit paths at audited victims were outside the audit scope**, 94.4% of that subset's dollars; median audit age at an in-scope incident 18 months -> R-S10-04, R-S10-05 |
| Beyer, *The Audit Gap in Blockchain Security* | arxiv.org/abs/2606.15465 | 2026-06-16, fetched 2026-09-09 | 23,818 audit findings, 22 firms, vs 218 rekt incidents | PROCESS (S10) | audit-finding mix vs loss mix |
| Landsman et al., SSRN 5198563 | papers.ssrn.com/sol3/papers.cfm?abstract_id=5198563 | 2025-26, fetched 2026-09-09 | 4,000+ protocols 2020 to 2025; audit-at-launch vs breach probability | CONTRADICTS (external) (S10) | **not statistically significant**; feeds C-4 (never accept "audited" as a compensating control) |
| Immunefi published disclosures / annual loss reports | immunefi.com/blog/research/ | fetched 2026-09-09 | 2022: infra/key handling = 46.5% of hacked funds; ecosystem-class losses 19% (2022) to <1% (2025) | DATASET, vendor self-reported: a **direction**, not a rate (S10) | corroborates C-1 |
| Rekt.news leaderboard | rekt.news/leaderboard | fetched 2026-09-09 | ~top-100 prose entries; **publishes no root-cause taxonomy and no aggregate split** | **UNREACHABLE as a dataset** (site is up, the data does not exist) (S10) | none. Anyone citing "Rekt says X% of hacks are Y" is inventing it |
| Code4rena / Sherlock / Cantina published solo-find rate | docs.code4rena.com/awarding , docs.sherlock.xyz/audits/watsons/first-submission-pot , docs.cantina.xyz | fetched 2026-09-09 | all three publish only a *payout formula* that decays with duplicate count | **UNREACHABLE / NOT PUBLISHED** (S10) | worked around by computing the rate from the raw C4 CSV. A payout formula is not an observed rate |
| Code4rena contest findings repos | github.com/code-423n4/{2024-10-loopfi,2024-10-ramses-exchange,2024-10-superposition,2024-11-ethena-labs}-findings | read via `gh api` 2026-09-09 | per-finding "Submitted by X, also found by Y" credit lines | DATASET (S6) | S6's 4-contest table (solo rate 0% to 67%), explicitly flagged as far too small to support a population claim. S10's 394-contest computation supersedes it |
| GPTScan | arXiv:2308.03314 (ICSE 2024, v3) | fetched 2026-09-09 | GPT + static-analysis confirmation for logic-vuln detection | DATASET, VERIFIED (S6) | S6 R-S6-40: raw LLM precision 57.14% on hard logic bugs, recall 83.33%; **the static confirmation step cut false positives by roughly two-thirds** |
| Self-consistency (Wang et al.) | arXiv:2203.11171 (ICLR 2023) | fetched 2026-09-09 | majority vote over sampled reasoning paths | DATASET, VERIFIED (S6) | supports the cohort argument, but it is same-model resampling, not model-family diversity |
| LLM-as-a-judge (Zheng et al.) | arXiv:2306.05685 (NeurIPS 2023) | fetched 2026-09-09 | GPT-4 judge vs human preference | DATASET, VERIFIED (S6) | >80% agreement, about equal to inter-human agreement |
| "Rating Roulette" | arXiv:2510.27106 | fetched 2026-09-09 | LLM-judge self-consistency by Krippendorff's alpha | DATASET, VERIFIED, **against** the cohort (S6) | alpha as low as 0.27 to 0.56. A judge that disagrees with itself cannot arbitrate |
| "Bias in the Loop" | arXiv:2604.16790 | 2026-04-18, fetched 2026-09-09 | prompt-framing bias swings in LLM judges | DATASET, VERIFIED, **against** (S6) | +31.6pp sentiment bias, -15.7pp verbosity bias; baseline self-consistency as low as 50.4% |
| Yu et al., security defects via code review | arXiv:2307.02326 | 2023-07-05, fetched 2026-09-09 | Cohen's kappa between two human raters, OpenStack/Qt | DATASET, VERIFIED (S6) | kappa = 0.87 on *labelling*, not on *discovery* |
| Stenberg, "Death by a thousand slops" | daniel.haxx.se/blog/2025/07/14/death-by-a-thousand-slops/ | 2025-07-14, fetched 2026-09-09 | curl's measured AI-slop rate in security submissions | DATASET, VERIFIED (S6) | ~20% of 2025 curl security submissions were AI slop, ~5% genuine. This is the reader's prior and we do not get to argue with it |
| LogicScan | arXiv:2602.03271 | 2026 | claimed successor to GPTScan | DATASET, SECONDARY, not fetched (S6) | none |
| LLM4Vuln | arXiv:2401.16185 | 2026-09-09 | eval framework | DATASET, SECONDARY, no headline numbers extracted (S6) | none |
| Code4rena Olas AI-agent contest | code4rena.com/audits/2026-01-olas | 2026-09-09 | AI agents in a live contest | DATASET, SECONDARY, snippet only, needs re-verification (S6) | none |
| RedVolt vendor benchmark | vendor page | 2026-09-09 | self-reported AI-auditor recall, no disclosed methodology | **REJECTED, MARKETING** (S6) | none |
| BeInCrypto audited-yet-hacked statistic | - | 2026-09-09 | claimed 88.44% of 2025+ hack losses at audited platforms | **REJECTED** (403 on fetch, snippet only) (S6) | none |
| cmichel Code4rena stats | cmichel.io/code4rena-first-1m-stats/ | fetched 2026-09-09 | 76 of 168 Highs solo = 45.2%, 97 contests, self-tallied, 2022 | DATASET, upward-biased (S10) | superseded by the 33.5% population figure |
| Inter-firm overlap ("two firms, same code, N% disjoint") | - | searched 2026-09-09 | - | **NOT FOUND** (S6) | one unsourced dev.to claim located and discarded. See Corrections |
| SmartBugs / SolidiFI / AuditGPT / SmartAuditFlow | - | - | not searched in this pass | **GAP, not an absence** (S6) | none |

### Prior-art audit reports (read as reports, not as checklists)

| source | url | commit / fetch | what it is | verdict | residue landed |
|---|---|---|---|---|---|
| MixBytes, Curve StableSwapNG, 2023-11-01 | github.com/mixbytes/audits_public/.../Curve Finance/StableSwapNG/README.md | fetched 2026-09-09 | 41 findings (3 Crit / 5 High / 12 Med / 21 Low) | RESIDUE (S4) | R-S4-38. **H1 acknowledged and NOT fixed in production**: external rate oracles are not whitelisted. Plus a biased `D` EMA and unlocked views, both acknowledged |
| MixBytes rate-oracle research | mixbytes.io/blog/safe-stableswap-ng-deployment-how-to-avoid-risks-from-volatile-oracles | fetched 2026-09-09 | a rate-oracle update moves the pool price with zero balance change; >2x the base fee extractable | RESIDUE (S4) | A-1 amendment to ORC-08. **This is BTR's mark push verbatim** |
| ChainSecurity, Curve LP oracle manipulation post-mortem | chainsecurity.com/blog/curve-lp-oracle-manipulation-post-mortem | fetched 2026-09-09 | read-only reentrancy as an export problem; >$100M at risk, dForce $3.65M and Sentiment ~$1M realized | RESIDUE (S4) | R-S4-38 |
| 100proof, KyberSwap Elastic post-mortem | 100proof.org/kyberswap-post-mortem.html | fetched 2026-09-09 | states the invariant, the precondition, the narrow fix and why the narrow fix failed, with wei-level numbers | RESIDUE, exceptional quality (S4) | R-S4-37 |
| Quantstamp, Hashflow 2023-06-16 | in `hashflow/evm/audits/` | fetched 2026-09-09 | 19 findings; HASH-1 High, HASH-19 fixed by deleting the function, HASH-3 Med **acknowledged** | RESIDUE (S4) | R-S4-40, R-S4-41 |
| PeckShield Wombat v1/v2/v4 | published reports | fetched 2026-09-09 | PVE-005 (a variant-sweep miss, in a report), v2 PVE-001 (haircut with the wrong token's decimals), v4 PVE-001 (skim ignores pending fees) | RESIDUE (S4) | corroborates AMM-04/05, GOV-10(a), P-01 |
| PeckShield / SlowMist / Sherlock DODO reports | published reports and contests | fetched 2026-09-09 | PVE-009 Critical (arbitrary callee + calldata), Sherlock #112/#132/#128/#164/#40/#70/#79 | RESIDUE (S4) | R-S4-39, R-S4-41; several marked **Won't Fix** |
| CDSecurity `audit reports/` (71 PDFs) | github.com/CDSecurity/audits | `f3275c4` (clone) | recent small-cap DeFi reports incl. `BeezieStableSwap-Audit.pdf`, `Euler_Audit.pdf`, `Hyperlend`, `MahaLend`, `Pear_Protocol` | **DATASET, not indexed, not mined** (S4) | none; only `BeezieStableSwap-Audit.pdf` is worth opening if a stableswap finding needs prior art |
| SlowMist `open-report-V2/` | github.com/slowmist/Knowledge-Base | `ce4cbdb` (clone) | published audit reports, mostly PDF and zh-cn | **DATASET, not mined** (S9) | none |

---

## Rejected, and why

Sources or specific rules we deliberately did NOT fold. This section matters as much as the registry.

| source or rule | why rejected |
|---|---|
| **devdacian/ai-auditor-primers** `base.primer.md` (`8e9e6a7`) | Closes with a "Friendship and Collaboration History" section and a greeting protocol instructing the model to maintain "a warm, friendly, and loving tone". Persona conditioning in an audit primer biases the model toward agreement, which is the opposite of what a refuter is for. The vulnerability lists are duplicates of checklists 01 to 16 and ~60% of the file is lending/liquidation, out of BTR scope. |
| **shuvonsec/web3-bug-bounty-hunting-ai-skills** frequency claims (`bbce8a5`) | "10 bug classes distilled from 2,749 Immunefi reports + 681 DeFiHack reproductions + 166 Nethermind reports" with **zero data files** in the repo (`find . -type f ! -name '*.md'` returns LICENSE and .gitignore). No label file, no methodology, no per-report mapping. The percentages do not reconcile: 28+19+17+12+8+3+2 = **89%** of Criticals across seven classes with three unaccounted, while "22% of Highs" sits in a different denominator in the same table. Immunefi does not publish per-class breakdowns, so the source data is not public and could not have been analysed by an outside party. The taxonomy is a reasonable ~90% duplicate; the frequencies are decoration. |
| **ToB triage Brocard 5** ("dismiss any report describing behaviour that is explicitly documented") | Contradicts our own rule that the code is the spec (`lenses.md` LN-45). Brocard 5 is written for triaging inbound reports against a third-party library whose docs are a published contract with downstream users. Here the docs (`content/docs/1. AIMM/*`) are ours, are written after the code, and are already known to lag it (oracle wire pages behind ticker22, deployments pages behind Arc). Documenting a hazard does not price it. Brocard 4's nuance was kept; Brocard 5 was not. |
| **austintgriffith `evm-audit-master` Phase 3** ("spawn one opus sub-agent per selected skill", 6 to 8 per audit, one checklist each) | Exactly the per-checklist allocation `toolkit/SKILL.md` section 0 forbids. The owner's core risk (undercoverage, contagion) is invisible to any single checklist and only appears when one lens owns a coupled-state group. |
| **austintgriffith `evm-audit-master` Phase 5** (`gh issue create` for every finding Medium and above) | On a private audit repo for a live testnet fleet this is an information-disclosure gesture, not a workflow. Must never be run for BTR. |
| **QuillShield severity-composition table** (missing guard x breaks invariant -> CRITICAL) | Its Type-3 invariant list names `k = reserveA x reserveB` explicitly. An inference engine run on an AMM that deliberately has no constant-product law will infer false invariants from the AIMM ledger and rate every intentional break HIGH. Adopt the inference method (R-S9-01) and the confidence metric; discard the severity table and keep our harm gate plus prerequisite tier. |
| **shuvonsec triage gate Q4** ("Admin can drain funds = centralization risk = KILL IT") | Correct for a bounty hunter maximising payout, exactly wrong for a pre-mainnet owner deciding what will take his money. It is the same trap S10's C-1 identifies in our own section 4. If folded, fold it with the warning attached. |
| **kcolbchain/audit-checklist heuristics** (`3d2869f`) | `OracleCheck.test_spot_price_manipulation` asserts a hardcoded ">10% price change in one block = vulnerable". For BTR, where the mark is externally pushed and deliberately frozen between pushes, that heuristic is **inverted**. `FlashLoanCheck` flags any function whose *name* contains "swap" or "liquidate"; the custom detectors are likewise name-based. Take the plugin scaffolding (R-S5-08), discard the heuristics. |
| **whackur/solidity-agent-toolkit** (`202807c`) | Advertises "AST-based vulnerability detection **with regex fallback**". A silent regex fallback behind an AST claim means a clean result may mean the parser failed: our own "a failed read is not an empty read" hazard wearing a tool's clothes. Also indirection over tools we run directly from Bash. |
| **Foundry MCP (three third-party wrappers)** | Wraps binaries we already call from Bash and spends context on tool schemas. No first-party Foundry MCP exists. |
| **vertigo-rs** (`94ec8bb`) | Ran it: 13 mutants and 58 s where slither-mutate made 41. Last commit 2024-09. Strictly worse. |
| **universalmutator** (`fec4223`) | Has `solidity.rules` but no test runner and no Solidity semantics. |
| **4naly3er** | A gas/QA report generator, not a vulnerability detector. Wrong target: our open work is economic and logic, not gas and QA. |
| **Certora AIComposer** (`c5f0d61`) | Generates implementations from docs and CVL specs. Generation, not review. |
| **SCSVS v2 requirement 14.12** | "Compute conversion price by multiplying numerator and denominator by the reserves, see `getInputPrice` in `UniswapExchange`". Correct for `x.y=k`, meaningless for an oracle-priced AMM. An agent applying it to `Pricing.sol` would file noise. |
| **CDSecurity `DEX_AMM_Checklist.md:82,241` and `DeFi_Lending_Checklist.md:172`** | Encode two assumptions BTR has deliberately rejected: that the invariant is `x.y=k`, and that a homegrown oracle is a smell. Both are correct advice for the median DEX and wrong for this one. The keeper-pushed k-of-n mark *is* the product; that trade-off already lives in the ASSUMPTIONS register (LN-11). |
| **RedVolt vendor benchmark** | Self-reported AI-auditor recall with no disclosed methodology. Do not present as neutral evidence. |
| **BeInCrypto "88.44% of 2025+ hack losses at audited platforms"** | 403 on fetch, snippet only, no primary source. Do not publish. |
| **Rekt.news as a dataset** | The leaderboard is per-incident prose with no root-cause field and no published aggregation. DeFiLlama's `api.llama.fi/hacks` is the correct substitute and is strictly better. |
| **"83% of exploits use a flash loan"** (shuvonsec `web3-bug-classes/SKILL.md:816`) | Unsourced, and DeFiLlama's technique labels do not support it at anything like that rate. Assume flash is enabled (LN-21 already does) and stop there; do not let the number drive prioritisation. |
| **Zealynx Security site** | Glossary, landing and "self-review" pages. No methodology artefact. The Viggiano material that is substantive (the eBTC retrospective, `scfuzzbench`) is not the agency's site. |
| **procur3.io Consensys statistics** (500+ audits / 5000+ findings / ~36% critical-high) | Third-party aggregator, not primary. Do not publish. |
| **"Code Inspector detects over 60% of low-severity issues"** (OpenZeppelin) | Search-snippet only, unverified. |
| **ToB `let-fate-decide`** (Tarot-based decision skill, in `trailofbits/skills`) | Ships with an explicit disclaimer against use as a deciding authority in safety-critical work. Recorded so it is not mistaken for an audit tool. |

---

## Unreachable

A failed fetch is not an empty source. Every one of these was attempted and failed; what was tried
and the failure mode are recorded so the next sweep does not silently treat them as absent.

| source | tried | failure mode | substitute used |
|---|---|---|---|
| `x.com/arsen_bt` researcher workflow thread (~2026-09-07) | WebFetch on x.com; searched for nitter, threadreaderapp and blog mirrors | **HTTP 402**, no mirror found | **None. Content deliberately NOT reconstructed** from the brief's own description, which would be citing ourselves. Account exists (Arsen, AdevarLabs, founder Defendor) |
| rekt-test.com | DNS resolve | `getaddrinfo ENOTFOUND` (domain dead) | the ToB blog post, which carries the 12 questions verbatim |
| Halborn, Platypus October 2023 | halborn.com/blog/post/explained-the-platypus-finance-hack-october-2023 | **HTTP 429** | mechanism recovered from BlockSec + CertiK, and root-caused from Dedaub |
| SharkTeam, Platypus principles | medium.com/@sharkteam/...-31a48481c952 | **HTTP 403** | as above |
| Consensys Diligence own site | consensys.io/diligence -> diligence.security, every path; archive.org | redirect loop then **HTTP 403**; archive.org blocked for the fetch tool | a 2019 report appendix on GitHub. The "no scope drift" rule the brief attributes to them was NOT found in any primary source |
| Immunefi help centre, PoC + primacy of impact | immunefisupport.zendesk.com | **HTTP 403** | a search snippet, explicitly marked as a paraphrase in R-S6-19 |
| OpenZeppelin audit readiness guide | openzeppelin.com/news/smart-contract-audit-readiness-guide | redirect, only a search snippet returned | the 1000-audits post |
| Solidity blog, transient storage (2024-01-26) | blog.soliditylang.org (301) -> www.soliditylang.org | **HTTP 403** | in-tree `docs/contracts/transient-storage.rst` at `deeab5a`, the same guidance maintained by the same team |
| `github.com/beirao/beirao.xyz` | git clone | `fatal: repository not found` (site is live, source repo is not public) | verbatim mirror at `cyfrin-audit-checklist/ref/beirao.md`, 555 lines, section counts match the live page 1:1 |
| entethalliance.github.io/eta-registry editor's draft | WebFetch | **HTTP 404** | the v3 release URL at `entethalliance.github.io/wg-ethtrust-site/spec/v3/` |
| solodit.cyfrin.io/checklist | WebFetch | SPA, content not server-rendered; only the shell returned | `Cyfrin/audit-checklist` `checklist.json`, which the repo readme states is the same data |
| `github.com/Hivework`, `Hivework/skills` | `api.github.com/users/{Hivework,hivework}`, `/repos/Hivework/skills`, WebSearch | **404** on all; nothing under that name exists | none |
| `crytic/slither-mcp` | git | wrong org | resolved: the repo is `trailofbits/slither-mcp` |
| `pashov/fizz` | git | not a standalone repo | resolved: lives at `pashov/skills/fizz` |
| `foundry-rs/foundry-mcp` | git | **404** | none; no first-party Foundry MCP exists |
| `OpenZeppelin/contracts-mcp` | git | **404** | resolved: OZ shipped *skills*, not an MCP |
| `0xMoi/claudit` | git | **404** | resolved: the repo is `marchev/claudit` |
| aviggiano benchmarking article | medium.com/@aviggiano | **HTTP 403** | none; listed so nobody quotes it as read |
| BeInCrypto audited-yet-hacked figure | WebFetch | **HTTP 403** | none; rejected rather than substituted |
| Rekt.news leaderboard as a dataset | rekt.news/leaderboard | site is up, the **data does not exist** (no root-cause field, no aggregation) | `api.llama.fi/hacks`, 1,261 labelled and $-weighted rows |
| C4 / Sherlock / Cantina published solo-find rate | docs.code4rena.com, docs.sherlock.xyz, docs.cantina.xyz | **not published by any of the three**; Cantina docs 301 -> 404 mid-migration | computed from the raw C4 `findings.csv` instead |
| Web3Bugs repository | `git clone` twice | timed out both times | the raw `results/bugs.csv` fetched directly, 492 rows |
| SunWeb3Sec/DeFiVulnLabs | `git clone` twice (S10) | timed out both times | none. **A clone nonetheless exists at `f61f6ee`**, so this record is disputed |
| `crytic/building-secure-contracts` `program-analysis/medusa/` | local read at `2de122e` | **empty directory in the repo** | medusa docs exist only at secure-contracts.com and in the medusa repo. Do not cite bsc-full for medusa behaviour |
| Uniswap/v4-core as an invariant exemplar | `find test -iname '*invariant*' -o -iname '*fuzz*'` at `46c6834` | **nothing**; only inline `test_fuzz_*` per function | none. Recorded so it is not cited as an exemplar |
| Wake / 4naly3er / Heimdall hands-on evaluation | a sub-task was dispatched to install and run all three | **did not return within the slice window** | nothing observed; those rows carry only what is documented, labelled NOT VERIFIED |
| Inter-firm overlap study | web search | **NOT FOUND**; one unsourced dev.to claim located and discarded | S10 computed a solo-find rate from C4 raw data instead |
| Bancor 3 withdrawal formulas | support.bancor.network withdrawal-mechanics page | page is prose only; the haircut and cooldown formulas are not on it | **none**; that block of S4 is qualitative by admission |
| Solady `audits/` (5 PDFs) | local, at `2afba69` | not attempted this round (PDF) | none; flagged rather than silently dropped |
| CDSecurity `audit reports/` (71 PDFs) | local, at `f3275c4` | not indexed, no search | none |
| SmartBugs / SolidiFI / AuditGPT / SmartAuditFlow | - | **not searched** | none. A gap, not an absence |
| LogicScan, LLM4Vuln, Olas contest figures | arXiv / code4rena | not fetched, or snippet only | none; marked SECONDARY, needs re-verification before use |

---

## Corrections this sweep made to received wisdom

Factual errors found in the slice briefs, in our own toolkit, and in common repetition. Each one is
checkable against the source cited.

1. **Trail of Bits code maturity is not six axes, and the axis set is language-dependent.** Three
   published sets exist, all from ToB, all on the same 8-value rating vocabulary: **Solidity reports
   carry 10 axes** (Reserve Folio 2025-04: Arithmetic, Auditing, Authentication/Access Controls,
   Complexity Management, Cryptography and Key Management, **Decentralization**, Documentation,
   **Low-Level Manipulation**, Testing and Verification, **Transaction Ordering**); Go/systems
   reports carry **11** (NATS); the `code-maturity-assessor` skill carries **9** on a different
   ladder. Nowhere is it six. For BTR use the Solidity set of 10.

2. **The Rekt Test provenance is commonly misreported.** The post is bylined **Dan Guido alone**;
   the others are named as conference *participants*, not co-authors. **Ribbit Capital** belongs in
   the participant list and was missing from the brief; **Solana Foundation does not appear at all**
   (it is associated with a separate, later 2025 push). The correct phrasing is "came out of a
   discussion led by Dan Guido of Trail of Bits with participants from Anchorage Digital, Euler
   Labs, Fireblocks, Immunefi and Ribbit Capital", not "co-authored with". Only question 2 has
   sub-bullets. `rekt-test.com` is DNS-dead.

3. **No published "two firms, same codebase, N% overlap" study exists.** Searched and NOT FOUND. One
   unsourced dev.to claim about an "OpenZeppelin / Trail of Bits public retro overlap" was located
   and discarded. What can be computed instead is the Code4rena solo-find rate: **33.5% of validated
   Highs and 42.3% of Mediums were credited to exactly one warden**, out of a median of 39
   reviewers on the same frozen commit (394 contests, 30,263 H/M rows, computed 2026-09-09). That is
   a different claim and must be stated as one.

4. **The "no scope drift once an audit has started" rule attributed to Consensys Diligence is not
   in any primary source.** Their own site is fully blocked (403 on `diligence.security`, redirect
   loop from `consensys.io/diligence`, archive.org blocked for the tool). Everything usable about
   their process comes from a 2019 report appendix on GitHub. Treat the rule as unverified.

5. **There is no ChainSecurity audit of Curve stableswap-ng.** Curve's own security index lists
   **MixBytes only (2023-11-01)**, and `curvefi/stableswap-ng` has no `audits/` directory.
   ChainSecurity audited Tricrypto-NG, crvUSD, PegKeeperV2 and scrvUSD. This matters because three
   of MixBytes' oracle findings are **acknowledged and unfixed in production**: unwhitelisted
   external rate oracles (H1), a deliberately biased `D` EMA, and remaining unlocked views. Anyone
   citing "stableswap-ng was audited" as reassurance is citing one firm's report with three open
   acknowledgements in it.

6. **One pack's banner arithmetic is wrong.** sanbir's skill banner and README say **328 attack
   vectors**; `attack-vectors-1.md:1` says **341 total** and 341 numbered entries exist across the
   five files. Cite 341 if we cite it at all.

7. **solc 0.8.36 is not clean.** `bugs_by_version.json` lists two open bugs at our pin:
   `MemoryByteArrayElementDeleteClearsWholeWord` (evmasm only, N/A under via_ir) and
   **`SpillSlotCollisionAcrossMutualRecursion`** (`viaIR: true`, introduced 0.7.2, fixed only in
   0.8.37). Our `aimm-checklist` BTR-03 note ("pinned `=0.8.36` here; keep the pin") was necessary
   and not sufficient. Verified and closed by V1-A (see below). Separately, **`bugs.json` is not
   complete by the project's own convention**: it carries only "Important Bugfixes", and the 0.8.37
   Changelog carries two more 0.8.36-live codegen bugs, one of which can delete a required overflow
   panic under via_ir.

8. **`abi.encodePacked` hash collision is not a compiler bug and has no version bound.** Our
   `20-version-compiler.md` SVI-4 lists it under a ">=0.8.17" column, inherited verbatim from
   Cyfrin. It is collidable in every Solidity version (SWC-133). Putting it in a table whose other
   rows are genuinely version-bounded reads as "pre-0.8.17 is safe".

9. **Solcurity, not tamjid0x01, is the primary source for three of our checklists.** Every
   `tamjid V*/F*/M*/C*/X*/E*/T*/P*/D*` citation in `checklists/17,18,19` is a Solcurity id, in the
   same order with the same text. tamjid0x01's repo is a copy. `references/sources.md:47` lists
   Solcurity as "Secondary", which is backwards.

10. **Solmate's `SafeTransferLib` has no code-existence check; Solady's does.** Our
    `05-erc-standards.md` E20-12 states the Solmate behaviour and we use **Solady**, whose
    `safeTransfer`/`safeTransferFrom`/`safeApprove` all revert on a codeless token. As written the
    line is a trap that would generate a false positive on every BTR review. The real Solady hazards
    are elsewhere: `safeTransferFrom2` silently falls back to Permit2 when `transferFrom` fails.

11. **Our own AR-1 / DP-1 prescribe an overflow-unsafe rewrite.** They offer `(a*c)/b` and
    `Math.mulDiv` as equivalent alternatives; they are not, since `(a*c)` overflows uint256 before
    the divide when either operand exceeds 2^128. Worse, `15-dacian-precision.md:6` gives as its
    *recommended* fix `Math.mulDiv(amount0 * token0Scale, 1e18, liquidity)`, where the
    pre-multiplication happens at uint256 before mulDiv's 512-bit intermediate can protect it. Our
    own prescribed remediation carries the hazard.

12. **EIP-7702 killed the EOA test.** Our AC-9 offers `tx.origin == msg.sender` or
    `code.length == 0`. A delegated account has code `0xef0100 || address`, so `code.length == 0` no
    longer implies EOA, and the EIP explicitly says it breaks atomic sandwich protections that rely
    on `tx.origin`. There is no reliable on-chain EOA test after Pectra.

13. **Certora is no longer licence-gated for local runs.** `Certora/CertoraProver` is GPL-3.0 with a
    documented local build; `CERTORAKEY` appears exactly once in the whole `scripts/` tree and is
    the cloud path. `properties.md` section 3's "needs a licence key, unavailable" row is wrong.
    (The build itself was not attempted.)

14. **halmos has no axiom mechanism for a user Solidity function.** `--uninterpreted-unknown-calls`
    is deprecated and a no-op, and only ever covered unknown *external* call selectors. Our
    "summarise `lnWad` by monotone-bound axioms (Certora-style ghost)" line names something that
    does not exist. The implementable technique is a two-sided sandwich, and it discharges only the
    direction lemma, not the magnitude lemma.

15. **Two reputations do not survive contact with the code.** Euler's post-hack invariant suite has
    **no economic-extraction property at all** (every `assert_` is share/asset bookkeeping) and its
    `LiquidationModuleInvariants.t.sol` is an **empty abstract contract with zero assertions**; its
    oracle handler fuzzes the price completely unclamped. Uniswap v4-core has **no invariant
    campaign in the public core repo at all**. Both are cited loosely as exemplars of economic
    invariant testing; neither is one.

16. **Recon's book is stale on medusa optimization mode.** It states optimization mode is
    Echidna-only; medusa's own config reference documents
    `testing.optimizationTesting.{enabled, testPrefixes}` and Balancer runs it in production. Here
    we were right and the source was wrong, but our line names neither the prefix nor the config key.

17. **The prerequisite tier under-rates the largest dollar-loss class in DeFi.** Our rule caps
    "compromised trusted key" at <= LOW. DeFiLlama's 1,261-incident dataset puts Key Compromise at
    152 incidents (12.1%) but **$8,551M, 41.4% of every dollar ever lost**, the largest single class
    by a factor of 2.7. Chainalysis 2024 has private-key compromise at 43.8% of all crypto stolen.
    mariano's own severity tree carries the carve-out we lack: centralization findings drop to
    MEDIUM "unless admin key is an EOA with no multisig", which is exactly BTR's Arc configuration
    (relay EOA == owner EOA).

18. **"Audited" is not evidence.** SoK's SEM model finds preventive defence (audit) at **p = 0.21**,
    no significant evidence it reduces harm, while reactive defence is p = 0.00. Landsman et al.
    (4,000+ protocols) find audit-at-launch has no statistically significant association with breach
    probability or loss. ack3 2026: **67.6% of exploit paths at audited victims were outside the
    audit's scope**, and the median audit age at an in-scope incident is 18 months.

19. **Acknowledged-and-unfixed is a status our registry did not have.** Curve H1, the biased `D`
    EMA, the unlocked views, DODO's Won't-Fix `adjustPrice` sandwich and min/max-answer bound, and
    Hashflow HASH-3 are five findings a reader would score as "audited" and that are live in
    production by choice. When our own ledger cites a peer protocol as precedent, cite the *status*,
    not just the finding.

20. **DeFiHackLabs' distribution is close to anti-correlated with BTR's exposure and must not be
    used as a prior.** It indexes what can be reproduced as one Foundry test against a forked block.
    Grepping all 842 index entries for `stale|keeper|sequencer|latency|liveness|heartbeat|
    oracle-push|signer|quorum` returns four hits, none of them an oracle-latency or keeper-liveness
    incident. Its #1 label at 21.3% is spot-AMM price manipulation, which BTR is structurally immune
    to. Anyone allocating BTR's eyes by "what got hacked most" would spend them all on the one class
    that cannot happen here.

### Where two slices disagree

Recorded rather than resolved, because both are defensible on the evidence each slice had.

- **Echidna under via_ir.** S5 C-1 **measured** it on 2026-09-09 in the sandbox (solc 0.8.36 +
  via_ir + optimizer 200): Echidna 2.3.3 ran to completion, 20,218 calls in 25 s, and **did** emit
  source-mapped coverage; Medusa 1.5.1 on the same source omitted the same two provably-executed
  lines *and* one more that Echidna caught. S5's conclusion is that our line
  ("via_ir breaks Echidna coverage maps, prefer medusa") is wrong in both halves, that the deflation
  is a property of via_ir and not of Echidna, and that the real reason to prefer Medusa is
  throughput (81,556 vs ~4,800 calls/s). S8 C-1, reasoning from documentation rather than a run,
  says our line is right. **S5 has the measurement; S8 does not.** Both slices agree on the second
  half (medusa optimization mode exists and our line should name the config keys).
- **DeFiVulnLabs reachability.** S10 records `SunWeb3Sec/DeFiVulnLabs` as UNREACHABLE after two
  clone timeouts. A clone nonetheless exists at `.corpus/repos/defivulnlabs` `f61f6ee`. Either the
  clone succeeded on a third attempt outside the slice, or another slice cloned it. It was not read
  by any slice either way.
- **create-chimera-app file count.** S5 says 31 files at `3620e32`; S8 says 26 files at the same
  SHA. Trivial, but it means at least one count was not taken from `git ls-files`.
- **slither-mcp confidence.** S1 rated it "probably" from the README; S5 ran it (tools/list plus 4
  real calls) and rated it "yes". Not a contradiction, a refinement: prefer S5's row.
- **Solodit MCP install.** S1 and S5 give the same recommendation but S5 additionally probed the
  endpoint (401 unkeyed, 405 on GET) and established the free-key and 20-req/60-s limits. Prefer S5.

---

## Measured tool results

Withheld. This section recorded what each tool returned when run against the protocol's own
source, including storage layouts, library link topology and one production incident. It is
implementation detail about unreleased code and is held under the policy in `DISCLOSURE.md`.

What it concluded, safely stated: of the static-analysis and compiler-bug leads this sweep
chased, every one was either refuted as unreachable or verified as not-applicable. None became
a finding. The sections above list the sources; that outcome is why the duplication figure
matters more than the tool list.

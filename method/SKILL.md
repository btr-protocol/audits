---
name: amm-audit
description: BTR AIMM audit orchestrator — lean multi-lens rounds to convergence on an oracle-priced multi-asset AMM. Economic risk first (undercoverage, contagion), then oracle, governance, code. Use for every BTR audit round.
---

# amm-audit

Distilled from the public-source sweep (counts, per-source duplication and verdicts:
`references/landscape.md`), plus `solidity-audit` (21 checklists) and `00-scope/PRIMER.md`.
Generic material is kept ONCE, here. Rule: **high reward, low effort** — agents are the expensive
resource; grep, forge and cast are cheap.

## 0. Token discipline (binding)

- **Allocate reviewers by shared mutable state, not by file.** One lens owns one coupled-state
  group (see `references/lenses.md` §1). Never one agent per file, per page, per module.
- **≤ 8 finders + ≤ 2 refuters per finding per round.** Refuters: 2 for CRITICAL/HIGH, 1 below.
- Agents load exactly: PRIMER + their lens section + the one checklist they own. No transcript
  history, no "for context" dumps. Output = structured JSON; the orchestrator writes the ledger.
  Every lens **enumerates its target set before analysing it** and marks each entry done or
  `N/A <reason>`; an unmarked entry is an unfinished pass, never a clean one.
- Deterministic work is a script, not an agent: entry-point census, live-state snapshot
  (`audit/scripts/live-state.py`), ledger merge, static-tool filtering, test baselines.
- Re-passes are **delta-only**: round N+1 re-reads only (a) sites with findings in round N,
  (b) files no lens logged in `EYES.md`, (c) every fix diff. A clean file is not re-read.
- **Carry the path, withhold the verdict.** A re-pass agent gets what round N *explored* ("traced
  the toll path; never opened the volatility path"), never what it *concluded*: a verdict in the
  prompt is re-confirmed, not re-tested. Brief adversarially — on an upheld finding, what adjacent
  bug does that analysis hide; on a refuted one, what enabler makes it live. "Safe, because X" on a
  new path is a complete pass. Inventing a finding to fill a round is not, and it breaks the stop.
- A workflow that would exceed ~25 agents is wrong; split by phase and read results between.

## 1. Loop

```
P0 understand  → PRIMER, CONCEPTS, INVARIANTS, ASSUMPTIONS, LIVE-STATE (once; refresh on change)
P1 align       → docs ↔ code, claims as INTENT-n → BEHAVIOR-n @ path:line (once; delta after fixes)
P2 hypothesise → 03-hypotheses/HYPOTHESES.md, one falsifiable line each, settle-by named
P3 round N     → find (lenses) → dedup (key) → refute → VERDICT.md + LEDGER.md + EYES.md
P4 fix         → owner ships; every fix diff gets a delta re-review (differential lens)
P5 converge    → baseline stop = TWO consecutive full rounds upholding zero findings at any
                 severity. Two further stop rules and the coverage-floor block: `METHODOLOGY.md §6`
```

Concept (economic/oracle/governance) rounds and code rounds run as two tracks inside one round;
the concept track leads because the owner's stated core risk is financial, not exploit.

## 2. Eyes budget

Score each entry point once in `00-scope/ENTRYPOINTS.md`: +1 value moves, +1 external call or
callback, +1 custom math or oracle read, +1 privileged/upgrade, +1 replaces a classical invariant
(swap, withdraw, flash, liability swap, haircut, toll). Eyes = independent lens passes that log
file:line in `EYES.md`:

| score | eyes | required lenses |
|---|---|---|
| 1–2 | **4** | accountant (LN-20), attacker (LN-21), verifier-hunter (LN-22), refuter (LN-30) |
| 3 | **6** | + oracle-skeptic (LN-11) *or* governor (LN-12), whichever owns the state · + gap-hunter (LN-23) |
| 4–5 | **8** | the six above + economist (LN-10) + a 2nd refuter |

Floor: **4 eyes per line in scope, 8 per hot path.** Hot paths, fixed so the tier is not re-argued:
`swap`/`batchSwap`, deposit/mint, withdraw/cross-withdraw/liability-swap, haircut, coverage toll,
spread+skew, spline traversal, oracle push/rebias/widen, `PoolIOLib.exec`, flash, every `Admin`
risk-param write. Score 1–2 carries the floor of 4 because "trivial" is one reader's classification.
first-principles (LN-13) is once per campaign and counts toward no tier; LN-31/LN-32 are triggered
by a fix or an upheld finding, not by score.

**Static analysis is not an eye.** Slither/aderyn/semgrep/wake are a precondition to the four; the
lens that triages their output is what counts (see `properties.md §1` triage rule).

Same model + prompt + session is one eye, not two (`METHODOLOGY.md §1`). Vary lens **and** model
family: a same-family multi-lens round is one methodology with ten remits. Every `EYES.md` row
carries the provenance fields of `METHODOLOGY.md §5.2` Rule M1, or it counts toward no stop rule.
Why these axes and in what order to spend them: `METHODOLOGY.md §2`.

## 3. Lens roster — names and ids only

Standing orders, remits and target sets: `references/lenses.md §2`.

Concept: **economist** LN-10 · **oracle-skeptic** LN-11 · **governor** LN-12 · **first-principles**
LN-13 (once per campaign, counts toward no eye tier).
Code: **accountant** LN-20 · **attacker** LN-21 (also owns the LN-07 call-frame axis) ·
**verifier-hunter** LN-22 · **gap-hunter** LN-23.
Triggered, not scored: **refuter** LN-30 (every finding, two refutation lenses) · **differential**
LN-31 (every fix diff) · **variant** LN-32 (every upheld finding, before its fix ships).

## 4. Gates

- **Quick veto** before any narrative: "what single check makes this impossible?" — grep it.
- **Prerequisite tier caps severity** by the hardest precondition: none → CRIT ok · victim
  signs/approves or specific market state → ≤ HIGH · one privileged actor's mistake → ≤ MEDIUM ·
  compromised trusted key → ≤ LOW unless authority propagates through an honest component or a
  fence fails (then re-tier).
- **Three axes, never collapsed**: `severity` (capped as above), `difficulty`, `blast_radius` ∈
  {leg, pool, fleet, treasury, LP principal}. The cap is right (without it a key leak makes
  everything CRITICAL) but alone it buries the biggest real loss class — key compromise is 41.4 % of
  every dollar lost (DeFiLlama, `landscape.md` Registry → *Empirical datasets*), and ORC-13 records
  the relay EOA **is** the owner EOA. A LOW with fleet radius must not sort to the bottom.
- **Severity order**: harm gate → composition floor → difficulty/blast → prerequisite cap LAST (the
  only step that lowers). Invariant break alone = HIGH only for invariants in
  `02-spec/INVARIANTS.md`; an *inferred* break is a LEAD.
- **Severity at the shipping configuration, never today's deployment.** A defect dormant only
  because a component has not shipped keeps its armed-state severity, with `live_today=false`.
  Dormant-today downgrades are banned; they under-tiered four rows in this campaign.
- **Promotion**: LEAD → FINDING only with (a) a reproduced number/trace/test, or (b) two lenses
  converging on the same `(contract, function, mechanism)`. Halmos/fuzz counterexample = LEAD
  until replayed in forge.
- **Material harm, named.** Who loses what, one line (an LP cohort, hub reserves, keeper liveness,
  an operator's exit). Harm stated as a mechanism only ("guard missing", "state diverges",
  "callable") is capped at INFO until a consequence is written, or re-filed as a design advisory.
- **Refute-first**: refuter defaults to `refuted`; verdict ∈ {TRUE, FALSE, DOWNGRADE, UPGRADE};
  the disagreeing party bears the burden with evidence; FALSE must name the compensating control.
- **Design advisory** is a first-class class: behaviour is intentional but its consequence is
  non-obvious or unbounded. Never dropped; reported to the owner as a design question.
- **Dedup key** = `(contract, function, state vars, invariant, fix shape)` — not similar prose. Two
  findings sharing a fix shape at one site are one row; a finding that *extends* an existing row is
  filed as an extension carrying its new information, never as a new row.
- **Variant sweep on every upheld finding** (`lenses.md` LN-32): one bug is a sample, not the
  population. Grep the family before the fix ships.
- **Compose before closing.** Refuted + INFO items go back once, pairwise: two individually
  unreachable preconditions are often reachable together (halt bit stranding an exit × a toll that
  grows with time; a dust push × the one-update-per-block lock). A read, not a round.
- **Fix batches are new attack surface** (round-2 lesson). LN-31 mandatory on every fix. A `require`
  added on a producer-controlled field (keeper/NXR wire) must first read the producer's contract and
  measure the field on chain (`cast logs`): an ordering the producer never promised is a
  self-inflicted liveness bug.
- **Fix sizing**: an incorrect fix is worse than none; every `fix:` names its shape (add-require /
  reorder / clamp / new state / control-flow / redesign) and what it could break.

## 5. Agent hygiene (from `PRIMER.md`, repeated because agents broke it)

Read-only on every tracked file. PoCs only under `dex-evm/test/scratch/<lens>/` (gitignored);
compile or delete before finishing. No git. Never print secrets. Cite `path:line` from `grep -n`.
A failed read is not an empty read: RPC 429 ⇒ FAILED, never a value.

## 6. Files

`references/routing.md` — **read first**: signal → what loads.
`references/lenses.md` — standing orders, coupled-state map, verification protocol.
`references/aimm-checklist.md` — protocol-native items (§A ledger/pricing · §B coverage/contagion ·
§C oracle · §D governance · §E routines · §F BTR constructs).
`references/properties.md` — property templates, tool pipeline (verified commands), formal bridge.
`references/landscape.md` — input registry: every external source with its SHA or fetch date, the
duplication measurement, and the rejected/unreachable lists. **Reference only: never loaded by a
cohort.**
`~/.claude/skills/solidity-audit/checklists/01..21` — generic Solidity; cite by file id.
`audit/METHODOLOGY.md` — campaign doctrine: independence axes, coverage floor, model rotation, stop
rules, publication obligations. This file runs a round; that one says why rounds are shaped so.

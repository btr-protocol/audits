# The BTR security framework

**v0.1 draft — 2026-09-09.** Operating manual for the BTR audit campaign, and the backbone of the
public write-up. Status of every section: `PRACTICE` = we already do this, `NEW` = added by this
revision, `GAP` = we do not do it yet and the campaign is weaker for it.

The one-line premise, and the reason this document is longer than a checklist:

> A protocol is exactly as secure as its least-secure reachable path. Coverage is therefore a
> **minimum over paths**, never a mean over rounds. A campaign that has run ninety rounds and never
> once looked at one hot function has a coverage of zero on that function, and the round count is
> not evidence about it.

Everything below exists to make that minimum measurable, and to make a *clean* result mean
something other than "we got tired".

---

## 0.1 What the free packs give away, and what audit firms actually charge for

By 2026 the *knowledge* layer of a smart-contract audit is free and commoditised. A dozen public
skill packs ship the same material: Pashov's twelve parallel hunters, QuillShield's ten semantic
plugins, Trail of Bits' plugin set, SolidityGuard's 104 patterns, sanbir's 210–328 attack vectors,
austintgriffith's 500+ checklist items, the Cyfrin/Solodit living checklist, OWASP SC Top 10,
SWC, weird-erc20, DeFiHackLabs' reproducible PoCs. Anyone can install all of it this afternoon for
nothing.

**The overlap is measured, not asserted.** The 2026-09-09 sweep read every source against our
existing toolkit and reported only the residue; the source count, the per-source duplication figures
and the verdicts are stated once, in `toolkit/references/landscape.md` (**Headline** section), and
are cited from here, never restated.

What none of them sell is the thing an audit firm's invoice is actually for. Spearbit does not
charge for knowing what reentrancy is; it charges for **four or five specialists reading the same
frozen commit with different brains, arguing about it, and someone senior arbitrating**. Strip the
knowledge out of an audit and what remains is three mechanisms:

⚠ **Novelty check, run against our own harvest 2026-09-09.** Only **M1 is absent from every pack
reviewed.** M2 exists in at least one orchestration pack (cross-vendor provider rotation per pass)
and M3 exists in several (dismissal-with-challenge, analyst-versus-validator loops, attacker/judge
separation) — our own harvest filed two of those as residue *we lacked*, which is the opposite of a
differentiator. The defensible distinction is **budget versus feature**: a pack rotates providers
when asked and judges the findings it happens to produce; neither is accountable to a per-line
floor, and none can answer "how many independent readers formed an opinion about this line, and was
that enough for what this line can cost". Claim the accountability, not the invention.

**M1 — Complexity-weighted re-reads (the 2nd, 3rd, 4th opinion).** Not "did a tool scan this line",
but *how many independent readers formed an opinion about it*, weighted by what the line can cost.
Every public pack runs N agents **once** across the whole scope: breadth, then done. None of them
encode a re-read budget per line. We do (§3.1; the budget itself is `toolkit/SKILL.md §2`), and the
campaign's reported coverage is the **minimum** over paths, naming the path that sets it.

**M2 — Model diversity.** The free packs are single-family by construction: a prompt bundle inherits
whatever the host model cannot see. We rotate families and treat a between-family disagreement as
the most informative event the campaign produces. Argument and rules: §5.

**M3 — Mandatory adversarial debate.** Public packs converge: agents find, an orchestrator dedups,
a judge scores. That pipeline has no one whose job is to *destroy* a finding, so its failure mode is
a confident false positive and, worse, a confident false negative that no one is paid to challenge.
Every finding here meets refuters that default to `refuted`, and a claim that survives no refuter is
never filed (gate: `toolkit/SKILL.md §4`; refuter budget: `SKILL.md §0`; standing orders:
`toolkit/references/lenses.md` LN-30). Debate is also the only device that keeps *re-reads* honest:
without it, opinions 2, 3 and 4 tend to ratify opinion 1.

Three secondary mechanisms come with the same invoice and are also absent from the free packs, so
they are specified here too: **scope and commit freeze** across all nine repos (§7.5), **the fix is
a new audit** — differential re-review plus a variant sweep of the whole bug family before any fix
ships (§7.6) — and **a residual-risk statement instead of a verdict** (§8).

Everything the packs *do* sell — the checklists, the patterns, the vectors — we ingest, dedup and
keep once, with provenance, in `toolkit/references/landscape.md`. That layer is table stakes. The
three multipliers are the product.

---

## 1. Vocabulary

| term | definition |
|---|---|
| **campaign** | the whole effort against one product, across surfaces and time. Current figures are generated, never typed: see §11, and regenerate before citing. |
| **surface** | a body of code and its state, audited as a unit. New code is a **new surface** and inherits none of the campaign's convergence. |
| **round** | one bounded execution: N finders, then refuters, then a merge into the ledger. Atomic unit of record. |
| **cohort** | one finder team inside a round: a lens (or lens pair), a scope regime, a priming state, and a model family. Recent rounds have run 6 cohorts each. |
| **lens** | a standing adversarial role with its own standing orders (`toolkit/references/lenses.md`, `LN-10..LN-32`). Lenses are allocated by **shared mutable state**, never by file. |
| **finder** | an agent whose job is to produce candidate findings under one lens. |
| **refuter** | an agent whose job is to destroy a candidate. Defaults to `refuted`; the disagreeing party carries the burden. |
| **eye** | one *independent* pass over a target that logs `path:line` in `EYES.md`. Same model + same prompt + same session is one eye, not two. |
| **finding** | a ledger row with a stable id, a severity, a `path:line`, and a named material harm. |
| **lead** | a candidate that has not yet met the promotion gate. Leads are recorded, never silently dropped. |
| **design advisory** | intentional behaviour whose consequence is non-obvious or unbounded. First-class output class; never downgraded into silence. |
| **clean round** | a round that filed zero rows **at any severity**, with every cohort's target set enumerated and marked done or `N/A <reason>`. |

---

## 2. Axes of independence

An eye counts only insofar as it is independent of the eyes already spent. Independence has four
orthogonal axes, and a campaign that varies only one of them is buying repetition at full price.

| axis | values | what varying it buys | status |
|---|---|---|---|
| **A1 lens** | the roster in `toolkit/SKILL.md §3` | different *questions*. The dominant axis: two lenses on one file beat two models on one lens. | `PRACTICE` |
| **A2 scope regime** | **free** (agent chooses its target) · **scoped** (assigned coupled-state group) | free measures *discovery*, scoped measures *coverage*. Neither alone is sufficient — see §4. | `PRACTICE` |
| **A3 priming** | **blind** (code + method only; ledger, TODO, spec, prior rounds withheld) · **primed** (full workbook) | blind measures whether a finding is *rediscoverable*; primed measures depth on known ground. Blind is the only defence against a ledger that teaches agents what to conclude. | `PRACTICE` (R11, R19–R21) |
| **A4 model family** | the rotation roster in §5.3 | different pretraining ⇒ **different false-negative sets**. This is the only axis that attacks blind spots shared by every prompt we can write. | `GAP` — see §5.2 |

**Rule I1.** A round must vary at least two axes across its cohorts. Six cohorts differing only in
lens is a good round; six cohorts differing only in model is a weak one.

**Rule I2 (carry the path, withhold the verdict).** Stated as a binding rule in
`toolkit/SKILL.md §0`. It belongs here because it is the single cheapest anti-correlation device we
have, and it applies across all four axes above — not only to re-passes.

---

## 3. The coverage floor `NEW`

The stop rules in §6 are about *findings*. This section is about *looking*, which is the thing the
premise actually cares about. Findings-based stopping is defeatable by not looking; coverage-based
stopping is not.

### 3.1 The object

**The operational table lives in `toolkit/SKILL.md §2`** — the entry-point scoring, score ⇒ required
eyes, the required lens per tier, and the fixed BTR hot-path list. It is stated there once and is
not reproduced here. This section states only why it is shaped that way, and the rules that bind a
campaign to it.

**Why a floor rather than a mean.** A "trivial" function is a classification made by one reader, and
the premise above is that the campaign is bounded by the line nobody looked at hard — so the low
tier carries a floor of four, not two as the public packs set, and the misclassification cost is
paid up front. This is the first of the three multipliers in §0.1 that the free packs do not sell.

**Rule C1.** The campaign's coverage is `min over entry points of (eyes spent / eyes required)`.
Report that minimum and *name the entry point that sets it*. A campaign never reports a mean.

**Rule C2.** A hot path (score ≥ 4) below its eye budget blocks the stop rule outright, regardless
of how many consecutive clean rounds have run. Findings-convergence on the covered part of a
system says nothing about the uncovered part.

**Rule C3 `NEW`.** The eye budget is per **(entry point × lens)** cell, and cells are also tracked
per **model family** for score-5 paths. A score-5 path signed off by one model family alone is
recorded as `single-family` in the residual-risk statement (§8) — not as covered.

### 3.2 The evidence

`EYES.md` is the coverage record: one row per pass, with the `path:line` ranges it actually cited.
A pass that cites nothing did not happen. `rebuild-report.py` already asserts that every ledger row
is placed exactly once (§7 of the report); the equivalent assertion for coverage is:

> every entry point in `ENTRYPOINTS.md` has ≥ its required eyes in `EYES.md`, or is listed by name
> in the residual-risk statement with the deficit.

**Rule C4.** Enumerate before analysing (`toolkit/SKILL.md §0`): an unmarked entry in a cohort's
target set is an unfinished pass, never a clean one, and a `clean round` (§1) requires every cohort's
set fully marked.

---

## 4. Scope regimes, and why both are mandatory `PRACTICE`, formalised

**Scoped** cohorts are assigned a coupled-state group. They are how the coverage floor is filled.
Their weakness is that they can only find what the scoping author thought to scope; a scoped
campaign inherits the blind spots of its own scope list.

**Free** cohorts choose their own target ("find the sharpest thing in this system"). They are how
the scope list itself gets audited. Their weakness is drift: they gravitate to the interesting and
re-derive the same three known mechanisms, which is why free rounds R59–R81 produced long clean
streaks that mean less than they look.

**Rule S1.** A model family is not retired (§6.3) until it has run **both** regimes clean. A free
streak with no scoped passes is a measurement of boredom.

**Rule S2.** Every free cohort declares its chosen target *before* it reports, and that target is
logged in `EYES.md` like any other. Free scope is not unlogged scope.

**Rule S3.** When a free cohort keeps landing on ground the ledger already owns, that is a signal
about the *ledger's visibility*, not about the code: re-run it blind (A3) before concluding the
system is quiet.

---

## 5. Model-family cohorts `NEW`

### 5.1 Why

Prompt engineering removes the blind spots we can *name*. Model diversity is the only instrument
that attacks the blind spots we cannot: a class of bug that a given pretraining distribution simply
does not represent well is invisible to every prompt we write for that model, at any temperature,
in any number of rounds. Independent-auditor practice (Spearbit, Pashov private, Sigma Prime) rests
on the same claim with humans — several people, same code, more than once — and the empirical
backing is contest data, where a large fraction of valid findings are submitted by exactly one
participant.

Model diversity is the **weakest** of the four axes per unit cost (§2, A4 buys less than A1) and
the **only** one that covers the unnamed. Both are true; spend accordingly — vary lens first, then
regime and priming, then family.

### 5.2 The provenance gap, stated honestly `GAP`

The workbook does not currently record which model produced which pass. Rounds 0–85 are attributed
by lens and cohort, not by family. Consequences:

- We cannot today substantiate a published claim of the form "N model families cross-validated this
  code" for the historical rounds. We can substantiate "N independent cohorts under M lenses".
- The per-family stop rule (§6.3) starts from the first round that carries provenance, not from R0.

**Rule M1 (binding from the next round).** Every row in `EYES.md` and every finding record carries:
`model_family`, `model_version`, `date`, `lens`, `scope_regime` ∈ {free, scoped},
`priming` ∈ {blind, primed}, `target_set_hash`. A pass without provenance is not counted toward any
stop rule.

### 5.3 Rotation

Roster as of 2026-09-09: **Fable 5.1 · Opus 5 · GPT Astra · Grok 4.6 · GLM 5.3 · Muse Spark 1.3**.
The roster is a living list; a family that ships a major version is a *new* family for stop-rule
purposes (its false-negative set moved).

**Rule M2.** Round composition draws cohorts from at least two families whenever more than one is
available, and the two must not both be scoped-primed — pair a scoped cohort in family X with a
free-blind cohort in family Y, so A2/A3/A4 vary together.

**Rule M3.** A finding confirmed by two families is `cross-family confirmed` and skips one refuter
tier. A finding found by one family and *refuted* by another is escalated, not dropped: a
disagreement between families is the most informative event the campaign produces, and it goes in
the dissent register (§7.4) whichever way it settles.

---

## 6. Stop rules — three, and what each one proves

Three distinct rules, deliberately not merged. Each proves something narrow; the campaign stops
only when all three hold **and** the coverage floor (§3) is satisfied.

### 6.1 Baseline stop `PRACTICE`
**Two consecutive full rounds filing zero rows at any severity, on a frozen surface.**
Proves: the audited baseline is quiet under the lenses we have.
Does not prove: anything about surfaces added since the freeze. Current status: **NOT MET**.

### 6.2 Surface reset `PRACTICE`
**Any new or materially changed surface resets the baseline stop for every path it touches**, and
enters the coverage floor with zero eyes.
Justification, from this campaign: the only new CRITICAL of the whole campaign was found by the
*first* adversarial pass on a surface that did not exist when the campaign began — while the baseline was in the middle
of a seven-round streak with zero MEDIUM-or-above. Both facts were true simultaneously. A stop rule
that cannot express that is a stop rule that ships the CRITICAL.

### 6.3 Per-family exhaustion `NEW` — the owner's rule
**A model family is retired from the rotation for a surface after six consecutive rounds — spanning
both scope regimes (§4) and at least one blind round — in which that family filed zero *new* rows.**

- *New* means new after dedup (§7.2). Re-deriving a known mechanism with a better number is a
  **transvalidation**, which is a valuable output and an explicit non-reset: it is evidence the
  ledger is right, not evidence the family is still finding things.
- Retirement is **per surface**, not global. §6.2 un-retires every family on a changed surface.
- Retirement is **reversible on evidence**: if another family later files a row on that surface, the
  retired family is recalled for one round scoped to the same coupled-state group. If it misses the
  row a second family found, that miss is recorded as a *family blind spot* — the most valuable
  calibration datum the framework produces, and the thing that tells us which family to weight on
  the next protocol.
- Six is a budget, not a proof. See §8.

### 6.4 What none of them prove
No combination of these rules proves absence of bugs. They bound *effort under a stated method*.
The honest output of the framework is §8, not a pass/fail.

---

## 7. The pipeline: raw agent output → published report

Deterministic wherever it can be. Agents produce structured JSON; humans and orchestrators never
hand-edit the record. `PRACTICE`, with the stages named.

### 7.1 Find
Cohorts run under the token discipline and the find-time gates in `toolkit/SKILL.md §0` and `§4`
(what an agent may load, deterministic work as a script, quick veto, material harm, citation or it
did not happen). Output is structured JSON; the orchestrator, never the agent, writes the record.

### 7.2 Dedup
Key and extension rule: `toolkit/SKILL.md §4` (**Dedup key**). Obligation 5 in §10 is the standing
argument for adopting the industry's cause-shaped test alongside it.

### 7.3 Refute and tier
Refute-first, the refuter budget, the two refutation lenses, the prerequisite-tier cap, the three
uncollapsed axes, the severity ordering and the shipping-configuration rule are all stated once, in
`toolkit/SKILL.md §0` and `§4`; the refuter's own standing orders, including the math-bounds gate,
are `toolkit/references/lenses.md` LN-30 and LN-41.

Why the stage exists at all, and why the cap is applied last: severity that ignores preconditions
prices every finding at the worst key leak, while a cap applied alone buries the largest real loss
class. Publishing difficulty and blast radius beside a capped severity is the only shape that
survives both objections — and it is the shape obligation 3 in §10 must be reconciled to.

### 7.4 Reconcile
- **Transvalidation**: a blind cohort re-deriving a known mechanism with independent numbers is
  recorded as a confirmation of the ledger row, with the numbers.
- **Dissent register**: any unresolved disagreement between cohorts, families or refuters is written
  down with both positions and who bore the burden. R11 carries one, and the ledger row held.
- **Corrections register**: numbers withdrawn, claims retracted, reviewer errors — kept in the
  report as a first-class section (§6 of the current report holds four, including a
  cross-validation claim that was itself wrong). A framework without a corrections section is
  asserting an accuracy it has not measured.

### 7.5 Consolidate
`LEDGER.md` is the source of truth. `REPORT-CONSOLIDATED.md` is a *generated view* over it
(`scripts/rebuild-report.py`) carrying a coverage assertion: every ledger id appears in exactly one
of an open theme, the dedup map, or the closed register — at its last generation 442 rows, 0 unplaced,
0 double-placed. Editing the prose without editing the ledger desynchronises them; the generator is
what makes "we did not drop a finding" checkable rather than claimed.

### 7.6 Fix, then re-attack the fix
Gates: `toolkit/SKILL.md §4` (**Fix batches are new attack surface**, **Fix sizing**); procedures:
`toolkit/references/lenses.md` LN-31 (differential) and LN-32 (variant sweep).
Why the stage is mandatory rather than advisory: this campaign's round-2 fix batch turned every
rebias into a potential permanent brick, and round 3's `executeFeedWiden` reintroduced a bug *with a
shipped test pinning it as intended*. **An incorrect fix is worse than none.**

### 7.7 Compose before closing
`toolkit/SKILL.md §4` (**Compose before closing**). The pairs are already written down, so the stage
costs a read, not a round.

---

## 8. The output: a residual-risk statement, not a verdict

The framework never emits "secure". It emits, and the published report must carry:

1. **Coverage floor** (§3.1): the minimum, and the entry point that sets it.
2. **Single-family paths**: every score-5 path signed off by one model family only.
3. **Assumptions nothing enforces**: this campaign lists 14 in `02-spec/ALIGN-*.md`, of which three
   are violated on the live testnet today. An assumption register with live violations is more
   useful to a reader than any severity histogram.
4. **Designed, priced, accepted risks**: for BTR, undercoverage is a *designed* persistent state
   whose only healers are exits, deposits, donations and net inflow. That belongs in the risk
   statement, not in the findings list.
5. **What formal methods reached**: measured, not aspirational — the per-property-class status
   table in `toolkit/references/properties.md §3`, quoted as measured, never summarised upward.
6. **Open findings by severity, with the fix status of each.**
7. **The corrections register** (§7.4).
8. **Effort, in units a reader can price**: rounds, cohorts, agent-passes, token cost, wall time,
   and the retired/active model roster.

---

## 9. Anti-gaming register

Ways a clean round can be manufactured. Each one has cost us something; each is now a rule.

| # | failure | rule |
|---|---|---|
| G1 | Re-running the same lens, model and prompt and counting it twice | Same model + same prompt + same session = one eye (§1). |
| G2 | Priming an agent with a verdict so it re-confirms | Carry the path, withhold the verdict (`toolkit/SKILL.md §0`; Rule I2 above). |
| G3 | Per-file fan-out to inflate the agent count | Allocate by shared mutable state. 110 agents / 6 M tokens was tried once, hit the session limit, and is banned. |
| G4 | Inventing a finding to make a round look diligent | Exploring a new path and reporting "safe, because X" is a complete pass. Fabrication is what breaks the stop rule, in the direction that hurts. |
| G5 | Downgrading a defect because the component has not shipped | Severity at the shipping configuration (`toolkit/SKILL.md §4`). |
| G6 | Grepping for the success string and calling the run clean | A filter for the expected output swallows the error that says the run never happened. Check exit codes, not patterns. |
| G7 | Treating a failed read as an empty read | RPC 429 / build failure / timeout ⇒ `FAILED`, never a value. PoC-runner semantics: `toolkit/references/lenses.md` LN-44. Agent-side rule: `toolkit/SKILL.md §5`. |
| G8 | Counting a static-tool run as a lens | Static analysis is not an eye; it is a precondition to the four, and the lens that triages it is what counts (`toolkit/SKILL.md §2`). The triage rule itself: `toolkit/references/properties.md §1`. |
| G12 `NEW` | **A tool's exit code taken as evidence the tool ran** — G6 and G7 generalised from a read to a whole tool. | The coverage-tripwire rule, with the measured case and what each row must assert: `toolkit/references/properties.md §1` (**Coverage tripwire**). |
| G9 | A property suite that cannot fail | The mutant gate and the vacuity gate: `toolkit/references/properties.md §2`, P-14 and P-26. No green property is citable until both pass. |
| G10 | Free-scope drift onto known ground | Rule S3: re-run blind before concluding the system is quiet. |
| G11 | A clean round on a shrinking scope | The stop rule reads the *shipping* surface. R40 verified two mandated allowlists **absent** — a clean round describing a system smaller than the one being launched. |

---

## 10. Publication obligations — what a professional reader will demand

Thirteen things a firm's report answers and an agentic campaign is assumed to be dodging until it
answers them. Each row states our honest position today. `✓` = we already satisfy it, `~` = partial,
`✗` = we do not, and it must be fixed before publication.

| # | obligation | our position |
|---|---|---|
| 1 | **Scope freeze and commit pinning** | `~` Nine repo heads are recorded at freeze, but `dex-evm` is recorded as "working tree dirty, audited at HEAD". A dirty tree is not a pinned commit and no reader will accept it. Fix: pin a hash per repo, no dirty trees, and define whether a mid-campaign move is a new round, a delta round or a stop. |
| 2 | **Scope in, scope out, both listed** | `✗` We publish no exclusion list. SCSVS makes it mandatory; ToB publishes it as *Coverage Limitations*. Needs the file list, the out-of-scope list, and the marked-boilerplate list (Solady vs OpenZeppelin vs ours). |
| 3 | **Severity rubric with an external mapping** | `~` Our tiers exist and the **prerequisite-tier cap** (§7.3) is our own invention: it must be shown to reduce to the standard axes rather than replace them. Adopt **Difficulty as a published second axis** (ToB) and **`Undetermined`** as a first-class verdict, then map to Immunefi v2.3 and the contest platforms' H/M thresholds. |
| 4 | **Who arbitrates, under what standard** | `✗` A named human with the final call, distinct from both the finding and refuting agents; an overturn standard of "clearly wrong", not "I disagree"; a time-boxed escalation window with a cost, so refuted items do not re-litigate every round. **An agent cohort with no human arbiter is not an audit, it is a search.** |
| 5 | **Dedup stated as a test** | `~` Ours is location-shaped, `(contract, function, state vars, invariant, fix shape)`. The industry rule is cause-shaped: same root cause, at least Medium impact, and a valid attack path, with the operational check *does fixing the root cause eliminate it*. Adopt both. And state explicitly that **cross-family agreement is evidence, never a second finding** — otherwise a multi-model cohort inflates its own count, which is the first thing a hostile reader will check. |
| 6 | **False-positive rate, measured** | `~` We have 85 rounds of data and have never computed it. Publish: candidates raised, candidates surviving refutation, findings surviving owner arbitration, findings that reached a reproduced PoC. Per model family. The reader's prior on LLM security output is bad and deservedly so; a framework that will not publish its precision is asking for faith. |
| 7 | **Coverage evidence, positive and negative** | `~` `EYES.md` is positive-only. Needs, per entry point: which lenses logged lines, which returned `N/A` with a reason, and which were **never reached**. Passed controls as well as failed ones. |
| 8 | **What the stop rule proves, and does not** | `✓` §6.4, sharpened: it proves that two consecutive rounds over prior-finding sites, files never logged in `EYES.md`, and every fix diff produced nothing that survived refutation, at the pinned commit, with the declared lens roster and model cohort. It is a **budget-exhaustion criterion with a falsifiability condition attached**, and must be published in those words. It proves nothing about the untouched majority of the tree, which the delta-only rule explicitly did not re-read. |
| 9 | **Reproducibility** | `✗` Publish model ids and versions, skill-file hashes, effort settings, agent counts, wall clock, token spend, and the ledger, coverage log, PoCs and *failed* search patterns. State what is **not** reproducible: sampling non-determinism, and provider-side model updates that silently change the cohort. A run that cannot be re-executed is an anecdote. |
| 10 | **Who reviewed the reviewers** | `✗` Which findings a human checked, on what sampling basis, with what qualification; whether any party outside the cohort re-ran a round; and **the disagreement rate between model families on the same finding**, published as a number. That last one is the metric that makes or breaks the cohort argument, and it is computable today. |
| 11 | **Effort in a comparable unit** | `~` Firms publish engineer-weeks and headcount, or dollars per week. We publish agent-passes × model × wall clock and compute spend, so a reader can price the claim against a firm's. |
| 12 | **Non-goals, stated** | `✗` No live-incident response, no key-custody review, no operational security review, and no validation of the economic model's *design* beyond what the stated derivations permit. Map to the Rekt Test and say which of the twelve we pass and which we do not: the contrast is more credible than a clean sheet. |
| 13 | **The named blind spot** | `✗` and the most important row. Every model in a cohort trained on overlapping corpora shares priors, so the cohort has a structural worst case: **novel-mechanism economics with no analogue in the training data.** That is precisely the coverage toll and the quartic impact curve, which is to say the parts of this protocol that are actually new. Compensating controls are derivation review, simulation, and a human economist, not more agents. Publishing the blind spot is what separates a framework from marketing. |

### 10.1 The standard a hostile reader will quote

SCSVS `0x04`: *"Automated tools alone are insufficient to verify SCSVS compliance. All verification
reports must provide conclusive, manually validated evidence."*

The wrong answer is to argue that agents are not automated tools. The right answer is obligation 10:
name the human validation, its sampling basis, and its rate.

### 10.2 Corrections to the received wisdom

The twelve corrections this campaign verified — ToB's ten Solidity maturity axes, the Rekt Test
provenance, the non-existent "two firms, same codebase" overlap study and its computable substitute,
the mis-attributed Consensys scope-drift rule, and the rest — are stated once, with their sources, in
`toolkit/references/landscape.md` (**Corrections this sweep made to received wisdom**). Nothing in a
published report may restate one without citing it there.

One prescription belongs here rather than in the registry: Dedaub's published position makes
mathematical correctness *secondary* and the client's job to specify. Use their reports as prior art
for BTR's code shape; do not cite their methodology as authority for economic review as a
first-class deliverable.

---

## 11. What this campaign has actually produced

⚠ **Regenerate before citing.** Verified 2026-09-09: the report of record was generated at 442 rows
while `LEDGER.md` holds **449**, and `04-rounds/ROUNDS.md` is headed "rounds 0–18" while carrying 41
table rows against **88 round directories running to round-97**. The §7 coverage assertion exists to
guarantee every ledger id is placed exactly once, so it is currently false by seven rows — the
assertion silently degrades from a check into a claim the moment the generation is stale. Run
`scripts/rebuild-report.py` before any external use, and never take an effort figure from a
hand-maintained index. Filed as CF-23/CF-24.

| metric | value | source |
|---|---|---|
| rounds | ~97 (88 round directories) | `ls 04-rounds/` — **not** `ROUNDS.md`, which stopped tracking |
| ledger rows | **449** | `grep -c "^| A-" LEDGER.md` |
| fully clean rounds | **disputed**: the two records that carry it disagree and both lag. Regenerate | report §10.2 vs `04-rounds/*/VERDICT.md` |
| blind rounds | R11 (8 independent experts, ledger withheld), R19–R21 | report §10.1 |
| repos pinned | 9 heads at freeze | report §0 |
| formal | halmos: skew bounds + monotonicity proven; toll lemmas timeout | `06-formal/`, `properties.md §3` |
| stop rule | **NOT MET** | report §10.3 |

---

## 12. Files

The file roster and what each one owns: `toolkit/SKILL.md §6`. `toolkit/references/routing.md` is
read first inside a round; this document is read once, outside one. This file says *why* rounds are
shaped as they are; `SKILL.md` runs one.

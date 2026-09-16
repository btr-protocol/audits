# BTR audits

Security review of the BTR AIMM: an oracle-priced, multi-asset AMM with coverage-based LP
protection. Internal reviews and third-party reports both land here.

Read [`DISCLOSURE.md`](DISCLOSURE.md) first. It says what is published and what is held, and why.
The short version: the method is public, open findings are not.

## What is here

- [`METHODOLOGY.md`](METHODOLOGY.md) — how a review round is actually run. Lens assignment, the
  refute-first protocol, what counts as evidence, and the stop rule that decides when a campaign is
  finished rather than merely quiet.
- [`method/SKILL.md`](method/SKILL.md) — the orchestration a round follows.
- [`method/landscape.md`](method/landscape.md) — every source the toolkit was built from: 196
  registered, 91 repositories cloned and read at a pinned commit. It records what was rejected and
  why, not only what was taken.
- `reports/third-party/` — vendor reports, published under the disclosure rule.
- `reports/internal/` — internal findings, published under the same rule.

## The finding that motivated publishing this

Measured duplication between this toolkit and the public corpus is **88 to 90 percent**. Nearly
everything in a private audit checklist is already written down somewhere public, and the residual
is small enough to count.

That has an uncomfortable implication and a useful one. The uncomfortable one: a checklist is not an
edge, and treating one as proprietary mostly protects the illusion. The useful one: if the checklist
is common property, the thing that separates a review that finds the bug from one that does not is
the *process* — how the work is split, how a claim is refuted, what makes a reviewer stop.

So the process is the part worth publishing, and it is what is here.

## Stop rule

A campaign stops after two consecutive clean full rounds on the launch surfaces — not when a round
is quiet, and not when a deadline arrives. A clean round is one that files zero rows at any
severity with every reviewer's target set enumerated and marked. The distinction matters: this
campaign has hit a clean streak and then broken it more than once, because a new surface always
yields a new round of low-severity findings, and any new or materially changed surface resets the
count for every path it touches. Severity converges long before count does, and only one of those
is evidence of anything.

`METHODOLOGY.md` carries the reasoning, including why a surface that did not exist when the campaign
began resets the count for every path it touches.

## Source of truth

This repository is a one-way mirror of `public/` in the private BTR audit workbook. Edit there;
publish with `scripts/publish-public.sh` (a `git subtree split` force-pushed to `main`).

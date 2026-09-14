# Disclosure policy

This repository is public. The protocol it audits is not yet fully deployed, and one testnet fleet
is live with real contracts. That combination decides everything below.

## The rule

**A finding is published when its fix is deployed to every chain running the affected code.** Not
when it is understood, not when it is fixed on a branch, not when it is merged. Deployed.

Until then it is held. There is no embargo clock and no partial hint: a held finding does not appear
here in redacted form, because a redacted finding still names its surface, and naming the surface is
most of the work.

## What that means in practice

| | |
|---|---|
| Published | The method: how the review is run, what it reads, how a finding is refuted or upheld, when it stops. |
| Published | The source registry: every checklist, corpus, prior audit and paper the toolkit was built from, and which of them turned out to be redundant. |
| Published | Third-party reports, once their findings are remediated and deployed, on the same rule. |
| Held | Every open finding, at every severity. |
| Held | Proof-of-concept exploits. These are never committed at all, in any repository. |
| Held | Implementation-specific checklists — call order, storage layout, constant values, known weak points. These are a map, not a method. |
| Held | Designs for defences that are not yet built, since a defence not yet built describes the gap it is meant to close. |

## Why the method is public and the map is not

An attacker with the method has what every serious reviewer already has. The measured duplication
between this toolkit and the public corpus is 88 to 90 percent — that number is in `method/landscape.md`,
and it is the reason publishing the method costs nothing. What it does not contain is which of those
generic questions this system answers badly. That is the map, and the map stays private until the
answers are fixed.

## Reporting something

Findings against deployed BTR contracts: **security@btr.markets**. Please do not open a public issue
for anything exploitable.

We will confirm receipt, tell you whether it is already in the private ledger, and tell you when the
fix is deployed. Once it is, the finding is published here with attribution unless you ask otherwise.

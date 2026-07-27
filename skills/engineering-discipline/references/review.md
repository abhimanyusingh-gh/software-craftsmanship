# Reviewing

## Every PR gets a reviewer. Mandatory.

No exemption for small, obvious, hotfix, or docs-only PRs. The moment an author reports "PR raised", the **very next tool call in the same response** dispatches the reviewer — reflexive, not reasoned. It runs in parallel with human review.

The only acceptable skips: a pure rebase or branch-sync commit with no diff, or an explicit instruction to skip this one.

It holds even for trivial PRs because the reviewer's value is the second opinion, not just bug-catching. Hotfixes shipped unreviewed turn out not to be one-line fixes; docs-only PRs carry broken references and drift. Skipped reviews mean issues surface later as owner-filed nits, or never surface and rot.

An automated reviewer is a real reviewer for this purpose — but it is not a human approval, and a PR that only ever had automated review should say so when it is surfaced for merge.

## What to review

**Walk `defect-classes.md` in frequency order.** It carries each class, the concrete shape to look for, its severity floor, and the rule it maps to. That list replaces per-dimension tabulation: report the classes that produced a finding and say nothing about the rest.

Beyond the class list, verify the load-bearing claims of the PR body actually hold — gate numbers, reused symbols, the migration's remaining-consumer list, the design-source citations. A body claim you cannot verify from the diff is itself a finding.

**Severity:** multiple correctness or duplication failures → request-changes. A single one → should-fix. Borderline style (a 51-LOC function against a 50 threshold) → nit, advisory. A dropped scope key, a deleted capability, a deleted shape with a live consumer, or an unbounded in-memory sort is a blocker regardless of how small the diff is.

## Write reviews as prose

**No verdict header. No Y/N tabulation. No labelled audit-line blocks.** A review that reads like a compliance form gets ignored.

Shape: one paragraph carrying the call embedded in a sentence, then numbered nits with `file:line` and concrete suggestions, then optional brief notes (informational findings, baseline verification).

Instead of:
> Verdict: FOLD-THEN-SHIP
> - Inline-cast audit: grep → 0 matches. Helper discipline holds.
> - Helper validation: rejects empty (Y/Y). Enforces pattern AND sequential check (Y).
> - Boundary completeness: all cast raw → branded via helpers. No slip-throughs found.

Write:
> Branding looks solid — every boundary I walked casts via the helpers, and the read paths re-brand on retrieval so the loose schema type holds up. One thing to fold before merge: the integration suite doesn't compile against the narrowed types, and since a compile failure is a gate failure it has to land in the same PR. The same pattern is already applied in the sibling suite — copy the import and helper-wrap the constants.
>
> Nits:
> 1. `types/branded.ts:95` — the fingerprint helpers reject only the empty string; whitespace and runaway lengths slip through. Suggest a trim check plus a length cap.

Keep the substance and the length. Change the voice. Concrete nits keep their numbered structure — that part is already useful.

**No bot-identity framing.** Drop opening prefixes and footers about how the comment was posted. A reader shouldn't perceive scaffolding around the prose. The mechanics of posting under a distinct reviewer identity are fleet plumbing — see `orchestration.md`.

**When line references would be noise, report to the orchestrator instead of the PR.** Full findings go back to the dispatching thread, which decides what to surface and what to dispatch as a fold. Post to the PR only when a specific finding needs to be traceable there for other humans — one short sentence, not a full review.

## Fold ALL reviewer items into the same PR

Critical, should-fix, nits, and test gaps — **one fold agent gets the whole punch list.** Never pre-split into "fold these now / file the rest as follow-ups": that's exactly how a backlog grows exponentially, and modules carrying money, compliance, or derived-value logic accumulate latent bugs fastest when "we'll fix later" piles up. The PR is whatever size the punch list dictates.

Deferral needs explicit owner approval and is limited to:
- An item requiring a new product decision the owner has to make.
- An item blocked on another in-flight PR's merge.
- An item genuinely larger than the rest of the PR combined — surface the trade-off ("this adds a new service and needs its own design; fold or defer?") and let the owner choose.

Anti-patterns: cherry-picking 3 of 10 items to "keep this PR small"; splitting nits and test gaps into a later "polish wave"; filing follow-up issues for items that fit in the current diff.

Fold agents stay strictly scoped to the findings (no scope creep), push to the **same branch** so the PR updates in place, and run the full gate sequence before pushing.

## Re-review after every fold

**Every fold triggers a fresh full-PR review** — not a review of the fold delta. Read the whole diff end to end, verify the folds landed correctly, and audit the rest at the same depth as the first pass.

Full scope matters because a fold can regress files outside its own lines (a renamed identifier propagating wider than the fold author noticed), and the original reviewer may have missed something on the first pass. A narrow re-review is too lossy. **Skipping the re-review means shipping the fold blind.**

The loop: review → fold → re-review → (fold → re-review) → clean. Every iteration is full-PR. Track where each PR sits in that loop.

**REVIEW-ROUNDS — expect two rounds; escalate on the pattern, not the count.** Two is typical and a third is unremarkable. Don't stop reviewing because a round budget expired, and don't read a 4th round as evidence the reviewer is pedantic. Escalate to the owner when the *shape* is wrong:

- Round N raises a defect class round N−1 also raised → the folds are treating symptoms. Demand the structural fix.
- A fold re-introduces a previously-fixed defect → blocker. Stop folding and re-scope.
- New findings still arriving in untouched files at round 4+ → the PR is too large. Split it.
- Rounds producing only nits → close the loop and file them.

Past 6 rounds, stop and re-scope by default. Long loops are usually mis-sliced PRs, not thorough reviews.

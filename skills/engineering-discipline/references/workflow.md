# Workflow — intake, PR slicing, migrations, gates, merge, docs

Fleet mechanics — agent isolation, briefs, the post-merge routine, phase sweeps — live in `orchestration.md`.

## Work intake

**Issue first, then PR.** File the tracking work item *before* spawning any author agent, and assign it to the owner on creation. The description carries: business context (why it matters), technical scope (concrete files, services, routes, hooks), verifiable acceptance criteria, plan reference, and explicit out-of-scope deferrals. Never one-liner descriptions.

**Walk the full lifecycle — never jump New → Closed.** Transition on dispatch, on PR raised, and on merge. The intermediate states are how the board reports velocity, in-flight load, and pickup gaps. **Query `<tracker>` for the work-item type's real state list** rather than assuming a default ladder — customised processes often reject the stock state names, and different item types often have different sets. Items stranded in states the current process dropped go invisible on the board.

Belt-and-suspenders on state: the agent flips state inline as part of the action that caused it, and a background watcher reconciles as a backstop. Neither alone is reliable.

**Every downstream work item cites the upstream capability that unblocks it.** A frontend item lists which backend phase and PR delivers each endpoint it consumes. If a dependency isn't shipped, mark the item explicitly blocked and don't dispatch. An item that introduces a *new* backend capability gets split backend-first, or scoped to one PR carrying both within the file budget.

## PR slicing

**≤20 files per feature PR**, tests and type files included. If a feature needs more, slice further. Mechanical refactors (rename sweeps, provider migrations, comment strips) relax to ~50 since per-file review burden is near zero — but don't bundle unrelated domain boundaries just to fit. When the budget is about to break, stop and report rather than sprawling.

**Backend and frontend alternate.** The backend PR lands first, shipping additive and backward-compatible API; the frontend PR then consumes it. Never one PR touching both sides — the system stays green between them, and rollback isn't all-or-nothing.

**One PR closes one issue.** A PR whose title enumerates a dozen closures is a batch, and a batch cannot be reviewed at the depth its riskiest item deserves.

**Respect the dependency chain.** Don't spawn a dependent PR's agent until its prerequisite merged, unless told to proceed speculatively. Audit file overlap across parallel PRs before launching a wave — conflict risk scales with file count × concurrency.

**Title format:** `<tracker-id>: <short description>`. Conventional-commit prefixes stay on the commits — orthogonal.

**PR descriptions are plain prose for a human.** Summary (one sentence of intent), why (the user-facing reason or bug), changes (a few bullets on the *shape* of the work — "extracted this helper", "backend service for that", "frontend consumes via this hook"), smoke evidence, gate results, issue link.

Banned: badges, shields, logos, colour-coded status chips, `file/path:123` citations, "locator at line NN" commentary, method-name enumerations, "N files changed" counts, implementation-mechanic explainers. The diff already shows the mechanics. Describe behaviour, not a mechanical inventory.

## Multi-PR migrations

**MIGRATE-ADDITIVE — cut over additively, in this order.** A rename, route move, key change, or schema reshape spanning more than one PR runs: (1) add the new shape alongside the old, both live; (2) move every consumer, one PR per consumer boundary; (3) delete the old shape in a final PR that greps clean. Never steps 1 and 3 in the same PR, and never step 3 before a repo-wide grep proves zero remaining consumers — tests, seeds, docs, and configs included.

Backward compatibility is non-negotiable throughout: additive schema fields, dual writes, feature flags, deprecation cycles. Never a breaking rename.

**MIGRATE-DUAL — while both shapes are live, they must agree.** Dual-mounted routes serve identical responses; dual-written fields stay consistent on every write path; one shim converts in one place, not per caller. Ship a test that exercises both shapes and asserts the same result.

**MIGRATE-CONTRACT — cite both sides of the seam.** Any PR that is one step of a migration names, in its body: the shape it adds or removes, the PRs that already moved, the consumers still on the old shape, and the PR that will delete it. Reviewers verify the claim against the diff. An unlisted remaining consumer is a should-fix; a deleted shape with a live consumer is a blocker.

**MIGRATE-KEY — every cache key, query key, and event name is grepped across the seam.** An invalidation naming a key no subscriber uses fails silently and forever. Before pushing, grep the literal key on both sides: the writer's invalidation and every reader's subscription must be the same string, produced by one shared key factory rather than typed twice. Applies equally to event names, channel names, flag names, and storage keys. Reviewers grep the key out of the diff and confirm at least one live consumer.

## Gates before push

Derive the actual commands from the repo — the lockfile, the task-runner scripts, the CI workflow — and record them in the project's `CLAUDE.md` on first use. The **order and the invariants** are the rule; the commands are the project's.

1. **Force-clean and rebuild `<shared>` first.** Any package other workspaces compile against gets its build output deleted and rebuilt from scratch. Incremental builds do not reliably refresh stale output, and a worktree resolving a shared package to another branch's build reports failures that don't exist.
2. **Type-check every workspace**, whole-program, not per-file.
3. **Run the full test suite per workspace** — no path filter, no changed-files scoping.
4. **Run the production build** for anything that ships one. It is stricter than a type-check alone and catches what a type-check misses.
5. **Run `<deadcode>`** if the repo has such a check.
6. **Run any project-specific contract or registry checks.**
7. **Run the full `<e2e>` suite against a live local stack built from this branch** — the target for functional-code PRs, per `testing.md`.

*Example, from a yarn/tsc monorepo — derive your own:*
```
rm -rf shared/dist && yarn build:shared
yarn --cwd backend tsc --noEmit
yarn --cwd backend test
yarn --cwd frontend tsc --noEmit
yarn --cwd frontend test
yarn --cwd frontend build
yarn --cwd backend knip
yarn e2e
```

If anything fails, **do not push** — fix in-PR or hand back.

**Scoped test runs are for the inner loop only, never the gate.** Scoping to changed files misses cross-file compile failures in *other* files' fixtures and mocks; those PRs merge and break trunk for everyone. The gate mirrors the unfiltered CI command exactly. A harness flake is not an excuse to skip — re-run once, read the actual output to distinguish flake from failure, and if it's still red, report rather than ship.

**A type error fails the build, pre-existing or not.** Never rationalise one as "pre-existing" without a clean rebuild; a phantom count contradicting an earlier green run on the same branch is the tell that the shared build is stale. CI needs the same treatment: any job that type-checks, tests, or builds a consumer must build the shared package first.

**Report every gate in the PR body with numbers** — pass counts, skip counts, durations, and the dead-code result. A gate claimed without a number is not evidence.

**Validate in a browser before claiming done.** Tests and type-checks are the floor, not the ceiling. For every UI-touching PR: build and start the local stack, confirm the health endpoints, open the affected surface, and smoke what you changed —

- Chrome or layout changes: render and compare against the design reference side by side; document any delta.
- Interaction changes: click every changed control three times; confirm deterministic navigation and state.
- Data-fetch changes: trigger it and verify the network call and response shape.
- Forms: valid, invalid, and empty submissions.
- Modals: open, close, reopen.
- Decompositions: every render branch the tests cover — loading, empty, error, data, filtered.
- Locale- or timezone-sensitive tests: re-run under both local and CI settings.
- Run the exact CI command, not an approximation — workspace and config differences change behaviour.

Report it in the PR body under a **Browser smoke** heading, concretely: surface, what was clicked, what responded. If you couldn't smoke, say so explicitly — "did NOT browser-smoke; gates passed but live behaviour unvalidated." No "all gates green" claim is valid for a UI PR without smoke evidence. Reviewers treat a missing or thin section as should-fix, request-changes for user-facing chrome. A reviewer who can't smoke may approve with nits at most, never clean, on visual changes.

## Merge

**Auto-merge is OFF by default.** Run the full author → review → fold → re-review loop, get every gate green, then **stop at "PR is ready"** — surface the number, summary, and verdict, and let the owner merge. Never complete a PR without an explicit per-PR or per-batch go-ahead. Any authorization given is scoped to that batch and lapses when it finishes.

When auto-merge *is* explicitly authorized, the gate sequence is: CI green → reviewer approve-clean → no outstanding request-changes → mergeable. Approve-with-nits or any should-fix means a patch agent addresses **every** item, then re-review; only approve-clean merges. A bare comment verdict goes to the owner. Surface rather than auto-merge on: non-trivial conflict resolution, CI failures outside known flake patterns, any surprise rippling beyond the PR (design-intent change, scope expansion, broken downstream surface), and initiative completion.

**Never:** skip commit hooks, amend a published commit without authorization, force-push the default branch, or bypass branch protection without explicit per-scenario authorization.

## Documentation

**Docs are never outdated.** If a change alters anything a README, doc, or runbook asserts, that update ships in the *same* commit. No "docs follow-up later" — a doc that lies is worse than no doc.

Triggers: a feature added, removed, or changed → feature docs, key-features list, roadmap. A file or folder moved → every doc link to it (grep repo-wide). A contract changed (route shape, response schema, export format) → architecture doc and any README claim. Env, port, seed, or setup changed → getting-started and operations docs. A milestone completed → roadmap and status.

**Operator-facing changes ship their runbook update.** Route contract changes (status codes, response shapes, sync→async, retry semantics), work moving off the request thread, new env vars and their defaults, container orchestration changes, schema and index additions, seed-data semantics, auth and capability changes — anything an operator gets paged about. Identify the doc when scoping; if none exists, create it. The PR body carries a "Docs touched" section. Reviewers hold any PR hitting the trigger surface with no docs diff.

A runbook that cites a line number is a runbook that rots on the next diff. Cite the function or the behaviour, not the line.

**Docs describe what the code does, not what's planned.** Aspirational content is explicitly labelled as not-yet-built. Never let a doc imply unbuilt behaviour is live.

The one carve-out: a pre-launch marketing page may present the full product vision under clear "coming soon" framing. Two honesty lines still hold even there — no fabricated traction, customer counts, or accuracy figures presented as real outcomes; and security certifications presented as "readiness underway", never asserted as already held. Product UI, docs, and runbooks stay strictly shipped-reality.

Keep one docs taxonomy (specs / runbooks / architecture / operations / qa / security / design) and don't invent a parallel tree. Briefs cite current paths after any reorg. Design-system paths are pinned — tooling and briefs depend on the exact path.

# Orchestration — running a fleet

These are fleet operating rules, not code rules. They come from running the workflow rather than from reviewing diffs, so treat them as an operating model to adapt rather than a measured standard.

## The main thread never codes

It reads specs, slices work, spawns agents, reviews returned PRs, and summarises. It never edits source files, never commits, and never fixes an agent's output directly — it spawns a follow-up agent. Its only writes are to plans and memory. Mixing edits into the orchestrator blurs attribution and burns the context budget.

**Workers on the cheap model tier, orchestrator on the session model.** Pin the cheaper tier explicitly on every author, fold, code-fix, reviewer, re-review, and research agent. Never *omit* the model when the session runs a strong one — omitting inherits it onto the whole fleet. Planning and synthesis get the strong model; execution and review don't need it.

## Decompose by unit, not by phase

**One agent per component or module, 2–3 files each.** Never hand all the structural changes in a feature to one agent. A single agent accumulates context until failures stop being isolable — in the case this rule comes from, a whole-phase agent ran two hours and finished with 18 test failures because it changed every component before running the suite once.

Two budgets, and they are different numbers. The `≤20 files` cap below is a **PR** budget. The 2–3 file cap is an **agent** budget. One PR is usually several agents.

Structure the work as rounds:

1. **New components** — nothing existing to break, so run them all in parallel.
2. **Independent refactors of existing components** — parallel, provided no two agents edit the same file. Audit that before launching.
3. **A single wiring agent, last** — sequential, and it only wires sub-components that are already implemented and tested.

**Review mirrors implementation.** Don't spawn one reviewer for the whole PR when the work was decomposed. Mirror the rounds: each validation agent checks its own component against the rules, its own coverage, and its own capability preservation. The orchestrator aggregates before any push.

**Incremental gates, never deferred.** Each agent runs `<unit>` after its own change and fixes red before moving on. Accumulated test debt across components is what produces the long fix loop at the end — the failures arrive together, entangled, at the point where context is most exhausted.

**Set a checkpoint.** Every brief carries a budget: "if you are more than N minutes in without a clean gate run, stop and report your blockers rather than continuing to iterate." Silent long-running loops are expensive and they rarely recover on their own.

## Isolation

**Every parallel agent gets an isolated work path, and isolation is the default.** Prefer the harness's worktree isolation; fall back to a git worktree under a temp path keyed by a short id. Never let two agents share one checkout.

The exception is narrow. Launch *without* isolation only when the agent must see **uncommitted changes that already exist in a specific working tree** — then give it that tree's absolute path and tell it to work there, because isolating hides the very work it was sent to finish. A clean checkout of the base branch is not that case; it is the fresh-work case, and it takes a worktree. Reading this backwards costs a whole round of parallelism.

**Branch from a freshly-fetched remote ref.** Run `git fetch origin` immediately before creating the worktree or checking out the base branch. A stale local ref silently bases the branch on an older main, causing a later rebase that touches files outside the PR's scope and contaminates the diff.

Symptoms of violation: HEAD swapping to a sibling's branch mid-shell-call, stashes auto-popping from another agent's pull, `git status` showing unfamiliar files, `git log` showing a sibling's commit, a PR opened with mismatched contents, an unexpected dependency install. Stay in the isolated path for the whole session — never change directory back to the primary checkout.

Reviewer agents don't need a checkout at all: read the diff through the platform API rather than physically checking out a branch in a contested workspace.

**Commit and push immediately, then after every unit of work.** A WIP commit plus an upstream-setting push right after branching. Uncommitted work in a worktree is unrecoverable if the agent stalls or a cleanup daemon reclaims the directory; pushed commits always are. Any worktree-reclaiming automation must skip worktrees with a dirty tree, and must live outside the git tree itself — an in-tree script gets deleted under its own running process by a stash.

## Concurrency

**Cap concurrency on heavy work; exempt continuation work.** Heavy new-work authors run 2–5 concurrent, tuned to host RAM — test and e2e agents are memory-hungry. Fold, rework, and re-review agents are **exempt**: they continue already-counted work, so dispatch them immediately rather than queueing. Drain a backlog through a concurrency-capped pool, not one-agent-per-item. Never kill a running agent to hit a lowered cap; let it drain.

**Cluster stalls mean the host slept.** If 2+ background agents trip the watchdog in the same window, the machine suspended — the briefs were fine. Relaunch each verbatim: same brief, same scope. Don't narrow scope, strip test commands, split into pieces, or offer the owner options A/B/C for what is an environmental fact. A *single* isolated stall can still be a brief problem — an overly long operation, or no heartbeats.

## Gates

**Run gates in the foreground.** An agent that backgrounds a long build or test run and then stops loses the result and reports a stall as if it were a completion. Run with a generous timeout; re-invoke and keep waiting rather than abandoning a gate mid-run.

**A gate that could not run is reported as not-run, by name.** `CLAIM-EVIDENCE` in `verification.md` carries the rule and what counts as evidence; the fleet-specific half is that a stalled foreground gate and a blocked one report differently, and an agent that conflates them hands back a false green.

## Verify returned work, don't trust the report

An agent's report is written by the party whose work the finding would invalidate, in the medium where a confident claim is cheapest. Treat it as a claim to check, not a result.

Before pushing anything an agent hands back, the orchestrator runs `REGRESS-DIFF` from `verification.md` against the base branch. Not because agents are dishonest — because the failure this catches removes its own evidence. A capability deleted along with its test leaves a green suite, a clean diff summary, and a plausible comment explaining the removal.

The same applies to the contract card. A card is only worth requiring if unfilled rows are treated as unfilled: an adjective where a number belongs, a missing gate row, an empty capability check. Send it back rather than reading past it.

## Briefs

**Every brief tells the agent to load `craft` itself, and pastes the contract card.** Loading is the agent's first action, before it reads a single source file; the references it read go on the card. A paraphrase of the rules in the brief is not a substitute — but neither is pasting all fourteen non-negotiables, which is what briefs used to do and how they grew to a hundred lines of prose with the invariants buried in the middle.

**Name safety-critical tests by exact test name** and label them blockers, rather than listing them among the acceptance criteria. A test guarding a production safety requirement should be impossible to miss in a skim.

**Pre-resolve environmental blockers in the brief.** Uninstalled dependencies (mandate the install), missing fixture paths (point at the actual file), missing browser binaries (waive or provide). Never let an agent hand back on something the orchestrator could have resolved up front.

```
Workspace:        isolation mode + fallback + "stay in this path"
Load first:       invoke `craft`; read the references for this task type; list them on the card
Scope:            2–3 files for a component agent; ≤20 for a whole PR. STOP and report rather than sprawl
Backward compat:  additive fields, dual writes, flags — never break a caller
Reuse mandate:    search first, extend over clone, name reused symbols on the card
Grounding:        the spec docs governing this work; cite decision IDs in the body
Invariants:       the contract card, pasted; safety-critical tests named as blockers
Gates:            the ordered pre-push sequence with baselines; run after each unit, not at the end
Checkpoint:       stop and report blockers past N minutes without a clean gate run
Tracking:         the work item this closes; title format; state transitions
No merge:         open the PR, return the URL, stop
If blocked:       stop and report — never invent a contract, guess a schema, or bypass a check
```

## Reviewer identity

Reviewer comments post under a distinct reviewer identity; author and patch agents replying or resolving post under the human identity, so the conversation reads reviewer → author. Post reviews via the raw API rather than a CLI review command that leaks the human identity, and **verify the identity after writing** — a token can fall back between shell invocations and silently post as the author. If the platform has no usable separate identity, don't invent one: report findings back to the orchestrator instead of posting.

## Multi-session mega-PR

When one PR is explicitly authorized to exceed a single agent's context, chain sequential layering sessions on the same branch rather than compressing into one over-budget run. A single-shot mega-session lands partially and stalls.

- Write a comprehensive plan first; have a separate agent audit it against the rules; revise before execution.
- Each session: worktree-isolated, branched off the latest tip of the shared branch, dependencies installed first, pushing incrementally after each sub-item, reporting per-item status plus a recommended next-session scope.
- Auto-dispatch the next session when the prior lands cleanly. Stop only on a genuine blocker, an interrupt, or completion.
- First session is heaviest (the architectural spine); mid sessions carry 3–6 items; the final session is the exit gate — closeout doc, PR description, ready-for-review flip.
- Track baseline test-pass counts across sessions. A regression past baseline minus known deletions is an introduced defect; fix before push.
- Re-paste the contract card every session, and require it back filled in. A later session inherits none of the earlier ones' context and drifts fastest.
- The closeout doc lists every backlog item as done / partial / not-done-with-reason. Nothing silent; the not-done items become the explicit follow-up scope.
- When an agent proposes "defer to a follow-up PR", override it. Single-PR mandate means single PR — that suggestion is usually a context-budget hedge, not a structural call.

Don't use this for small scoped PRs, work where backend/frontend alternation matters, or time-critical hotfixes.

## After merge

Five steps, in order, every time:

1. **Fast-forward the local default branch and push every mirror** so all remotes stay in lockstep. Keep it a *free ref* — agent worktrees branch from the remote ref and never check it out locally; a stale worktree holding it jams pushes to any non-bare mirror, so make mirrors bare. A background watcher polling the canonical remote is the safety net for merges happening outside the workflow.
2. **Confirm the auto-triggered CI build goes green.** Fix red immediately, before the next merge — never let CI accumulate red across merges.
3. **Run the full production build.** A PR changing a shared type can pass its own isolated type-check and tests while breaking consumers — a newly-required field at a construction site, a new enum member an exhaustive mapping must cover. Only the whole-program build catches it. Any PR changing a shared type must run the consumer's production build *and* grep every consumer of that type, updating them in the same PR.
4. **Refresh the local deployment** from the merged default branch so the running stack reflects merged code. Use the additive seeder, never a destructive reseed on a running stack. Don't clobber a checkout sitting on a feature branch with uncommitted work, and don't launch the heavy rebuild while many subagents compete for CPU and RAM.
5. **Clean up the merged PR's worktrees and temp dirs.** Remove the worktree through git and prune, then delete ad-hoc temp dirs. Never touch a worktree an agent is still using; never delete one from the filesystem directly — git's admin state goes stale. Periodically reconcile the worktree list against in-flight agents.

## Phase hygiene

**Drain open issues to zero before starting a new phase.** At every boundary, list open items and give each a disposition: resolved now, closed with a justification comment in its own thread, or explicitly carried into the next phase's plan *with the reason recorded*. No item silently survives a boundary. Carrying a tail hides scope, compounds every phase, and blurs deferred versus forgotten.

**Run a phase-focused artefact sweep after each phase.** Per-PR review catches what's in the diff; cross-PR composition issues don't surface there. Once the functional PRs are merged, the full e2e suite has run, and surfaced bugs are closed, dispatch a phase-bounded audit — one agent, or several across disjoint artefact classes:

- **Code defects** — dead exports, unused imports, stale markers, orphan files, abandoned flags, log leaks, hardcoded values that should be config-backed.
- **Cross-PR inconsistency** — naming drift with stale callers, raw strings where an enum exists, casts bypassing brand helpers, test patterns diverging from the standards, cache keys with no live subscriber.
- **Tracker hygiene** — every item closed if shipped, correctly parented, dependency block populated, PR referenced, thin descriptions flagged.
- **Docs** — a runbook for every operator-facing flow matching code reality; README and env examples current.
- **Test coverage** — every endpoint has backend tests, every component a unit test, every user flow an e2e spec, the seeder rule honoured, the skip baseline unchanged or lower.
- **Decommissions** — stub UIs, "coming soon" placeholders, and dead copy removed now that the real feature landed.
- **Design fidelity** — inline-CSS sweep, design-reference comparison on visual surfaces, undefined-token grep.
- **Scope isolation** — every model carrying the scope key enumerated and guarded; negative tests present per surface.

Output: defects classified by severity, filed as work items (genuine regressions, not nits), fixed as one consolidated PR if small and co-located, otherwise individually. **A phase isn't complete until its sweep is done and every defect it surfaced is closed.** Skip only for a "phase" of a single PR.

**Carry known bug patterns forward as a per-feature checklist.** After a hardening cycle, distil the failure categories into a checklist every subsequent feature is tested against at unit, integration, and e2e layers; reviewers hold a PR if an in-scope category is uncovered. The categories that generalise are the ones in `defect-classes.md` — plus these, which are cross-PR rather than per-diff:

1. **Field-name and unit coherence** — every response field a consumer reads is named and unit-tagged consistently across producer and consumer.
2. **Modal close-on-success plus invalidation cascade** — close on success, invalidate every relevant key (list, detail, sidebar counts), surface errors without closing on failure.
3. **Identity derivation** — one producer function for fingerprints and slugs, matched at every consumer; no fallback chains.
4. **Seed determinism** — deterministic ids, upsert on canonical keys, idempotent on re-seed, prior test artefacts cleared on start.
5. **Fixture validity** — identifiers match their real format. Malformed fixtures look like product bugs.
6. **Cascade purge** — deleting a parent cascades to every dependent table, and the dependency count is exhaustive.
7. **Required-relationship enforcement at both layers** — the backend invariant throws *and* the form disables submit. Persisting a record without its mandatory relation is a data-corruption class bug.

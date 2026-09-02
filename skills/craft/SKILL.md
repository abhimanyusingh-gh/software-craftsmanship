---
name: engineering-discipline
description: Engineering standards for writing production code, slicing and reviewing PRs, authoring tests, running pre-push gates, and merging — on any stack. Covers code reuse and duplication, layering, enum and type discipline, complexity caps, tenant/scope isolation as a security invariant, dead code versus unwired capability, config-from-day-one, index-per-query, URL providers, server-versus-client state, loading and empty state contracts, accessibility, bulk-operation partial failure, design fidelity, seed data, invariant and e2e test standards, the review-fold-re-review loop, multi-PR migration cutovers, the full-gate sequence, merge policy, and agent-fleet orchestration. Load before writing production code, opening or reviewing a PR, authoring tests, pushing, or merging.
---

# Engineering discipline

## The non-negotiables (apply always, no reference read needed)

1. **Scope isolation.** Every read, write, aggregate, export, and storage path filters on the full `<scope-key>`, enforced at the lowest layer that can rather than at N call sites. A dropped scope key is a cross-customer data leak: blocker, never a nit.
2. **Reuse before writing.** Search for an existing implementation first; extend or parameterize rather than clone. List every reused symbol in the PR body and justify anything new. Duplicated logic: 2× → should-fix, 3× → request-changes.
3. **No narrative comments.** The code is the doc. Comment only a genuinely non-obvious *why* — a hidden invariant, a known workaround, a surprising external constraint. No file-header docstrings, no "TODO until the issue lands".
4. **Const-as-enum over repeated string literals.** Any fixed variant set gets a named constant object plus derived type, in preference to a language `enum` keyword. Consume the exported enum; never re-type its values.
5. **No raw `string` for semantic values.** Branded types for IDs and domain primitives, const enums for finite value sets. Grep for the existing brand before creating one.
6. **Export only what has an in-PR consumer** — no speculative exports, no barrels nothing imports through. But when a dead-code tool flags something, **classify before deleting**: scaffolding is deleted, a spec-required capability is wired to its consumer in the same PR. Deleting a capability is not cleanup.
7. **YAGNI.** Nothing beyond what the issue requires. No speculative props, no single-caller indirection, no "while I'm here" expansions.
8. **Debloat on touch.** Editing a file means scanning the surrounding 50–100 lines for dead code, duplicated logic, repeated literals, oversized functions, mixed concerns. Fix or flag in the same commit.
9. **Deliberate before coding.** Produce an edge-case checklist, then a one-paragraph plan stating what's guarded and what's deliberately not done — before implementation.
10. **Tests define the product.** A red test is fixed in product code, seed data, or config. Never re-pin an assertion. Only sanctioned spec edits: compile-blocking typos, framework misuse that never caught a bug, stripping banned patterns.
11. **Full unfiltered gates before push.** Never scope the test run to changed files as the gate. Report every gate in the PR body with numbers. If anything is red, don't push.
12. **Docs ship in the same commit as the change they describe.** No "docs follow-up later."
13. **Raise and stop.** Auto-merge is off by default: run the full review loop, then surface the PR for the owner to merge.

**Severity vocabulary**, used everywhere: `blocker` (must not merge — data leak, deleted capability, broken contract), `should-fix` (fold into this PR), `nit` (advisory, may be filed).

## Project bindings

Rules are written against capabilities, not tools. Derive these from the repo on first use — the lockfile, the task-runner scripts, the CI workflow, the schema — and record them in the project's `CLAUDE.md`. Never assume them.

```
<pm>          package-manager / task-runner invocation
<typecheck>   whole-program type check, per workspace
<unit>        full unfiltered unit + integration suite, per workspace
<build>       production build, per workspace
<deadcode>    dead-export / unused-dependency check, if the repo has one
<e2e>         browser end-to-end suite
<shared>      the shared package other workspaces compile against
<workspaces>  the deployable units
<query-lib>   the server-state cache library
<store>       the client-state store
<db>          the datastore and its query vocabulary
<tracker>     the work-item system and its id format
<remote>      the canonical git remote
<scope-key>   the field that partitions data between customers
```

## Which reference to read

Read on the trigger, before starting — not after review flags it.

| Trigger | Read |
|---|---|
| Any query, write, storage path, or cache key touching customer data | `references/tenancy.md` |
| Production code — types, config, naming, complexity, dead code, layering, indices | `references/authoring.md` |
| Any UI work | `references/frontend.md` |
| Any interactive component, form, dialog, or announcement | `references/accessibility.md` |
| Any test, unit through e2e | `references/testing.md` |
| Reviewing a diff | `references/defect-classes.md`, then `references/review.md` |
| Folding review findings | `references/review.md` |
| Slicing PRs, multi-PR migrations, gates, merge, docs | `references/workflow.md` |
| Running an agent fleet | `references/orchestration.md` |

Authoring a UI feature usually means `frontend.md` + `accessibility.md` + `testing.md`. A backend feature usually means `tenancy.md` + `authoring.md` + `testing.md`.

Every worker agent's brief restates these non-negotiables **verbatim** — agents drift without explicit re-statement. Fleet mechanics are in `references/orchestration.md`.

---

Distilled from review findings and PR history across two production codebases. The evidence behind each rule — instance counts, and the claims the data disproved — is published in `EVIDENCE.md` in the repository, deliberately outside this skill so it costs no context here.

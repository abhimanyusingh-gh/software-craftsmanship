# Authoring production code

Rules are ordered by how often the defect actually occurs, most frequent first.

## Before writing any code

1. **Edge-case checklist** derived from the issue, the file's call sites, and known past failure classes for the domain.
2. **One-paragraph plan**: what the change does, which edge cases are guarded, what is deliberately NOT done.
3. **Debloat on touch** — scan the surrounding 50–100 lines for dead code, redundant predicates, duplicated logic, repeated literals, oversized functions, parameter explosion, mixed concerns. Fix or flag in the same commit.
4. **YAGNI** — nothing beyond what the issue requires. No speculative props, no single-caller indirection, no "while I'm here" expansions.
5. **SOLID** — single responsibility per unit; depend on abstractions at boundaries.
6. **DRY** — search for the existing helper before writing a new one.

Reactive bug-fixing after merge is the failure mode this replaces.

## Code reuse

1. Search the codebase for an existing implementation covering the use case.
2. Prefer extending or parameterizing over cloning.
3. If new code is genuinely needed, design it to *consolidate* rather than fragment — one service for every report query, not one per endpoint.
4. When a pattern appears a second time, extract the shared primitive and refactor the first occurrence — same PR if it fits the file budget, next PR otherwise.
5. List every reused symbol in the PR description; justify anything new.

**REUSE-WRAPPER — wrappers that add no behaviour are a defect.** A hook that only calls one store action, or a service function that only forwards one API call, is a layer for nothing — collapse it to direct use. Name things after the action (`selectItem`), not the implementation (`useEnsureItemSelected`).

**REUSE-SIBLING — two hooks or services hitting the same endpoint is one too many.** Near-identical fetchers that return disagreeing shapes for the same resource are the most common form of this defect. Grep the endpoint before writing a consumer of it.

## Layering

- **Domain** — pure business types and transformations; imports nothing from the source tree outside its own folder.
- **Application** — hooks/services composing domain + adapters into use cases.
- **Adapters (outbound)** — API clients, persistence, integrations.
- **Adapters (inbound)** — thin components/controllers: state from hooks, layout and wiring only, no business logic.

**LAYER-PORT — components import from the feature's use-case hook, never from the API layer directly.**

**LAYER-LEAK — a provider or strategy never leaks into the steps that use it.** Pipeline steps stay agnostic of which engine, vendor, or extraction source is behind them; the choice is resolved once at the boundary and passed as a port. A step that branches on the provider name has to be edited every time a provider is added — that is the open/closed violation this rule exists to prevent.

**LAYER-GOD — a method that orchestrates every branch of every strategy is the same violation one level up.** Push each branch behind its own boundary and let the orchestrator dispatch.

## Dead code

**DEAD-EXPORT — export only what has an import site inside the same PR.** Barrel files re-export only what something in this PR imports through the barrel — and don't create the barrel at all if nothing consumes it, or the barrel itself becomes the dead export. The next PR pays the one-line cost of adding its own export, bundled with the consuming code, exactly where dead-code detectors expect it.

**DEAD-CAPABILITY — an unwired capability is not dead code.** When `<deadcode>` flags something, classify it before deleting.

**Delete** when it is *scaffolding*: a speculative export you or a recent PR added, a barrel nothing imports through, a wrapper that adds no behaviour, an abandoned duplicate of a live path, a flag whose feature already shipped.

**Wire it** when it is *capability*: it implements behaviour the spec, design source, issue, or API contract requires, and the only thing missing is the call site. Deleting it deletes the feature. In that case, in the same PR: wire it to its real consumer, cover it with a test, and say so in the PR body.

The test: would a reader of the spec expect this behaviour to exist? If yes, the defect is the missing wiring, not the unused symbol. Never resolve a dead-code flag on a spec-required function by deleting it and filing "re-implement later" — that is scope loss disguised as cleanliness. An integration whose credentials are already in the environment is capability, not scaffolding.

If you cannot tell which case you are in, stop and ask. Do not delete.

## Enums and literals

**ENUM-CONST — any parameter or field with a fixed variant set gets a named const-as-enum object plus derived type**, in preference to a language `enum` keyword where the language has both. Enums make valid values discoverable through the type system; literals plus a clarifying comment don't.

```ts
export const ITEM_STATUS = { draft: "draft", approved: "approved" } as const;
export type ItemStatus = typeof ITEM_STATUS[keyof typeof ITEM_STATUS];
```

**ENUM-IMPORT — consume the exported enum, never re-type its values.** A component that accepts `"small" | "large"` when the design system already exports `BADGE_SIZE` has forked the contract.

**ENUM-GROUPED — when code repeatedly asks "is this one of a subset?", the subset is part of the enum.** Export the group (`TERMINAL_STATUSES`) beside the values rather than making every call site grep for which members qualify.

## Complexity limits

| Signal | Threshold |
|---|---|
| Function length | >50 LOC → question it; >100 → should-fix unless the length is data (dispatch table, config) |
| File length | >400 LOC for one concern → split candidate |
| Conditional chain | 4+ `else if` branches, or 6+ `switch` cases → lookup table / strategy map |
| Parameters | 5+ positional → options object; 8+ → smell regardless of shape |
| Nesting | past 4 levels → guard clauses / early return / extract helper |
| Growth over time | touched-and-grown in 3+ recent PRs → cost-of-extension is climbing, call it out |
| Mixed concerns | parsing + validation + persistence + side effects in one function → decompose |
| Implicit state | long function threaded with mutable locals and flags → extract pure functions |

Exempt: schema/config tables where length *is* the data, generated files, test files (covered by the test-value rule). Every flag proposes a concrete decomposition — never just "this is long."

## Naming

**NAME-TAXONOMY — one word per concept, repo-wide.** Grep for the concept before naming anything and reuse the existing word. Two words for one thing (`org`/`tenant`/`workspace`, `member`/`user`/`profile`, `archive`/`disable`/`deactivate`) is a defect the moment the second appears — in code, types, routes, query keys, test ids, copy, and docs. When you find a genuine split, unify it in the PR that noticed it and grep for stragglers.

**NAME-SHAPE — the same concept keeps the same shape across layers.** A field is `startDate` in the schema, the DTO, the hook, the component prop, the test id, and the design source — or the rename is deliberate and documented at exactly one boundary. Silent per-layer renames force every reader to hold a translation table.

**NAME-CASING — one casing per surface.** Enum values, route segments, query keys, and test ids each pick one convention and never mix. A value written `PartialItem` in one file and `partial_item` in another is two values as far as any datastore is concerned, and queries silently miss rows.

## Configuration

**CONFIG-DAY-ONE — tunables are DB- or config-backed from the first PR that introduces them.** Never hardcode-then-refactor; never a constant plus a "make configurable later" marker. Weights, thresholds, and scoring factors are the most-missed case: a bare numeric literal in a ranking or matching function is a magic number.

| Category | Storage |
|---|---|
| Externally-mandated rates, thresholds, and deadlines that change on an effective date | DB table keyed by effective period — auditable, per-scope override, hot-rollable |
| Per-scope operational tuning (approval thresholds, retention windows, batch sizes) | DB row on the scope record |
| Process tunables (retry backoff, poll intervals, timeouts, cache TTL) | config file / env |
| Safety guards (max batch, max response size) | config file |

*Not* tunables: API-contract enums (interface boundaries), sentinel values, branded-type carriers, file paths, table and collection names.

Every config service carries a hardcoded default for fresh installs — living **in the service**, never scattered through callers. Callers always go through the service.

**CONFIG-FETCH — user-data config comes from the backend; the frontend never mirrors it.** Two categories of frontend constant:

- **A — user-data config** (option lists, master data, rates, workflows, classifications, role assignments): must be fetched. Never a static frontend file mirroring a backend seed — two sources of truth that drift the moment either side changes.
- **B — API-contract enums** (status values, wire-format identifiers, error codes): fine as a frontend const-as-enum, with reviewers verifying matching backend values. This is the one place a "keep in sync with the backend type" comment is acceptable.

Every PR adding a frontend constant answers: user-data or API contract? Never dual seeds.

**CONFIG-DERIVED — if a typed constant already enumerates the set, derive from it.** A local array re-listing valid tab names beside an existing route-map constant is a second source of truth.

## URL / URI abstraction

**URL-PROVIDER — no raw URI or path string at a call or registration site.** Every URL comes from a typed provider module. Three providers, one per direction, at paths bound from the project:

- Frontend callers (API client, `fetch`, image sources, event streams, sockets) → the frontend's URL provider for that domain.
- Backend route registrations → the backend's route-URL provider for that domain.
- External services (third-party APIs, identity providers, cloud endpoints) → the backend's integration-URL provider for that service.

Need a new path? Add the constant to the provider first, then consume it. Stop and ask if you can't tell which provider owns a path.

**URL-ENFORCE — layered enforcement; a custom checker must understand the constant it mandates.** Typed providers catch within-side drift at compile time, reviewers grep the diff on URL-touching PRs, and e2e catches cross-boundary drift at runtime. That is the default stack and it needs no extra tooling.

A custom static checker is worth writing only when a rule spans two languages or two build graphs no single type checker sees at once — a client/server route contract, a path-scoping invariant, a bundle budget. If you write one:

- It must resolve the *typed constant*, not just string literals. A checker matching only `"/api/items"` will flag the provider constant this skill mandates as a violation and block the correct fix. This has happened.
- It ships with its own tests, including one asserting the compliant typed form passes.
- It reports file and fix, never just a count.
- It is owned. When it blocks a correct change, the checker is the defect: fix or delete it in that PR. Never work around it by reverting to the string literal.

## Style

**No narrative comments.** Default to zero. If a reader needs prose to follow the code, the code is wrong, not the comment. Comment only a genuinely non-obvious *why* — a hidden invariant, a known workaround, a surprising external constraint.

Also banned:
- Top-of-file "invariants asserted by this file" docstrings on test files — the assertions are the documentation.
- "TODO until the linked issue lands" markers.
- Narrative ceremony added by reviewer or fold agents.

A comment explaining what an enum value means is a request for a named enum, not a comment.

**Test names are self-documenting.** Banned in test and suite names: issue or PR numbers, internal bug nicknames, PR scaffolding ("post-rebase", "PR 2 follow-up"), anything meaningful only inside the current review thread. A reader six months out must understand the locked-in behaviour from the name alone.

## Type safety

**Raw `string` for a semantic concept is a bug.** For every new field, parameter, and return type ask:

1. **Semantic ID?** (scope id, user id, document id) → branded type: `type Uuid = string & { readonly __brand: "Uuid" }`. **Grep for an existing brand first** — it usually exists.
2. **Enum-like with finite well-known values?** → const-as-enum per `ENUM-CONST` above.
3. **Domain primitive?** (money in minor units, dates in a fixed timezone, formatted identifiers, ratios) → branded or wrapper type. Reuse first.
4. **Schema or ORM field?** The schema takes the loose primitive; the application type uses the typed value. Cast at the schema boundary — never propagate raw strings outward.

New typed declarations go in one focused file per concept under the workspace's types directory, named for the domain concept not the consumer. Export the type and the const object together; don't bundle unconsumed validators or converters.

Reach for the const-as-enum first and brand where a real mix-up is possible — cross-scope id mix-ups are the case that matters, and they pass every type check without a brand. A brand around a single-call-site value is ceremony.

## Database

**INDEX-COVERING — every new query path ships its covering index in the same PR.** Latency is never "the datastore being slow" — it's a missing index. Any new filter-plus-sort combination is a new query path (document store: a new finder or aggregation; SQL: a new `WHERE`/`ORDER BY` pair; ORM: a new finder method).

- Order compound index fields by selectivity, and **lead with the `<scope-key>`** — see `tenancy.md`.
- Paginated list queries include the sort key in the compound index.
- **Include only fields the query actually predicates on.** A compound index carrying a field absent from the query's filter cannot serve it — the index looks present and is dead.
- Per-row fan-out on list endpoints: even with a per-request cache, every unique key resolves through the index — confirm the cache key columns are covered.
- Multiple lookup paths for the same entity need one index each.
- **A sort on a derived or computed field cannot be indexed.** Either persist the field and index it, or bound the result set before sorting. An in-memory sort over the full population is a latent outage, not a slow page.
- Dropping or renaming an index is a deliberate migration — surface it.

**No backfill scripts before production.** With no real data, the empty table *is* the migration. Drop from scope: backfill scripts, recompute services existing only to serve a backfill, dry-run toggles whose only consumer is the backfill, row-mutating migrations (additive schema changes are fine), "idempotent re-runnable" claims that only matter against existing data, cursor-batching introduced solely for the backfill.

Backfill *is* warranted after launch with real customer data, for a schema change that breaks reads on existing rows, or as a one-shot repair after data corruption. When one does ship, the backfill and the real-time write path must produce identical results — one shared function, tested against both entry points.

## Not in application code

**No rate limiting.** That's a gateway/proxy concern. No rate-limit middleware, no in-process buckets, no rate-limit 429s (429 for genuine resource exhaustion is fine). Don't propose a distributed rate limiter as a follow-up either — still the gateway's job.

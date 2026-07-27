# Testing

Rules are ordered by how often the defect actually occurs, most frequent first.

## Seed data

**Every PR that adds a model, status, signal, or scenario updates the seeder in the same PR.** Otherwise test authors later either hand-craft fixtures (multiplying scope across agents) or skip the scenario (hollowing out coverage).

Coverage floor:
- New status value → one record at that status.
- New signal → one entity carrying it, open.
- New service → 1–3 representative records covering its main paths.
- New accumulator or running total → one row at the threshold and one above, so boundary math is visible.
- New relationship → both sides seeded, so a test can exercise the join rather than assuming a row exists.

Seeders are idempotent: upsert by stable natural key, so running twice produces the same state. Reviewers flag any new state with no demo example, and any seeder change that reverses a prior idempotency fix.

**Seed the entities the tests will assert on.** A spec that assumes a related record happens to exist is a flake waiting for the seed to change. Extend the seeder to produce the exact entity the assertion names.

**SEED-MODE — seeder mode is environment-gated once real testers exist:**
- Local and dev stay REPLACE — drop and reseed deterministically; e2e snapshot-restore depends on it.
- Shared and staging go ADDITIVE — upsert-by-natural-key, skip if present, **never delete a row** (it may be user-created), **never overwrite a field** of an existing seed row even if the canonical value changed. New seed records still get inserted on the next deploy. Log every skip so operators see what didn't apply. Every seed-managed table declares its natural key; reviewers verify new ones do.
- A repeated production deploy must not overwrite the administrator account it created the first time. That is the concrete case this rule exists for.

Tests required when introducing additive mode: additive against a user-mutated row leaves it mutated; additive against a missing row inserts it and touches nothing else; replace mode is unchanged.

Gate demo-only seeding behind an explicit environment flag so it cannot run where real data lives.

## Invariant tests over happy-path

For every calculator, projector, validator, and renderer of derived values, ship an invariant companion **in the same PR** that quantifies over the whole population, not one named fixture.

- **Backend:** exercise every mode and branch, asserting cross-field consistency — `rate === 0 ⟺ amount === 0`; `derived === a - b`; `sum(parts) === total`; `part <= whole`.
- **Frontend:** load *all* fixtures (or generate responses property-based) and assert the rendered cross-field equalities; fail if any fixture violates.
- **E2E:** the coverage matrix enumerates *scenarios*, not components — one case per matrix cell (status × configuration × accumulated position × entity type).

The failure this prevents: every layer has complete-*looking* tests for its own scope, while no assertion says "for any rendered row, the body value equals the footer value equals the computed difference." That's how an internally-inconsistent display ships — a label reading 0% next to a non-zero amount.

**Exhaustiveness is part of the invariant.** A check that walks 4 of 15 dependent tables reports the wrong answer confidently. Derive the list from the schema rather than typing it out, so a new table joins the check automatically.

Reviewers hold the PR until the invariant is present.

## Test-id contract

Test ids are a public contract between the product and the e2e suite. Drifting one silently deletes coverage.

**TESTID-SCHEME — one scheme, declared once.** `<surface>-<element>-<variant>`, kebab-case, no indices, no user data, no copy strings. Declare the scheme in the e2e config or a shared ids module and derive ids from a typed builder wherever the set is finite, so a rename is a compile error rather than a red suite. Segments come from the same taxonomy as the component they name — an element under a `filter-picker` is not `filter-trigger`.

**TESTID-STABLE — never key a selector on copy, class name, DOM position, or index.** Text and layout change for design reasons; the selector must not. Renaming or removing a test id is a contract change: grep the suite in the same PR and update every consumer.

**TESTID-COVER — every element the design enumerates carries an id.** Interactive controls, rows, empty states, error states, and loading affordances included — the element-existence assertions below have nothing to bind to otherwise.

## E2E rules

Non-negotiable for every e2e spec:

1. **No file-header preamble comments.** The test names are the documentation.
2. **Config lives in one shared place** — timeouts, viewport, retries, network policy, fixture paths, credentials, env gates in the config file and shared helpers. Specs consume, never redefine. A per-test timeout override is allowed only when one test legitimately needs longer, with justification.
3. **Never "account for network issues."** No catch-and-continue around navigation, fetch, or response waits. Fix the upstream flake instead of papering over it.
4. **Real user actions for everything under test.** Click the control. Don't drive the app by evaluating fetch in the page, don't call the API client. Direct API calls are for the *arrange* phase only (reset a scope, seed an entity).
5. **Hard assertions only.** No soft expectations, no swallowing exceptions around assertions. Failures must be loud — soft-fail patterns become permanent green coverage over broken behaviour.
6. **Tight action timeouts.** UI actions ≤10s (5s default), navigation ≤15s; long waits only for a genuinely long external operation. Anything over 30s otherwise is suspicious. Constants named once in shared config and consumed by name.
7. **After every state-change action, assert backend state before asserting UI.** Poll the entity until it reflects the mutation, then assert the rendering. Catches a non-mutating backend, false-green cached UI, and real UI-lag flakiness. For running-total flows, verify the total after each step before proceeding.
8. **Every assertion has an explicit expected value computed from inputs** — compute the expected number in the spec using the same formula as production, then exact-match. Not a bare visibility check, not a broad regex over formatted output.
9. **Sequential journeys are one continuous-state test**, not N isolated steps with mocked starting state — that's what catches state-bleed. Independent table-driven arithmetic cases legitimately stay separate.
10. **Cover side-effects of user-driven config changes.** For any field the user can edit, assert the downstream derived field actually re-derives. A critical gap, not a nit.
11. **Zero exception swallows** around assertions, actions under test, waits, or post-action state reads. Even a log-and-rethrow is a future place for drift to mute a failure. Cleanup hooks may catch, but must rethrow last.
12. **Arithmetic exhaustively validated.** Every displayed value originating from a formula — totals, rates, balances, computed deadlines — asserted against an expected value computed in the spec. All currency fields, all percentages, all counts, all derived dates. Use the app's own formatting helpers; don't regex the rendered string. Never lift a rendered value and reuse it as the source of truth downstream.
13. **Element existence *and* content, per scenario.** First assert every named element from the design is present, then assert the content each scenario should produce. Missing element = layout/wiring bug; wrong content = data bug — the two-layer assertion makes the failure class diagnosable from the output alone. Walk the canonical scenario set per surface (empty / one / many / loading / error), plus each filter's effect and a clear-resets-all case.
14. **Reset the datastore to pristine post-seed state after every test.** Snapshot once before the run, restore after *every* test, not just between files. Without this, retries compound residue and produce false failures unrelated to product bugs. New specs inherit the reset fixture with no opt-out; specs that hand-clean their own state become removal candidates.
15. **Cover the feature's primary path, not only its edges.** A suite that exercises every filter but never the document upload the product exists to perform is not covering the product.

**TEST-SKIP — the skip count is a baseline that only goes down.** Zero is the target. No new skip, exclusion marker, `todo`, or `fixme` in your diff, and no conditional skip that silently passes when its fixture is absent — a missing fixture is the PR's scope, not a reason to skip. Existing skips are a declared baseline: record the count, and no PR may raise it. Every gate line reports it (`1331/1331 passing, 10 skipped`) so a rise is visible; a rise needs the reason in the body and an owner decision. Claiming zero when it is not zero is worse than the skips.

An environment-gated suite (absent browser binary, absent external credential) may skip only if it prints the skip at WARN, names the missing capability, and is counted in the baseline. A gate that skips quietly is a hole, not a gate.

## E2E execution gates

**Every PR that adds or modifies an e2e spec runs it green before the PR is raised.** No "written but not executed" deferrals — post-merge validation silently becomes never. Missing local fixtures are part of the same PR's scope. This one is a hard gate.

**Running the FULL e2e suite green against a live local stack before raising is the target for every functional-code PR** — a PR that changes behaviour. Not before merge, not just the touched specs:
- New code breaks old specs — a regression in a feature the change didn't touch.
- "Before merge" leaves a window where author and reviewer agree, merge happens, and the suite reveals the bug after the fact.
- "Touched specs only" misses the integration surface.

The suite runs against a live local stack built **from the PR's branch**, not a stale image. Report the result in the PR body with real numbers — pass count, skip count, duration. If you did not run it, say so explicitly rather than omitting the line; a silent omission reads as a pass.

Design-pass PRs, docs, and chore PRs are outside this target — they carry the gates their diff can actually affect. If a pre-existing spec fails on a functional PR, it's now your PR's failure: fix the product, or file it with explicit owner approval citing the issue.

**A PR touching a designed surface ships matching e2e coverage in the same PR:**
- New page → new spec with element-existence plus content-per-scenario assertions for every visible interactive element.
- New component or state → assertion in the parent surface's spec, or a focused unit test if presentation-only.
- Route change → spec covering the redirect and default destination.
- Filter, sort, or search added → a case per filter's effect, plus clear-resets-all.

No "coverage lands in a follow-up." Once the suite is all-green, any PR introducing a new failure is held until fixed in that PR.

**Spec file organisation:** one spec per surface; don't bolt new flows onto the auth spec. Extract a shared login helper the moment a second spec needs it (not before — YAGNI), driving the real UI form rather than mocking. Per-surface specs start from the helper and focus on their own behaviour.

## Test value per PR

When writing a test, ask:
1. Does an existing test cover this behaviour? → extend it (another case in the same suite, another table row).
2. Am I testing real behaviour, or a mock? A test that "the wrapper calls fetch with these args" tests the mock.
3. Would this ever fail for a real bug? If it only fails when someone edits the test setup, delete it.
4. Is this contract already enforced by the type system or schema? Then the schema is the test.
5. Am I duplicating coverage across layers? Pick the highest-value layer (usually integration against a real datastore harness) unless the lower layers cover unique edge cases.

**A test that restates a literal declared beside it is a tautology.** Asserting a constant object has eight keys, next to the eight-key literal, can never fail independently of the source.

New test *files* are the exception — justify in the PR body why no existing file could host it. Reviewers spot-check 3–5 added cases against these questions and flag mock-asserting, constant-restating, and duplicate coverage.

**Subtractive diffs are a positive signal** when a stronger test in the same PR subsumes weaker ones. Celebrate it, don't flag it as lost coverage.

Never cull: coverage minimums for critical business invariants, regression tests tied to historical bugs (load-bearing tribal knowledge), integration tests against a real datastore.

## Every manual bug becomes a test

Any bug-fix PR — however small — includes a test that fails on the pre-fix code and passes after. Write the test first, confirm it fails, then ship the fix.

- The test asserts the displayed or computed value that was manually noticed as wrong, pinned via shared expectation constants when the bug is in a list or dashboard.
- Reviewers request changes on any bug-fix PR without one, regardless of how small the fix.
- The threshold for "this didn't need a test" is essentially zero — even a shipped-wrong copy string gets a content assertion.
- A validator shipped without a case exercising the format it validates is the same gap one step earlier.

Manual testing is for *discovering* new bug classes (UX judgement, design mismatch, performance perception). Once discovered, a class moves into the test infra and stops needing manual re-checks. Manual smoke is never regression coverage: if the automated suite is green and a regression slipped, the suite has a hole — file it before the next phase opens.

## Tests define the product

**Tests are the contract; product is the variable.** A red test is fixed in product code, seed data, or config — never by re-pinning the assertion. Tests encode intent captured during triage and earlier review cycles; re-pinning erases that intent silently.

The only sanctioned spec edits:
- Genuine syntax or typo fixes that block compilation.
- Framework-API misuse that was always wrong and never caught a real bug.
- Stripping a banned pattern (a skip, an exception swallow).

Anything that changes *what* is asserted — value pins, count expectations, selectors implying a different surface — is a product fix, full stop. If a spec references an ID, slug, or entity that doesn't exist, extend the seed to produce that exact entity; don't rewrite the spec to use a different one. If a `>= 1` assertion fails, fix the seeder — don't relax it to `>= 0`.

Reviewers hold any PR that changes an assertion in place of fixing the product.

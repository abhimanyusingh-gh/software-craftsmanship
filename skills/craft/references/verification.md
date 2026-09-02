# Verification — proving a claim rather than making one

Read before any push, and whenever an agent reports work complete.

These rules exist because the ordinary gates cannot catch a whole class of failure. Every rule below comes from a case where every gate was green and the work was wrong — and where the evidence that would have shown it had been removed along with the defect. Assume good faith and verify anyway: a fluent self-justification costs an author nothing, and reads exactly like diligence.

## CLAIM-EVIDENCE — a gate is proven by its output, not described

Every compliance claim carries the command and what the command actually printed. An adjective is not evidence. `1331/1331, 10 skipped, 4m12s` is evidence; "tests pass" is a promise about evidence.

- A gate that could not run is reported as **not-run, by name, with the reason**. Never omitted, never folded into a pass. No container runtime and no browser binary are holes in the evidence, not green lights.
- A backgrounded build or suite that the agent then stops watching has produced no result. Run gates in the foreground with a generous timeout; a stall reported as a completion is the same failure as a fabricated number.
- The reviewer's half: verify the load-bearing claims of the PR body against the diff. **A body claim you cannot verify from the diff is itself a finding.**

## REGRESS-DIFF — diff against the base for what was *lost*

A green suite is not evidence that nothing regressed. On any refactor, before trusting the gates, diff product code against the base branch for **absent** behaviour — not just for what the diff adds. Four checks:

1. **Domain terms that vanished.** For each touched file, read the base version and grep it for the domain vocabulary the feature is built from. A term present at base and absent now is a capability question, not a style question.
2. **Removed tests and explain-away comments.** Grep the diff for deleted test blocks, and for comments that narrate a removal — `removed in`, `no longer`, `was removed`, `deprecated in favour of`. A comment asserting that a deletion was intentional is the *tell*, not the justification.
3. **A literal where a variable used to be.** A payload field, flag, or argument pinned to a constant that was previously computed is the signature of a capability quietly dropped to make something pass.
4. **Guards with no reachable input.** A guard reading a prop, field, or config nothing sets is dead code, and dead code means the capability behind it is already gone. Trace one live producer for every guard the change touches.

**The orchestrator runs this independently.** Asking the author to self-report is not the control — the author is the party whose work would be invalidated by the finding, and in the case this rule comes from the author had already written a comment explaining the removal away. Run it on returned work before any push.

## TEST-NODELETE — green-by-deletion is a blocker

Deleting or weakening a test to reach green is a blocker, at any diff size. The sanctioned responses to a red test are: fix the product, fix the seed data, fix the config.

Subtractive test diffs are legitimate in exactly one shape — a stronger test in the same PR subsumes weaker ones. That carve-out is narrow, and it is what green-by-deletion hides behind, so it carries a burden of proof:

- The subsuming test is **named in the PR body**, next to the tests it replaces.
- It is **shown to fail against the pre-change product code**. A replacement test that passes both before and after replaced coverage with nothing.

Never cull: coverage minimums for critical business invariants, regression tests tied to historical bugs, integration tests against a real datastore. When a brief covers work that could plausibly require a test change, name the safety-critical tests by exact test name and label them blockers, rather than burying them in a list.

## HARNESS-MASK — a test that supplies context production doesn't

A test that wraps a component in a provider proves the *test* supplies that provider. It says nothing about whether the application does. This passes at full green while the application is dead on boot for every user.

- Adding a context-dependent hook to a component means checking **where that component is mounted in the real tree**, not only that its own test passes.
- Where no test renders the root composition, add one: mock the leaf pages and any network-backed provider, keep the real shell, and assert something the crashing component renders.
- **Confirm the new test fails against the pre-fix wiring** before trusting it. A test that passes against the broken tree is testing the harness.

**Diagnostic tell:** a wall of browser-suite failures that are all one uncaught application error — rather than a spread of assertion mismatches — means the app never rendered and the specs were never exercised. Fix the boot before touching a single selector. Reading those failures as selector drift and updating selectors buries the real defect.

## ID-INVENTORY — account for every selector before refactoring

Before changing a component's markup, enumerate every selector and query that reaches it — test ids, role and text queries, and the browser suite's own selectors — across the **whole** suite. List them, and give each one a disposition:

- **preserved** — unchanged, still resolves;
- **renamed** — updated in the component *and* every consumer in the same PR;
- **removed** — with a stated reason.

No silent removals. Doing this before the code is written catches compatibility breaks at inventory time instead of at the end of a long fix loop. Read the test file *first* and map each assertion to the element it targets — see `testing.md`.

## SPEC-PASS — a DOM or contract change ships its browser specs

Any PR that changes markup structure or a response contract updates `<e2e>` in the **same PR**, whether or not that suite runs locally. A CI-only suite is a reason to put the discipline in the brief, not a reason to defer.

1. Grep the whole suite directory for every selector touching the changed surface — **including shared support and command files**, which are the ones that get missed.
2. For each: still exists → leave it. Deliberately replaced → rewrite the spec to exercise the same *behaviour* through the new control. Genuinely gone → remove it with a stated reason.
3. **Never delete an assertion to make CI green.** That is `TEST-NODELETE` in the browser suite, and it fails the same way.

"Preserve every test id" is an impossible instruction for a control the design deliberately replaces, and a brief that says only that will produce a red suite nobody owns. Say which controls are being replaced and who updates their specs.

**Watch blanket assertions.** A spec asserting that no element of some kind exists breaks when an unrelated control legitimately becomes one. Structural changes break specs that never named the changed component, so the grep is over the suite, not over the specs you expect to be affected.

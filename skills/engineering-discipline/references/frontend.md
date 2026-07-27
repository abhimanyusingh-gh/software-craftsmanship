# Frontend

Rules are ordered by how often the defect actually occurs, most frequent first. Interactive components also need `accessibility.md` — a11y findings are correctness findings and belong to the functional pass, not the design pass.

## Design fidelity

**The design source is ground truth — a 1:1 translation target, not an inspiration.** DOM hierarchy, class names, density, content slots, ARIA attributes. When the current frontend and the design diverge, fix the frontend. "A prior PR established this pattern" and "the departure was intentional" are not defences — prior divergences were themselves deferrals, and reconciling them is the point.

The inversion: the design source is the *starting point*. Port it into typed components, then wire data, state, handlers, and queries into that shell. Existing feature files are replaced, not edited-toward-alignment. Stylesheets are regenerated from the design source, not patched — accumulated legacy rules get deleted, and the design's class taxonomy becomes the universe.

Concretely:
- Only design-source class names; port a missing one rather than inventing.
- **Design tokens, never raw colour values.** Fonts from the design system.
- **A token referenced but never defined renders as its fallback, silently and forever.** Grep every custom property you consume for its definition; a missing one usually surfaces as one theme looking right and the other unreadable.
- Specified iconography only — no emoji, no hand-drawn SVG.
- Translate the design source's own inline styles into stylesheet rules.
- Match column counts, casing, density, button placement, and modal structure exactly.
- Light and dark polished equally, and verified separately.
- It's reference, not source — translate it, don't cargo-cult it into production paths.

### No inline CSS

**No inline style objects.** All styling — layout, colour, spacing, typography, shadow, radius — comes from stylesheets referencing design-system tokens, wired via the class attribute. Inline colour literals are equally banned. The only exception is a genuinely dynamic single value.

Inline styles bypass the token system (colour drift between components), defeat the `:focus-visible` / `prefers-reduced-motion` / dark-mode media queries that live in stylesheets, and force component-file edits for what should be CSS edits.

If you reach for an inline style while writing a component, stop: add a class referencing existing tokens, or add the token first and then use it via the class. Inline styles that pre-date your PR but sit on lines this diff touches get replaced under debloat-on-touch.

## Design completeness

Before coding, **enumerate every element**: every page section, tile, button, interaction, state (loading / empty / error / populated), and responsive variant — citing the design source per item.

Then resolve gaps *before* coding:
- Design element needs data the backend doesn't expose → extend the backend in the same PR (preferred), or surface the choice to the owner.
- Design element references a component that doesn't exist → build it in the same PR.

**Substitutions are not permitted** ("that tile's data doesn't exist yet, render a different one"). Surface the gap first. Ship every checklist item; verify the checklist against the implementation before pushing. Deferrals need explicit approval recorded in the PR body.

## Fidelity is a hard gate

The PR body must carry:
1. Side-by-side comparison evidence — design vs implementation, light and dark, every changed surface, deltas annotated.
2. Token audit — every changed CSS rule cites its design-source line, or is a documented additive deviation with rationale.
3. DOM-shape audit — a mapping from design-source structure to implementation structure for the reviewer to line-walk.
4. Interaction parity — hover, active, focused, disabled states match, described explicitly or captured.

**No comparison evidence → reviewer requests changes on the body alone, before reading code.** "Tests pass" and "looks fine in browser" are not fidelity proof. No author-judged "close enough": a different font weight, spacing unit, colour shade, divider thickness, or text casing is drift, and drift gets fixed. If you can't capture the evidence, stop and hand back — don't raise the PR.

## Functional / design split

Every UI-changing feature ships **two PRs by two agents**, because bundling mixes correctness signals with subjective ones, lets design thrash block working features, and demands one agent be expert in both data/state concerns and typography and motion.

**Pass 1 — functional** (ships first): data wiring, state, mutations (optimistic updates, invalidations, rollback), loading/empty/error/retry states, accessibility, behaviour tests. Uses the design system as-is, styled minimally with the simplest token-hitting class. Explicitly excludes typography choices, palette decisions, density tuning, custom component shapes, hover/focus styling, visual snapshots.

**Pass 2 — design** (after Pass 1 merges): typography, tokens, spacing and density, component shapes, hover/focus styling, motion. Stylesheet and class-attribute changes only. Explicitly excludes behaviour changes, type-signature changes, new API calls, and behaviour-test edits. Net-new primitives the design implies get their own issue.

Bundle only when the surface is trivial (1–2 files), the design is silent on it, and no new primitive is implied. If Pass 2 finds Pass 1's structure can't carry the design, Pass 2 **stops and reports** — either restructure Pass 1 or adapt the design. The issue stays open until Pass 2 merges too.

## Server state vs client state

*If the project has no server-state cache library, the rule becomes: one module owns fetching per domain, components never fetch, and the invalidation rules below become explicit refresh calls at the same seams.*

**All server state goes through `<query-lib>`.** Server state = anything fetched from the backend that the UI shows or another component might read. The test: would two callers benefit from a shared cache?

Genuine raw-fetch exceptions:
1. Auth bootstrap before the app mounts (no provider yet).
2. The streaming connection itself — hydrate the entity through `<query-lib>`, invalidate on stream events.
3. Uploads to presigned URLs.
4. File downloads / document opens.
5. Telemetry pings.
6. Logout.
7. Anything outside a component context (service worker, web worker, CLI).

For (1) and (3), wrap the imperative *trigger* in a mutation so retry and error-toast behaviour matches the app, while the underlying call stays raw.

Anti-patterns:
- A cached query on a one-shot non-cacheable URL "because it's the convention" — bloats the cache, can keep expired URLs alive.
- Raw fetch in a component for server state because "it feels simple." Two months later there are five hand-rolled variants.
- Hand-rolling local loading/error/abort state around a fetch — that's a badly-shaped query hook.
- Bypassing `<query-lib>` in a service module for "reusability outside hooks" — the reuse pattern is a shared hook with a parameter.

**Migrate existing raw fetch on touch, never in bulk.** When you're in a file for another reason and it has non-exempt raw fetch, migrate it in that commit. If that blows the PR budget, file a follow-up and note it. Never a dedicated "migrate everything" sweep — blast radius without proportional benefit.

**Client state lives in `<store>`, server state does not.** UI flags, form drafts, modal visibility, theme, collapsed panels, selected filters → store, consumed via narrow selectors, mutated via named actions (not raw setters). Persist only what should survive reload. Server data in the store is an anti-pattern; UI flags in the query cache are too. Flag shared local state, prop-drilling more than two levels, and context that should be a store.

Hook shape: explicit query key, error handling, refetch semantics. Mutations declare success handlers that invalidate the relevant keys so consumers refetch automatically. **Query keys come from one shared key factory per domain, never a literal typed at both the subscription and the invalidation** — see `MIGRATE-KEY` in `workflow.md`.

## State contracts

**STATE-CONTRACT — undefined, empty, and loading are three different states.** Every data surface resolves four states explicitly and separately: initial load, loaded-and-empty, loaded-with-data, error. Absence of data is not a loading check — a careless truthiness guard renders the spinner forever on an empty result, or renders "no results" while the first request is still in flight. Branch on the query's own flags, and render the empty state only once the request has settled with zero rows.

**STATE-REFETCH — a refetch never blanks the screen.** Distinguish first load from background refetch. First load gets the skeleton; a refetch, a filter change, or a poll keeps the current rows mounted with a subdued in-flight affordance. Swapping mounted content for a spinner on every refetch is layout thrash, not a loading state. Paginated and filtered lists keep the previous page's rows during the next fetch. A mutation's invalidation must not unmount the surface the user is looking at.

**STATE-NODELAY — never add a delay with no work behind it.** No timer before a synchronous computation, no artificial minimum spinner duration, no "let it feel like it loaded." If the work is synchronous, render it synchronously. A loading state represents real pending work; a timer that represents nothing is a bug that also masks real latency once the work becomes async.

## Errors

**Errors are toasts, never inline.** One global toast layer at the app root, backed by a queue store. Every failure path — validation, network, 4xx/5xx, optimistic-update revert — surfaces as a toast.

Blocking validation disables the submit control (clear labels, no red-ring-plus-helper-text); if the user submits anyway, toast. Inline is correct only for loading states, empty states, and disabled-reason tooltips. Full-page error boundaries are for genuinely unrecoverable state only — otherwise toast plus retry.

Delete on sight: field-error spans under inputs, inline alert banners.

**A failure path with no toast is the more common defect than a toast that should have been inline.** Walk every branch that can reject — including the ones inside a loop — and confirm each surfaces.

## Bulk and optimistic mutations

**BULK-ATOMIC — a bulk action is one request, or it reports per-item outcomes.** Prefer one endpoint taking N ids and returning a per-item result. A loop of sequential awaits from the client is N independent mutations wearing one button: item 3 failing after 1 and 2 committed leaves the server in a state no client-side rollback can undo.

**BULK-PARTIAL — a partial failure never presents as a total failure.** If the client must fan out, settle all of them — never a short-circuiting loop — then report succeeded and failed counts and name the failures. One "Something went wrong" toast over two successes and one failure is a lie the user then acts on.

**BULK-ROLLBACK — roll back only what did not commit.** An optimistic rollback that restores the whole pre-mutation snapshot after a mid-sequence failure reverts the successful items in the UI while they remain committed on the server; the UI is now wrong until a refetch. Choose one and state which in the PR body: atomic server-side, per-item rollback, or no optimistic update for bulk paths with invalidation on settle.

**BULK-IDEMPOTENT — a retried bulk action does not double-apply.** Retry is the user's first instinct after a partial-failure toast. Key the operation so re-submitting the same set is safe.

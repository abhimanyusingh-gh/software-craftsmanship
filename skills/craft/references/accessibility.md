# Accessibility

A11y findings are correctness findings, not polish. They are in scope for the **functional** pass and never deferred to the design pass — a keyboard-unreachable control is a broken feature, not an unstyled one.

## A11Y-DISABLED — never remove an element from the accessible tree to convey state

A native `disabled` attribute strips the element from the accessible tree: screen readers skip it, it loses focus, and the *reason* it is unavailable goes unannounced. A "Coming soon" label on a natively-disabled item is never read aloud.

For anything the user must still perceive — an unavailable option, a locked row, a not-yet-permitted action, a placeholder item — use `aria-disabled`, keep it focusable, and suppress activation in the handler. Reserve native `disabled` for a submit control whose form is invalid, where adjacent copy carries the reason.

## A11Y-MODEL — one interaction model per widget

Pick one pattern and implement it whole: roles, keyboard behaviour, and selection semantics from the same pattern. Never mix. A listbox with checkbox children, an `option` role carrying `aria-checked`, or a multi-select reporting `aria-selected` on some children and `aria-checked` on others is unusable in a screen reader — the two models disagree about what "selected" means and assistive technology picks one.

- Multi-select list → either `listbox` + `aria-multiselectable` with `option` children using `aria-selected`, **or** a labelled group of native checkboxes. Not both.
- Name the pattern being implemented in the PR body whenever a composite widget is added, so the reviewer can check it against the whole pattern rather than guessing.
- Keyboard support is part of the pattern, not an extra: arrow-key traversal for composites, Enter/Space activation, Escape dismissal, Home/End where the pattern defines them.

## A11Y-ANNOUNCE — announce after the state settles, never before

A live-region message written before the mutation resolves announces an outcome that has not happened — and announces success for a request that then fails.

- Write to the live region in the settled path, success and failure separately.
- Keep the region mounted with a stable politeness setting; a region that mounts at the same moment its first message arrives is often not announced at all.
- Optimistic UI may announce optimistically only if the rollback also announces.
- Long-running work announces start and completion, not every intermediate tick.

## A11Y-FOCUS — visible focus and reduced motion are required, not optional

Every interactive element has a `:focus-visible` style with a real contrast delta **in both themes**; never remove the outline without a replacement. Every transition, animation, autoplay, and parallax is wrapped in a `prefers-reduced-motion: reduce` query with a genuine no-motion path.

Both live in the stylesheet — one more reason inline styles are banned.

## A11Y-FLOOR — the baseline on every UI PR

- Labels bound to controls. Never placeholder-as-label.
- Icon-only controls carry an accessible name.
- Images carry alt text; empty alt for decorative.
- One top-level heading per page, no skipped levels.
- Every dialog is keyboard-reachable, Escape-closable, focus-trapped while open, and returns focus to its trigger on close.
- Hit targets at least 24px.
- Colour is never the sole carrier of meaning — pair it with text, shape, or an icon.
- Contrast verified in light and dark, not assumed from the token name.
- Dynamic content that replaces a focused element moves focus deliberately; never drop focus to the document body.

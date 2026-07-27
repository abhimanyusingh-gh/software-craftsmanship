# claude-skills

Skills for [Claude Code](https://claude.com/claude-code).

## engineering-discipline

Engineering standards for writing production code, slicing and reviewing PRs, authoring tests, running pre-push gates, and merging. Stack-neutral, and **graded against real review findings** — 1,020 pull-request review comments across two production codebases, published in [`EVIDENCE.md`](EVIDENCE.md) with instance counts per rule and an honest list of the claims the data disproved.

Claude loads a short list of always-on non-negotiables, then reads one of nine references on the trigger that matches the task:

| Reference | Covers |
|---|---|
| `tenancy.md` | scope isolation — every read, write, aggregate, storage path and cache key scoped; enforcement at the lowest layer; the negative test |
| `authoring.md` | reuse, layering, dead code vs unwired capability, const-as-enum, complexity caps, naming taxonomy, config-from-day-one, URL providers, comment style, type safety, index-per-query |
| `frontend.md` | design fidelity and tokens, no inline CSS, design completeness as a hard gate, the functional/design PR split, server vs client state, loading and empty state contracts, bulk-operation partial failure, errors as toasts |
| `accessibility.md` | disabled-vs-aria-disabled, one interaction model per widget, announce-after-settle, focus and reduced motion, the per-PR baseline |
| `testing.md` | seed data coverage and modes, invariant tests over happy-path, the test-id contract, 15 e2e rules, the skip baseline, e2e execution gates, test value, manual-bugs-become-tests |
| `defect-classes.md` | the reviewer's entry point — 20 defect classes in observed-frequency order, each with the shape to look for and its severity floor |
| `review.md` | reviewer-per-PR, reviews as prose, fold-all-findings, full-PR re-review, escalating on the pattern not the round count |
| `workflow.md` | work intake, PR slicing, multi-PR migration cutovers, the pre-push gate sequence, browser smoke, merge policy, docs currency |
| `orchestration.md` | agent fleet operating model, worktree isolation, briefs, the multi-session pattern, the post-merge routine, phase sweeps |

Progressive disclosure is the point: `SKILL.md` stays small enough to carry no real context cost, and no single task loads more than three references.

## Install

As a plugin, which gives you versioning and updates:

```
/plugin marketplace add abhimanyusingh-gh/claude-skills
/plugin install engineering-discipline@software-craftsmanship
```

Or symlink it directly, if you'd rather track `main` and edit in place:

```bash
git clone https://github.com/abhimanyusingh-gh/claude-skills.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)/claude-skills/skills/engineering-discipline" ~/.claude/skills/engineering-discipline
```

For a single project rather than every project, link into `<project>/.claude/skills/` and commit it so the whole team picks it up.

Claude invokes the skill on its own when a task matches the description; `/engineering-discipline` invokes it explicitly.

## Adapting it

**Bind the placeholders.** `SKILL.md` declares a `Project bindings` block — the task runner, the type-checker, the test and build commands, the dead-code check, the shared package, the query library and store, the datastore, the tracker, and the field that partitions your data between customers. Claude derives these from your repo on first use and records them in your `CLAUDE.md`. Nothing in the references hardcodes a toolchain; the one concrete gate sequence in `workflow.md` is explicitly labelled as an example to derive from.

**Check the evidence before you accept a rule.** [`EVIDENCE.md`](EVIDENCE.md) grades every rule family by how many independent review findings back it and across how many PRs. Rules near the bottom of that table — no pre-production backfill, no application-level rate limiting, branded types, never-re-pin-an-assertion — are deliberate positions with thin or no supporting data. Everything in `orchestration.md` is an operating model with no evidence either way, and says so at the top of the file. Cut what doesn't fit your context; the high-weight rules are the ones worth keeping.

**Read the reversals.** The same file lists seven claims this skill made that its own history contradicted, and what changed as a result. If you are evaluating whether these standards are worth adopting, that section is more informative than the confirmations.

## License

MIT

# Publish checklist

Run before every publish. Genericization rots silently — a rule rewritten in a hurry re-introduces the origin project's toolchain, and nobody notices until a stranger loads the skill against a Python repo.

## 1. Genericization greps

Every pattern below must return **zero hits inside `skills/`**, except where the hit sits in a block explicitly labelled as an example.

```bash
cd skills/

# toolchain — the gate sequence in workflow.md carries ONE labelled example block; nothing else may
grep -rnE 'yarn|npm run|pnpm|npx|\bknip\b|tsc --noEmit' .

# test and browser frameworks by name or API
grep -rniE 'jest|vitest|playwright|cypress|xdescribe|test\.skip|page\.evaluate' .

# datastore by vendor or operation name
grep -rniE 'mongo|mongoose|postgres|prisma|findOne|\.lean\(|\$match|\$group' .

# frontend libraries by name or hook shape
grep -rniE 'react-query|tanstack|useQuery|useMutation|zustand|redux|express|router\.(get|post)' .

# origin domain residue
grep -rniE 'statutory|fiscal|tds|gst|invoice|vendor|tax|refund|regulator|ledger' .

# internal identifiers, incidents, and nicknames
grep -rnE 'RB-[0-9]|PR #?[0-9]+|issue #[0-9]+|chaos suite|AB#[0-9]+' .

# the employer, the real repository and product names, colleagues, ticket prefixes and
# internal hostnames — substitute your own, keep the literals out of this file's history
grep -rniE 'employer|name-one|name-two|colleague-one' .
grep -rnE '\b[A-Z]{2,6}-[0-9]+\b' .

# absolute and home-relative paths
grep -rnE '/Users/|/home/|~/' .
```

Known-benign hits, so a future maintainer doesn't chase them:

- The gate sequence in `workflow.md` carries one `yarn`/`tsc` block under an explicit *"Example — derive your own"* label. That block is the only permitted toolchain reference.
- `defect-classes.md` ships reviewer greps that necessarily name concrete skip markers. Its heading says "adapted to the project's language".
- `authoring.md` and `defect-classes.md` use "vendor" in the plain-English sense of a third-party supplier, and `authoring.md` quotes `"PR 2 follow-up"` as an example of a *banned* test name.
- The origin-domain grep's `tax` pattern collides with "taxonomy", which several references use for naming and test-id schemes. Substring noise, not residue.
- `accessibility.md` names web-platform APIs (`aria-disabled`, `:focus-visible`, `prefers-reduced-motion`). Those are platform standards, not a stack choice.
- `verification.md` quotes the explain-away comment phrases it tells reviewers to grep for ("removed in", "no longer"). Those are the detection strings, not residue.

## 2. Manifests

```bash
claude plugin validate . --strict
```

Then a real install round-trip from a scratch directory:

```
/plugin marketplace add /path/to/software-craftsmanship
/plugin install craft@software-craftsmanship
/reload-plugins
/craft
```

Confirm `SKILL.md` loads and at least one reference resolves by relative path.

## 3. Rule IDs, both directions

```bash
# every ID used in a reference
grep -rhoE '\b[A-Z0-9]+-[A-Z]+\b' skills/craft/references/ | sort -u

# every ID claimed in the evidence table
grep -oE '`[A-Z0-9]+-[A-Z*]+`' EVIDENCE.md | tr -d '`' | sort -u
```

Every named rule has an evidence row; every evidence row names a rule that exists.

Four known-benign non-rule matches, so a future maintainer doesn't chase them. The `[A-Z0-9]+-[A-Z]+` pattern cannot tell a rule ID from illustrative text:

- `FOLD-THEN` — `review.md` quotes `Verdict: FOLD-THEN-SHIP` as an example of the *banned* compliance-form review style.
- `51-LOC` — a numeric example in `review.md` illustrating the function-length threshold.
- `MUST-FIX`, `SHOULD-FIX` — `EVIDENCE.md` quotes an external automated reviewer's seven-level severity vocabulary. This skill's own vocabulary is the lowercase three, defined in `SKILL.md`.

## 4. The contract card is a return format

`SKILL.md`'s card is the skill's only enforcement surface, so check it still behaves like one:

- Every row asks for a command, a count, or a named symbol — never a yes/no or an adjective.
- `NOT RUN: <reason>` is a legal value for a gate row. Silence is not.
- No rule stated in the card is *also* stated as a separate always-on instruction. The card replaced the paste-13-rules-verbatim brief block; if both exist, briefs bloat again and the invariants get buried, which is the failure it was written to fix.

## 5. Duplicate instructions

The worst failure mode for an instruction document is one rule stated twice in divergent wording — the model then follows whichever it read last. For each rule family, grep its key noun and confirm exactly **one** authoritative statement plus, at most, cross-references pointing at it.

```bash
for n in skip index scope-key "cache key" reset toast token seeder comment \
         "reused symbol" inline focus-visible "test id" verbatim isolation; do
  echo "== $n"; grep -rni "$n" skills/craft/ | head -20
done
```

## 6. Size and frontmatter

```bash
wc -l skills/craft/SKILL.md skills/craft/references/*.md

# description must stay under the 1536-char frontmatter cap
awk '/^description:/{print length($0)}' skills/craft/SKILL.md
```

`SKILL.md` stays under ~110 lines — it carries the non-negotiables and the contract card, and both are pasted into briefs, so growth there is felt everywhere. No single reference should exceed ~180.

## 7. Behavioural smoke — the only real test

Three runs in fresh sessions against a repo on a **different stack from the origin** (Python or Go).

**Does it load at all?** Ask for a code change *without naming the skill*. It must load `craft` unprompted. This is the first thing to break — a description that drifts back toward a comma-run of twenty topics stops matching, and every other rule in here becomes moot because nothing reads them.

**Does it route?** Ask it to review a diff. It must:

- derive the bindings from that repo rather than assuming a toolchain,
- emit no reference to a package manager, type-checker, datastore, or frontend library the repo doesn't use,
- route correctly: a query touching customer data pulls `tenancy.md`, an interactive component pulls `accessibility.md`, a review pulls `defect-classes.md` first.

**Does the card hold?** Give a subagent a refactor whose easiest path is deleting a failing test. It must refuse, and return a card whose `Tests deleted` and `Capability check` rows are filled with real output. Seed the check first: delete a test by hand, run the `REGRESS-DIFF` sequence from `verification.md`, and confirm it actually surfaces the deletion. A check that cannot catch a planted failure is not a check.

## 8. Version

Bump `version` in `.claude-plugin/plugin.json`. It is the only place the version is set — `plugin.json` silently wins over the marketplace entry — and installed users receive nothing until it changes.

## 9. Repository conventions — do not "fix" these

**`main` requires a pull request from everyone, including admins.** Zero approvals are
required, so the merge click is the approval and there is no self-approval deadlock.
Force pushes and branch deletion are blocked. Deliberate: nothing reaches `main`
without a PR someone explicitly merges.

**Squash-merge commits carry `GitHub <noreply@github.com>` as committer, and the
account's display name as author.** That is a consequence of merging through the
platform, and it is accepted. Do **not** rewrite history, force-push, or lift branch
protection to make committer lines uniform — the protection is worth more than the
cosmetic consistency. Author commits locally under the repository's configured
identity and let the merge commit be whatever the platform stamps.

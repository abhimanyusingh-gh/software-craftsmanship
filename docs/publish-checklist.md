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

# the real repository and product names — substitute your own, keep them out of this file's history
grep -rniE 'name-one|name-two' .

# absolute and home-relative paths
grep -rnE '/Users/|/home/|~/' .
```

Known-benign hits, so a future maintainer doesn't chase them:

- The gate sequence in `workflow.md` carries one `yarn`/`tsc` block under an explicit *"Example — derive your own"* label. That block is the only permitted toolchain reference.
- `defect-classes.md` ships reviewer greps that necessarily name concrete skip markers. Its heading says "adapted to the project's language".
- `authoring.md` uses "vendor" in the plain-English sense of a third-party supplier, and quotes `"PR 2 follow-up"` as an example of a *banned* test name.
- `accessibility.md` names web-platform APIs (`aria-disabled`, `:focus-visible`, `prefers-reduced-motion`). Those are platform standards, not a stack choice.

## 2. Manifests

```bash
claude plugin validate . --strict
```

Then a real install round-trip from a scratch directory:

```
/plugin marketplace add /path/to/software-craftsmanship
/plugin install engineering-discipline@software-craftsmanship
/reload-plugins
/engineering-discipline
```

Confirm `SKILL.md` loads and at least one reference resolves by relative path.

## 3. Rule IDs, both directions

```bash
# every ID used in a reference
grep -rhoE '\b[A-Z0-9]+-[A-Z]+\b' skills/engineering-discipline/references/ | sort -u

# every ID claimed in the evidence table
grep -oE '`[A-Z0-9]+-[A-Z*]+`' EVIDENCE.md | tr -d '`' | sort -u
```

Every named rule has an evidence row; every evidence row names a rule that exists.

## 4. Duplicate instructions

The worst failure mode for an instruction document is one rule stated twice in divergent wording — the model then follows whichever it read last. For each rule family, grep its key noun and confirm exactly **one** authoritative statement plus, at most, cross-references pointing at it.

```bash
for n in skip index scope-key "cache key" reset toast token seeder comment; do
  echo "== $n"; grep -rni "$n" skills/engineering-discipline/ | head -20
done
```

## 5. Size and frontmatter

```bash
wc -l skills/engineering-discipline/SKILL.md skills/engineering-discipline/references/*.md

# description must stay under the 1536-char frontmatter cap
awk '/^description:/{print length($0)}' skills/engineering-discipline/SKILL.md
```

`SKILL.md` stays under ~80 lines. No single reference should exceed ~180.

## 6. Behavioural smoke — the only real test

Load the skill in a fresh session against a repo on a **different stack from the origin** (Python or Go), and ask it to review a diff. It must:

- derive the bindings from that repo rather than assuming a toolchain,
- emit no reference to a package manager, type-checker, datastore, or frontend library the repo doesn't use,
- route correctly: a query touching customer data pulls `tenancy.md`, an interactive component pulls `accessibility.md`, a review pulls `defect-classes.md` first.

## 7. Version

Bump `version` in `.claude-plugin/plugin.json`. It is the only place the version is set — `plugin.json` silently wins over the marketplace entry — and installed users receive nothing until it changes.

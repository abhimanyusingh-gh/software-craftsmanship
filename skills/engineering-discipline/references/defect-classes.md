# Defect classes, by observed frequency

The reviewer's entry point. Walk this list in order — it is a prior, not a checklist to tabulate. Report only the classes that produced a finding and say nothing about the rest.

Severity ceilings are defaults; a class can escalate on impact but never silently drop below its floor.

| # | Class | The shape to look for | Severity | Rule |
|---|---|---|---|---|
| 1 | Scope key dropped | a filter, update, delete, aggregate, export, or blob path without the full composite scope key; scope read from a client-controlled parameter | blocker | `tenancy.md` |
| 2 | Duplicated logic | the same parsing, validation, formatting, error handling, or state update in 2+ places; two hooks or services hitting one endpoint with disagreeing shapes | 2× should-fix, 3× request-changes | `authoring.md` reuse |
| 3 | Layering violation | a component importing the API layer past its use-case hook; a provider or vendor name branched on inside a pipeline step; one method orchestrating every strategy | should-fix | `LAYER-*` |
| 4 | Design drift | raw colour values where a token exists; a consumed custom property never defined; invented class names; density, casing, or column count off the design source; one theme unpolished | should-fix, request-changes with no evidence in the body | `frontend.md` fidelity |
| 5 | Gate evidence missing | no gate numbers in the body, a scoped test run presented as the gate, a UI PR with no browser smoke, a skip count that rose | should-fix | `workflow.md` gates |
| 6 | Dead export **vs** unwired capability | an added export with no in-PR import site — then classify: scaffolding is deleted, spec-required capability is wired. A deleted integration is the failure mode here, not the fix | should-fix; deleting a capability is a blocker | `DEAD-EXPORT`, `DEAD-CAPABILITY` |
| 7 | Raw literal / casing split | enum-like literals repeated across files; a component re-typing a union the design system already exports; one concept spelled two ways or cased two ways | should-fix | `ENUM-*`, `NAME-CASING` |
| 8 | Accessibility | native `disabled` used to convey state; a widget mixing two interaction models; a live region written before the mutation settles; missing `:focus-visible` or reduced-motion path | should-fix | `accessibility.md` |
| 9 | Seed coverage | a new model, status, signal, or relationship with no demo row; a seeder that overwrites or deletes in additive mode; a reversed idempotency fix | should-fix | `testing.md` seed data |
| 10 | Complexity breach | a threshold from the table crossed with no proposed decomposition; a god method that has to be edited for every new case | should-fix | `authoring.md` complexity |
| 11 | Loading / empty / refetch | a truthiness guard conflating absent with pending; a refetch that blanks mounted content; an artificial delay with no work behind it | should-fix | `STATE-*` |
| 12 | Naming and test-id drift | a selector keyed on copy, class, or index; a test id whose segments disagree with the component taxonomy; a field renamed silently between layers | should-fix | `NAME-*`, `TESTID-*` |
| 13 | Missing covering index | a new filter-plus-sort with no index; compound fields out of selectivity order or missing the scope key; an index carrying a field the query never predicates on; a sort on a derived field | should-fix; unbounded in-memory sort is a blocker | `INDEX-COVERING` |
| 14 | Happy-path-only test | a new calculator, projector, validator, or renderer of derived values with no population-wide invariant; a check that walks a subset of the tables it claims to cover; a tautology restating an adjacent literal | should-fix | `testing.md` invariants |
| 15 | Server state outside the cache | raw fetch for cacheable server data; hand-rolled loading/error/abort state; server data in the client store; a query key typed as a literal rather than built by the factory | should-fix | `frontend.md` state |
| 16 | Cross-PR contract drift | an invalidation naming a key no subscriber uses; dual-mounted shapes that disagree; an old shape deleted with a live consumer still on it | dead key should-fix; deleted-with-consumer is a blocker | `MIGRATE-*` |
| 17 | Bulk partial failure | sequential awaits over N ids from the client; a short-circuiting loop; a whole-snapshot rollback after a mid-sequence failure; one generic error toast over a partial success | should-fix | `BULK-*` |
| 18 | Docs not shipped | a route contract, env var, schema, index, or operator-facing behaviour changed with no docs diff; a runbook citing a line number the diff moved | should-fix | `workflow.md` docs |
| 19 | Magic number | a bare numeric literal that could change with regulation, customer need, or ops tuning — weights and scoring factors especially; a local array re-listing what a typed constant already enumerates | should-fix | `CONFIG-*` |
| 20 | Narrative comment | prose restating the code; a comment explaining what an enum value means; a test-file invariants docstring; ceremony added by a fold agent | nit | `authoring.md` style |

## Active hunting, not passive scanning

Don't accept "looks fine". For each class above, form the specific question and answer it from the diff. Every flag proposes a concrete fix — extract this helper, lift to a table, wrap in an options object, add this index — never just a diagnosis.

Useful greps, adapted to the project's language:

```
: string[,;]                      # semantic values typed raw (minus message|description|name)
=== "[a-z_]+"                     # enum-like literals; repeating across 2+ files is a flag
\bfetch\s*\(\s*["']/              # raw paths bypassing URL providers
<img[^>]+src=["']/                # same, in markup
style=\{\{|style="                # inline CSS
\.skip\(|\.todo\(|xit|xdescribe   # skip markers — compare against the declared baseline
try\s*\{                          # exception swallows in e2e — each hit needs justification
disabled(=|\s|>)                  # candidates for aria-disabled
```

A grep hit is a lead, not a finding. Read the surrounding code before reporting it.

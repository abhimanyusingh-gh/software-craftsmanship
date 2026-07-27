# Scope isolation

The **scope key** is whatever partitions rows between customers — tenant id, org id, workspace id, realm, account id. Bind `<scope-key>` from the project and read every rule below against it. Projects with a nested partition (tenant → client org) have a *composite* scope key: every rule applies to the whole composite, not the outer half.

Read this before writing the query, not after review flags it.

## SCOPE-QUERY — every read and write carries the scope key

Every query, update, delete, aggregate, and count filters on the scope key. Not "the caller already checked" — the filter is on the query. A query object built without it is a cross-customer data leak: **blocker**, never a nit, never a follow-up.

- The scope key is the first field of the filter and the first field of the covering index.
- Aggregations match on the scope key in the first stage, before any join or lookup. A join into an unscoped table re-widens the query — scope the joined pipeline too.
- Bulk, batch, export, report, and admin paths are not exempt. An unscoped bulk write is the worst version of this bug, not an acceptable one.
- With a composite scope key, filtering on the inner half alone is still a leak. `{ clientOrgId }` without `{ tenantId }` trusts that inner ids never collide across tenants.
- Cross-scope reads exist only as an explicitly named capability (`listAllScopesForSupport`), one function, authorization checked at its entry — never an optional parameter that defaults to unscoped.

## SCOPE-REPO — enforce at the lowest layer that can

Do not rely on N call sites remembering. Put the invariant where it cannot be forgotten:

- a repository layer taking the scope as a required first argument and injecting the filter; or
- a pre-save / pre-query hook on every model owning a scope key; or
- a schema-required scope field plus a write guard that throws when it is absent.

**Enumerate every model that carries a scope key and check the whole list.** Spot-fixing only the model in the diff is how this defect survives — the sweep is the fix. Say in the PR body how many models you enumerated and how many needed the guard.

## SCOPE-REGRESS — a re-introduced scope violation is a blocker

Once review has flagged a dropped scope key on a file, every later round re-checks that file specifically. Re-introducing it is a blocker, and the fold must add the structural guard from `SCOPE-REPO` — not just re-add the filter to the one query. A defect that comes back is evidence the fix was at the wrong layer.

## SCOPE-RESOLVE — one resolver, at the edge

Scope is resolved once, in middleware, from the authenticated principal — never from a request body, query string, or path segment the client controls. A handler that accepts a scope id as a parameter is accepting a client-supplied authorization decision.

- Downstream code reads the resolved scope from request context and never re-derives it.
- **Reject rather than default** when resolution fails. No "fall back to the first org", no "if absent, use the only one".
- Where a path segment does name a scope, a dedicated guard asserts the authenticated principal owns that segment before the handler runs — resolution and ownership are two checks, not one.
- The resolved scope is stamped onto every created record at the boundary, not by each writer remembering to pass it.

## SCOPE-PATH — object storage, file paths, and cache keys are scoped too

Blob keys, upload prefixes, export filenames, and cache keys embed the scope key: `uploads/<scope>/<entity>/<id>`, never `uploads/<entity>/<id>`. Ownership is verified from the path's scope segment on read, not from the requested id alone. A signed URL over an unscoped path is the same leak with a longer half-life.

Cache and query keys include the scope key too, or one customer's response is served to another on a shared key.

## SCOPE-TEST — the negative test is the test

Every scoped surface ships a test that seeds two scopes and asserts the second cannot read, list, count, update, or delete the first's rows — per surface, not once per project. A positive test proves the feature; only the negative test proves the isolation.

- Seed the second scope with data that *would* match the filter if the scope key were dropped. A negative test against an empty second scope passes with the bug present.
- Cover the list endpoint, the detail endpoint, every mutation, every export, and every aggregate separately — they are separate queries and they fail independently.
- Dependency and cascade checks are exhaustive or they are wrong: counting 4 of 15 dependent tables reports "safe to delete" for a record that is not.

# Cross-repo navigation questions — CodeGraphContext (CGC) over the harness corpus

Corpus: `cgc/corpus/harness/workspace/{api-service,shared-lib,web-frontend}` (+ sibling
worktree `workspace/web-frontend.worktrees/wt1`, which must stay out of the index).

Indexed with:

```
cgc context mode global
cgc index corpus/harness --summarize --no-progress
```

CGC auto-detected the nested git repos under `workspace/` and split the single
`index corpus/harness` invocation into four `Repository`/`Project` entries: `harness`
(the parent, no code files of its own), `api-service`, `shared-lib`, `web-frontend`.
All queries below were run with `cgc query "<cypher>"` against that index.

For each question: **Expected answer** (based on reading the corpus source), then
**Confirmed?** — whether step 1/2's actual indexing let us verify it via the CGC graph,
with the query/evidence used.

---

## 1. Does `api-service` reference `shared-lib`?

**Expected:** Yes — `api-service/scripts/apply_discount.py` and
`api-service/scripts/validate_users.py` both do `from shared_lib import ...`, and
`shared-lib` is a separate git repo under `workspace/`.

**Confirmed: YES**, at both the import and the function-call level.

- `IMPORTS` edge: `apply_discount.py` / `validate_users.py` → `shared_lib` (module node,
  `b.path` is `null` — the import statement itself resolves to a placeholder module node,
  not directly to a File).
- `CALLS` edge (function-level, fully resolved across the repo boundary):
  ```
  cgc query "MATCH (f:Function)-[:CALLS]->(g) WHERE f.path CONTAINS 'api-service' RETURN f.name, f.path, g.name, g.path"
  ```
  returned, among others:
  - `apply_seasonal_discount` (api-service/scripts/apply_discount.py) `CALLS`
    `calculate_discount` (shared-lib/shared_lib/pricing.py)
  - `filter_valid_emails` (api-service/scripts/validate_users.py) `CALLS`
    `is_valid_email` (shared-lib/shared_lib/validation.py)

  Both target `g.path` values point at files under `workspace/shared-lib/...`, i.e. CGC
  resolved the cross-repo Python call to the real function definition in the other repo.

## 2. Does `web-frontend`'s HTTP call resolve to an `api-service` endpoint?

**Expected:** Yes, by convention/route string — `web-frontend/src/api/client.ts` calls
`fetch(`${API_BASE_URL}/api/products`)` and `POST ${API_BASE_URL}/api/orders`, matching
`ProductsController`/`OrdersController`'s `[Route("api/[controller]")]` in
`api-service/src/ApiService/Controllers/*.cs`.

**Confirmed: NO — not via the CGC graph.** This is a cross-language (TypeScript → C#)
relationship expressed only as a URL string plus ASP.NET routing convention; CGC has no
node/edge type for "HTTP call resolves to controller route", and no relationship type in
this graph spans `web-frontend` and `api-service`:
```
cgc query "MATCH (a)-[r]->(b) WHERE a.path CONTAINS 'web-frontend' AND b.path CONTAINS 'api-service' RETURN a.name, type(r), b.name"
```
returned `[]`. The relationship is real (confirmed by manually reading both files) but
**not** something this index can surface — it would need to be verified by other means
(e.g. an OpenAPI-spec diff or a route-string matcher), not `cgc query`.

## 3. Is content from `web-frontend.worktrees/wt1` excluded from the index?

**Expected:** Yes — `corpus/harness/.cgcignore` has `*.worktrees/`, and the worktree is a
non-canonical sibling of the real `web-frontend` git repo.

**Confirmed: YES.**
```
cgc query "MATCH (f:File) WHERE f.path CONTAINS 'worktrees' RETURN f.path"
```
returned `[]`. The full file list (`MATCH (f:File) RETURN f.path`, 29 rows) contains only
paths under `workspace/api-service/`, `workspace/shared-lib/`, `workspace/web-frontend/` —
none under `workspace/web-frontend.worktrees/`.

## 4. Are the harness root's own loose top-level files excluded, and is only `workspace/**` indexed?

**Expected:** Yes — `corpus/harness/.cgcignore` starts with `*` (ignore everything) then
`!workspace/` / `!workspace/**` to re-include only the workspace tree.

**Confirmed: YES.** The 29 scanned files are all under `workspace/`; the harness root's own
`.cgcignore`/`.gitignore` are not present as `File` nodes. The `harness` `Repository` node
exists (CGC records the parent root it was pointed at) but has no source files scanned
from directly under it — all real content is one level down.

## 5. Does `shared-lib` know about (import/call into) `api-service` or `web-frontend`?

**Expected:** No — `shared-lib` is a leaf dependency; it should have zero outgoing
references into the other two repos.

**Confirmed: YES (zero found, as expected)** — the `IMPORTS`/`CALLS` dump for the whole
graph (`MATCH (a)-[:IMPORTS]->(b) ...`, `MATCH (f:Function)-[:CALLS]->(g) ...`) shows
`shared-lib/shared_lib/{__init__,pricing,validation}.py` only importing stdlib (`re`) and
its own sibling modules (`.validation`, `.pricing`); no edge originates from `shared-lib`
into `api-service` or `web-frontend` paths.

## 6. Does indexing `corpus/harness` as one root pick up all three repos, or does each need indexing separately?

**Expected:** Unclear before running it — CGC's CLI takes a single path argument, but it
was unknown whether nested `.git` roots would be auto-split or flattened into one
`Repository`.

**Confirmed: YES, auto-split.** A single `cgc index corpus/harness` produced four entries
in `cgc list`: `harness`, `api-service`, `shared-lib`, `web-frontend`, each with its own
`Repository` node and correct per-repo file scoping (see question 1's cross-repo edges,
which only work because CGC still resolved calls *across* those separately-tracked
repos). Indexing the workspace root once was sufficient; separate per-repo `cgc index`
calls were not necessary.

# Cross-repo navigation questions

Concrete questions against the harness corpus (`corpus/harness/workspace/{api-service,shared-lib,web-frontend}`),
each with the expected answer and whether the **unmerged, one-`graph.json`-per-repo** strategy
(as extracted by `graphify extract <repo>` run separately per repo) can answer it.

**Graph location convention:** each repo's graph output no longer lives inside that repo. It is
extracted (or relocated after a default extract) into the harness repo's own output tree, under
`<harness>/graphify-out/<repo-name>/graph.json` — e.g. `corpus/harness/graphify-out/api-service/graph.json`
instead of `corpus/harness/workspace/api-service/graphify-out/graph.json`. All example commands below
that previously ran "inside `<repo>/`" now pass `--graph <harness>/graphify-out/<repo-name>/graph.json`
explicitly instead.

Corpus facts used below:
- `api-service/scripts/sync_shared_lib.py` does `from shared_lib import add_numbers, slugify`,
  reaching across to `shared-lib/shared_lib/utils.py`.
- `web-frontend/src/components/GreetingCard.tsx` calls `fetch(`http://localhost:5000/api/greeting/${name}`)`.
- `web-frontend/src/components/HealthStatus.tsx` calls `fetch('http://localhost:5000/api/health')`.
- `api-service` exposes `GET /api/greeting/{name}`, `POST /api/greeting` (`GreetingController`) and
  `GET /api/health` (`HealthController`).

## 1. Does `api-service` reference `shared-lib`?

**Expected answer:** Yes — `sync_shared_lib.py` imports `add_numbers` and `slugify` from `shared_lib`.

**Answerable with unmerged per-repo graphs?** Partially.
`graphify query "does api-service use shared-lib?" --graph corpus/harness/graphify-out/api-service/graph.json`
finds the edge `sync_shared_lib.py --imports_from--> shared_lib` (see
`corpus/harness/graphify-out/api-service/graph.json`).
So the graph confirms *that* api-service imports something named `shared_lib`. It does **not** confirm
*what* shared-lib actually exports: the `shared_lib` node in api-service's graph is a bare unresolved
stub (`src=`, `loc=` empty) — it is not the same node as `shared_lib_utils_add_numbers` /
`shared_lib_utils_slugify` that live in `corpus/harness/graphify-out/shared-lib/graph.json`. Answering "does
`add_numbers` still exist and match the call site's usage" requires opening the second repo's own
graph (or source) separately; the two graphs share no node IDs.

## 2. Does `web-frontend`'s HTTP call resolve to an `api-service` endpoint?

**Expected answer:** Yes — `GreetingCard.tsx` calls `GET /api/greeting/{name}`, which matches
`GreetingController.GetGreeting()`'s route (`[Route("api/greeting")]` + `[HttpGet("{name}")]`), and
`HealthStatus.tsx` calls `GET /api/health`, matching `HealthController.GetHealth()`'s route
(`[Route("api/health")]` + `[HttpGet]`).

**Answerable with unmerged per-repo graphs?** No.
`corpus/harness/graphify-out/web-frontend/graph.json` has no node or edge for the literal strings
`api/greeting`, `api/health`, or `localhost:5000` — the AST extractor records
`GreetingCard()`/`handleSubmit()` and `HealthStatus()` as callable nodes with only a structural
`contains` edge, not the `fetch(...)` call target or its URL argument. `graphify query "does
GreetingCard call the api-service greeting endpoint?" --graph
corpus/harness/graphify-out/web-frontend/graph.json` returns a single isolated node (`GreetingCard()`,
0 edges) — confirmed empirically. Likewise `corpus/harness/graphify-out/api-service/graph.json` has no
node for the route string itself, only method names (`.GetGreeting()`, `.GetHealth()`) and the bare
`HttpGet`/`HttpPost` attribute names (no route path attached). Matching the two sides requires a
human/LLM to read both sources side by side and reason from naming convention (`GreetingCard` ~
`GreetingController`, `HealthStatus` ~ `HealthController`) — the graph structure alone cannot make this
link.

## 3. Can a query scoped to one repo's graph see the other repo's nodes?

**Expected answer:** No, by design of the unmerged strategy.

**Answerable with unmerged per-repo graphs?** Confirmed empirically.
`graphify query "does GreetingCard call the api-service greeting endpoint?"` run with
`--graph corpus/harness/graphify-out/web-frontend/graph.json` only ever traverses that one file's 90
nodes; it has no way to reach `corpus/harness/graphify-out/api-service/graph.json`'s 48 nodes. The same
holds for the MCP server (`graphify-mcp`, see `graphify.serve`): each tool call resolves an optional
`project_path` argument to `<project_path>/graphify-out/graph.json` and loads/selects exactly one graph
context per call (`_select_graph`); it does not merge or cross-reference multiple projects' graphs even
though it can hold several in its LRU cache. With graphs now relocated into the harness, `project_path`
for a given repo must resolve to `<harness>/graphify-out/<repo-name>` (not the repo's own path) for this
lookup to find its graph. Cross-repo answers require either (a) the caller to make two separate
calls/queries, one per repo, and reconcile the answers itself, or (b) `graphify merge-graphs` /
`graphify global add` to build one merged graph up front — a different, explicitly-merged strategy
outside what this ticket evaluates.

## 4. Did `.worktrees` content leak into any repo's graph?

**Expected answer:** No.

**Answerable with unmerged per-repo graphs?** Confirmed empirically — see
[report/worktrees-and-mcp-findings.md](../report/worktrees-and-mcp-findings.md).
Neither `corpus/harness/graphify-out/` (harness root — never fully extracted; its cache only indexed
the root `README.md`) nor any of the three repos' relocated `manifest.json` / `graph.json` under
`corpus/harness/graphify-out/<repo-name>/` contain the substring `worktrees`.

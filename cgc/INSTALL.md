# Installing CodeGraphContext (CGC) for cross-repo navigation

This is a from-scratch install/config guide for CodeGraphContext (CGC), generalized
beyond the synthetic corpus in `cgc/corpus/harness/` to any set of real
`harness/workspace/*` repos. It documents what was actually verified working in this
sandbox (Python venv + FalkorDB Lite, `global` context mode).

## Prerequisites

- Python 3.10+ (verified with 3.12.3). No system Neo4j/Redis/Docker required for the
  FalkorDB Lite path below — it's an embedded, in-process graph DB.
- `git` repos to index (CGC discovers nested `.git` roots automatically — see
  "Indexing a multi-repo workspace" below).

## 1. Install

Use an isolated virtualenv rather than installing into the system Python:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install codegraphcontext
```

This pulls in `FalkorDB` (Python client) and `falkordblite` (embedded FalkorDB server)
as dependencies automatically — no separate `pip install falkordblite` needed.

Verify the CLI is on `PATH` (inside the venv) and everything is healthy:

```bash
cgc doctor
```

Expect all five checks (config, database connection, tree-sitter, file permissions, CGC
command) to pass. If "Database Connection" fails here, see the FalkorDB Lite section
below before doing anything else — indexing will fail the same way.

## 2. Context mode: `global`

CGC supports three context modes (`global`, `per-repo`, `named`). For cross-repo
navigation across a multi-repo workspace, use `global` so every indexed repo lands in
one shared graph and cross-repo edges (imports/calls) can resolve between them:

```bash
cgc context mode global
```

Confirm with `cgc context list`. This setting is persisted in
`~/.codegraphcontext/config.yaml` and applies to all subsequent `cgc index` calls until
changed.

## 3. Database backend: FalkorDB Lite

`falkordb` is already the default `DEFAULT_DATABASE` for a fresh install
(`cgc config show`). FalkorDB Lite runs embedded — no separate server process, no
Docker, no network — it manages its own on-disk store under
`~/.codegraphcontext/global/db/falkordb` (path shown by `cgc config show` as
`FALKORDB_PATH`). In this sandbox it worked out of the box with **no extra setup**: no
`falkordblite` server needed to be started manually, and `cgc doctor` reported
`✓ FalkorDB Lite is installed` / `✓ FalkorDB Lite connection successful` immediately
after `pip install`.

If `cgc doctor` instead reports a FalkorDB Lite connection failure (missing native lib,
unsupported platform, etc.), try:

```bash
cgc config db falkordb   # make sure it's not pointed at neo4j/falkordb-remote/kuzudb
cgc doctor                # re-check
```

If it's still failing after that (e.g. a genuinely unsupported platform/sandbox with no
way to run the embedded native binary), that's a real blocker — don't force it; fall back
to another backend (`cgc config db kuzudb` is also fully local/embedded and needs no
server) rather than fighting FalkorDB Lite further.

## 4. `.cgcignore` / `.gitignore` conventions for a multi-repo workspace

Put one `.cgcignore` at the root you actually pass to `cgc index` (e.g.
`harness/.cgcignore` if you run `cgc index harness/`). Pattern syntax is gitignore-style.
The pattern that matters most for a multi-repo `workspace/` layout with sibling git
worktrees is excluding `*.worktrees/` directories (the directory git creates next to a
repo when you `git worktree add ../repo.worktrees/wt1`) — these are not independent repos
and indexing them duplicates every file already indexed from the real repo:

```gitignore
# harness/.cgcignore — index only workspace/, and never its sibling worktrees
*
!workspace/
!workspace/**

node_modules/
bin/
obj/
dist/
*.worktrees/
```

Each repo under `workspace/*` can additionally carry its own `.cgcignore` (CGC
auto-generates a reasonable default — `node_modules/`, `venv/`, `dist/`, `build/`,
binary/media extensions — on first index if none exists); those apply within that repo's
own subtree.

**Verify exclusions actually held**, don't just trust the ignore file — query the graph
after indexing:

```bash
cgc query "MATCH (f:File) WHERE f.path CONTAINS 'worktrees' RETURN f.path"
# expect: []
```

## 5. Indexing a multi-repo workspace

You do **not** need to index each repo separately. Pointing `cgc index` at the common
parent directory (e.g. `harness/`, containing `workspace/api-service`,
`workspace/shared-lib`, `workspace/web-frontend` as separate git repos) is enough — CGC
auto-detects each nested `.git` root and splits them into separate `Repository`/`Project`
entries in the graph, while still resolving cross-repo `IMPORTS`/`CALLS` edges between
them (e.g. a Python script in one repo calling a function defined in another repo it
depends on):

```bash
cgc index harness/ --summarize --no-progress
```

Confirm what landed:

```bash
cgc list                                            # one row per detected repo
cgc query "MATCH (f:File) RETURN f.path"             # sanity-check scope
```

If you ever need per-repo isolation instead (e.g. testing ignore rules for just one
repo), `cgc index workspace/<repo>` works the same way against a single root.

## 6. Re-indexing after initial setup

- **Incremental refresh** of a repo already in the graph (picks up new/changed/deleted
  files without touching unrelated repos):
  ```bash
  cgc update harness/
  ```
- **Full rebuild** of one repo (drops and re-creates its subgraph — use after changing
  `.cgcignore` rules or upgrading `codegraphcontext` across a version with graph schema
  changes):
  ```bash
  cgc index harness/ --force
  ```
- **Live watch** (auto-update on file save, useful during active development):
  ```bash
  cgc watch harness/
  ```
- Remove a repo entirely: `cgc delete <repo-name>` (see `cgc list` for exact names).

## 7. Useful checks after (re-)indexing

```bash
cgc doctor                          # overall health
cgc list                            # which repos are indexed
cgc stats                           # indexing statistics
cgc query "<cypher>"                # ad-hoc graph queries, e.g. cross-repo CALLS/IMPORTS
cgc report                          # generate CGC_REPORT.md (god nodes, complexity, cross-module connections)
```

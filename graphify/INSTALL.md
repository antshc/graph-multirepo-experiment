# Installing and running graphify (per-repo, unmerged) for this harness

Generalized install/config guide, written from the actual install used for
`corpus/harness/workspace/{api-service,shared-lib,web-frontend}`.

## Prerequisites

- Python >= 3.10 (this install used 3.12.3).
- A working `pip`/venv toolchain.
- Per-repo git checkouts already in place (graphify extracts from the filesystem, not from a git
  remote — `graphify clone <url>` is only a convenience wrapper for fetching one first).

## Install

The importable module and CLI command are named `graphify`, but the **PyPI package name is
`graphifyy`** (double y) — installing `graphify` (single y) will not get you this tool.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install graphifyy
```

Verify:

```bash
graphify --help
```

## Running `extract` — once per repo, never at the workspace root

`graphify extract <path>` writes `<path>/graphify-out/{graph.json,manifest.json,.graphify_analysis.json}`.

**Always point `extract` at an individual repo folder, never at the parent workspace folder that
holds several repos.**

### Graph storage rule: every repo's graph lives in the harness, never inside the repo itself

A workspace repo's own checkout must never end up holding a `graphify-out/` folder. Every repo's
graph is relocated into (or, if you use `--out`, would be produced by pointing `extract` at) the
**harness repo's own** `graphify-out/<repo-name>/` — e.g. `corpus/harness/graphify-out/api-service/`,
not `corpus/harness/workspace/api-service/graphify-out/`.

Two ways to get there were evaluated:

- **`--out` flag** (`graphify extract workspace/api-service --out corpus/harness/graphify-out/api-service`):
  rejected. `extract` always appends its own `graphify-out/` under whatever `DIR` you pass to `--out`,
  so this actually writes the doubly-nested
  `corpus/harness/graphify-out/api-service/graphify-out/graph.json` — not the flat layout wanted.
- **Default extract, then move (the method used here):** run `extract` with no `--out` (writes to the
  repo's own `<repo>/graphify-out/` as usual), then move that whole folder into the harness and delete
  it from the repo:

  ```bash
  source .venv/bin/activate
  graphify extract corpus/harness/workspace/api-service --code-only
  graphify extract corpus/harness/workspace/shared-lib --code-only
  graphify extract corpus/harness/workspace/web-frontend --code-only

  mv corpus/harness/workspace/api-service/graphify-out corpus/harness/graphify-out/api-service
  mv corpus/harness/workspace/shared-lib/graphify-out  corpus/harness/graphify-out/shared-lib
  mv corpus/harness/workspace/web-frontend/graphify-out corpus/harness/graphify-out/web-frontend
  ```

  This gives the flat, wanted layout — `corpus/harness/graphify-out/<repo-name>/graph.json` — and after
  the `mv`, `git -C corpus/harness/workspace/<repo> status --porcelain` confirms the repo checkout no
  longer contains a `graphify-out/` folder at all. Re-running the two commands above from a clean state
  reproduces byte-identical `graph.json` output (only cache/manifest timestamps differ), confirming the
  workflow is reproducible.

This produces three independent `graph.json` files, one per repo, with no shared node
IDs between them — see [tests/cross-repo-navigation.md](tests/cross-repo-navigation.md) for what that
does and doesn't let you answer.

### The sibling-`.worktrees/` danger: real risk, not observed here

The ticket calls out a risk: pointing `extract` at a workspace folder as a whole could wander into a
sibling git worktree directory (here, `workspace/web-frontend.worktrees/wt1`, a separate worktree of
`web-frontend`) and mix its files into the graph.

- **Confirmed not an issue in this corpus**, because `extract` was only ever run against each repo
  folder directly, never against `workspace/` itself — see
  [report/worktrees-and-mcp-findings.md](report/worktrees-and-mcp-findings.md) for the exact checks
  (no `.worktrees` substring in any `manifest.json`/`graph.json`; the one attempt at a harness-root
  extract only ever indexed the root `README.md`).
- **Still a real risk in principle** if someone runs `graphify extract workspace/` (the parent) instead
  of `graphify extract workspace/api-service` (one repo): nothing in the CLI refuses to walk into a
  `*.worktrees/` sibling directory that happens to live under the same parent. Treat "one `extract`
  call per repo folder" as a hard rule, not a suggestion, and use `.graphifyignore` in the parent
  folder (if you ever do extract there) to exclude `*.worktrees/`.

## Re-running after initial setup

- **Incremental re-extract** (no LLM calls, fast): `graphify update <path>` re-extracts changed code
  files and updates that repo's `graph.json` in place. Use `--force` (or `GRAPHIFY_FORCE=1`) if a
  refactor deleted enough code that the rebuild would otherwise look smaller than the existing graph
  and get skipped.
- **Full re-extract**: re-run `graphify extract <path>` (same command as initial setup); it is
  idempotent per repo.
- **Continuous**: `graphify watch <path>` rebuilds the graph on code changes for one repo while you
  work.
- Do this **separately for each repo** every time — there is no "refresh all repos" command; loop over
  the repo paths yourself.
- `update`/`watch`, like `extract`, write into `<path>/graphify-out/` under the repo path you give them
  — since that folder was relocated out of the repo, re-run the `mv` step above after each of these to
  move the refreshed output back into `corpus/harness/graphify-out/<repo-name>/` (overwriting the old
  copy). There is no flag that makes `update`/`watch` target the relocated folder directly.

## Querying

Since the graph no longer lives inside the repo, use `--graph` pointed at its relocated location in
the harness (there is nothing to `cd` into inside the repo itself anymore):

```bash
source .venv/bin/activate
graphify query "does api-service use shared-lib?" --graph corpus/harness/graphify-out/api-service/graph.json
```

It is still exactly one graph per invocation — `graphify` has no CLI flag to query two repos' graphs
at once. To ask a question that spans two repos, query each repo's graph separately and reconcile the
answers yourself, or build a merged graph up front with `graphify merge-graphs g1/graph.json
g2/graph.json` / `graphify global add <graph.json> --as <tag>` (a different, explicitly-merged
strategy, not the per-repo default this guide describes).

## MCP server (`graphify-mcp`)

A second console script, `graphify-mcp`, serves graph-query tools over MCP (stdio or
`--transport http`) for AI assistants. Every tool takes an optional `project_path` argument that
resolves to `<project_path>/graphify-out/graph.json` and loads it as the active graph for that call.
With graphs relocated into the harness, `project_path` for a given repo must be set to its relocated
folder in the harness (e.g. `corpus/harness/graphify-out/api-service`), not the repo's own checkout
path — the repo checkout no longer has a `graphify-out/` folder to resolve. This still does not merge
repos: switching `project_path` between calls switches which single graph is active, it does not let
one call see two repos' nodes at once (see
[report/worktrees-and-mcp-findings.md](report/worktrees-and-mcp-findings.md)). Running `graphify-mcp`
requires the `mcp` package (`pip install mcp`) in addition to `graphifyy` — it is not a dependency of
the base install.

# Findings: `.worktrees` leakage and MCP `project_path` cross-repo isolation

Short notes backing [tests/cross-repo-navigation.md](../tests/cross-repo-navigation.md). The main
narrative and conclusions belong in the final report message, not here.

## `.worktrees` leakage check

- `corpus/harness/graphify-out/` has no `manifest.json` and no `graph.json` at its own top level —
  only `cache/stat-index.json` (a single entry for the harness root `README.md`) and the three
  relocated per-repo subfolders `api-service/`, `shared-lib/`, `web-frontend/`, each holding its own
  `manifest.json`/`graph.json` (moved here from each repo's own `graphify-out/`; see `INSTALL.md`). A
  full `graphify extract` was never run against the harness root itself; only against each sub-repo
  directly.
- `grep -n "worktrees" corpus/harness/graphify-out/*/manifest.json corpus/harness/graphify-out/*/graph.json`
  across all three relocated per-repo folders returns no matches.
- `corpus/harness/graphify-out/web-frontend/manifest.json` keys are all relative paths rooted at
  `web-frontend/` itself (e.g. `src/components/GreetingCard.tsx`); none reference
  `web-frontend.worktrees/wt1`.
- Conclusion: the danger described in the ticket (an `extract` pointed at a workspace folder
  wandering into a sibling `.worktrees/` directory) did not occur here, because no `extract` was ever
  run above the individual repo directories, and each individual repo's `graphify-out` only sees
  files inside that one directory. The risk is real *in principle* (nothing stops someone from running
  `graphify extract corpus/harness/workspace` instead of per-repo) — see `INSTALL.md`'s callout — but
  not observed in this corpus.

## MCP `project_path` cross-repo isolation

- `graphify-mcp` (`graphify.serve:_main`, a separate console-script entry point from `graphify`
  itself) exposes graph-query tools over MCP (stdio or streamable-http). Every tool accepts an
  optional `project_path` argument (`serve.py`, search `Multi-project support`).
- `_resolve_graph_path(project_path)` maps it to `<project_path>/graphify-out/graph.json`; `_select_graph`
  loads/binds exactly that one file as the active graph for the call via a `_GraphContextCache` (an
  LRU of loaded graphs, default capacity 8, `GRAPHIFY_MAX_CONTEXTS`). Nothing in `_select_graph` or the
  tool handlers merges two contexts together — each call operates against one graph.
- Live protocol test was not run: the `mcp` PyPI package isn't installed in `graphify/.venv`
  (`import mcp` fails), so `graphify-mcp` cannot start as-is. Confirmed the isolation behavior instead
  via source read of `serve.py` plus the equivalent CLI test: `graphify query` run with
  `--graph corpus/harness/graphify-out/web-frontend/graph.json` only ever sees that file's 90 nodes
  (confirmed: a query for `GreetingCard`'s call target returns just the one isolated node, no path to
  anything in `api-service`'s graph). Since `project_path` resolves to the same per-repo `graph.json`
  files (now at their relocated harness path), the same isolation applies to the MCP server.

## Re-verified after relocating graphs into the harness

Each repo's `graphify-out/` was subsequently moved out of its own checkout and into the harness repo,
under `corpus/harness/graphify-out/<repo-name>/` (see `INSTALL.md`). This is a storage-location change
only — re-ran the isolation check against the new path to confirm the semantics above still hold:

```
graphify query "api-service" --graph corpus/harness/graphify-out/web-frontend/graph.json
# -> No matching nodes found.
graphify query "App" --graph corpus/harness/graphify-out/web-frontend/graph.json
# -> 10 nodes found, all web-frontend's own (App(), GreetingCard(), HealthStatus(), ...)
```

Confirms cross-graph isolation is unchanged: querying the relocated `web-frontend` graph still finds
only its own nodes and nothing from `api-service`, and the query mechanism itself still works
normally against the new path (positive control above). The `.worktrees` exclusion finding above was
also re-checked against the relocated paths and still holds.

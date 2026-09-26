# graph-multirepo-experiment

Throwaway experiment repo for antshc/brain issue #128 (wf/harness workspace redesign).
Compares graphify, CodeGraphContext, and Sourcegraph + scip-dotnet for multi-repo code-graph navigation.

Each tool folder is fully self-contained: its own synthetic multi-repo corpus, its own `tests/`
navigation cases, its own `report/` results, and its own `INSTALL.md` — so agents can work on
each folder in parallel without conflicts. No existing repositories are reused.

- `cgc/` — CodeGraphContext (issue #141, experiment #135)
- `graphify/` — graphify (issue #142, experiment #134)
- `sourcegraph/` — Sourcegraph + scip-dotnet (issue #143, experiment #140)

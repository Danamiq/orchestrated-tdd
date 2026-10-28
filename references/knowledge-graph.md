# Knowledge-graph traversal

The goal is **fewer tokens per sub-session**. Every sub-session starts cold, and exploring a codebase file by file is its biggest token cost. When the repo has a code knowledge graph, the orchestrator queries it once and hands each sub-session a precomputed **context slice**. The sub-session then queries the graph before it opens any file, and reads symbol snippets instead of whole files.

If no graph exists, skip this file. Use normal search, and don't build a graph unless the user asks.

## Detect (orchestrator, during Ground)

| Signal | Tool | Main commands |
|---|---|---|
| `graphify-out/graph.json` in the repo (often with `graphify-out/wiki/index.md`, `GRAPH_REPORT.md`) | graphify CLI | `graphify query "<question>"`, `graphify explain "<symbol or concept>"`, `graphify path "<A>" "<B>"`; each takes `--graph <path>` |
| `mcp__codebase-memory-mcp__*` tools, or the `codebase-memory-mcp` binary | codebase-memory-mcp (CBM) | `index_status`, `detect_changes`, `get_architecture`, `search_graph`, `trace_path`, `get_code_snippet`, `query_graph` |
| Project instructions (`CLAUDE.md`, `AGENTS.md`) name another graph or index tool | that tool | follow the project's instructions |

**Freshness**
- **CBM:** run `index_status` for the repo's project. If it is indexed, run `detect_changes`; if it is not, run `index_repository`, which is local and costs no API calls.
- **graphify:** if the graph is older than HEAD, run `graphify update .`. It is AST-only and needs no LLM. Building a new graphify graph from scratch costs API calls, so do that only if the user asks.

**CLI forms.** These matter because sub-sessions may not have the MCP server loaded:
- CBM: `codebase-memory-mcp cli <tool> --flag value`, e.g. `codebase-memory-mcp cli trace_path --project P --function-name F --direction inbound`. Run `codebase-memory-mcp cli <tool> --help` for the flags. If the MCP transport fails, retry the same tool through the CLI.
- graphify: the commands above, from the repo root or with `--graph`.

If a query errors or the results look stale, fall back to grep and file reads. Never block a task on the graph.

## Orchestrator: build each context slice

Use the graph during Ground and Plan:
- **Structure:** `get_architecture` or `graphify explain` on the feature's area.
- **Locate:** `search_graph` (`--query` for keywords; `--detail ids` for cheap wide sweeps) or `graphify query`.
- **Impact:** `trace_path` with `--direction inbound` on every symbol a task will change, to see its callers. Callers in another task's area mean shared ownership or a hot spot: sequence those tasks, or consolidate them into one sub-session.

Put the result in each contract's "Knowledge graph" section, kept small:
- the graph tool and the exact commands with project name or graph path
- the symbols the task changes, as `qualified name — file:line`
- their inbound callers that must keep working
- the existing tests that cover them

A slice is a handful of lines that replaces dozens of file reads.

## Sub-session rules (copied into the contract)

1. Start Phase A with the slice, then query the graph. Open a file only when a snippet isn't enough.
2. Before changing a symbol's signature or behavior, trace its inbound callers and update every call site in your ownership. Report call sites outside your ownership as BLOCKED.
3. The graph reflects the **base** commit, not your edits. Use `git diff` for your own changes.
4. **Don't** reindex your worktree, and **don't** run `graphify update` or commit graph artifacts (`graphify-out/`, index files). Parallel branches that each change the graph produce merge conflicts.

## After integration

- Run a ripple check on the combined diff. For each changed public symbol, `trace_path` inbound against the base index lists the callers that existed before. Confirm the integration tests exercise them, alongside the full suite.
- If the project tracks graph files, refresh them once, in the integration worktree: `graphify update .`, and commit that update with the integration. For CBM, run `detect_changes` or reindex after the merge lands. Mention in the final report that the graph was refreshed.
- If the repo has graphify's git merge driver set up (`graphify hook install`), conflicts on `graph.json` merge automatically. Otherwise, regenerate the graph instead of hand-merging it.

# Knowledge-graph traversal

The goal is **fewer tokens per sub-session**. Every sub-session starts cold, and exploring a codebase file by file is its biggest token cost. When the repo has a code knowledge graph, the orchestrator queries it once and hands each sub-session a precomputed **context slice**. The sub-session then queries the graph before it opens any file, and reads symbol snippets instead of whole files.

If no usable graph exists, ask the user whether to build one (see "No graph yet: ask the user"). If they decline, use normal search.

## Detect (orchestrator, during Ground)

| Signal | Tool | Main commands |
|---|---|---|
| `graphify-out/graph.json` in the repo (often with `graphify-out/wiki/index.md`, `GRAPH_REPORT.md`) | graphify CLI | `graphify query "<question>"`, `graphify explain "<symbol or concept>"`, `graphify path "<A>" "<B>"`; each takes `--graph <path>` |
| `mcp__codebase-memory-mcp__*` tools, or the `codebase-memory-mcp` binary | codebase-memory-mcp (CBM) | `index_status`, `detect_changes`, `get_architecture`, `search_graph`, `trace_path`, `get_code_snippet`, `query_graph` |
| Project instructions (`CLAUDE.md`, `AGENTS.md`) name another graph or index tool | that tool | follow the project's instructions |

## Graph source: latest main first

Before any graph query, **update the graph to the latest commit of the repo's main (default) branch**. The only exception is when the user explicitly names another branch for the graph: then use that branch's latest commit instead. A graph left at whatever the checkout happened to hold sends every sub-session to stale symbols and callers.

1. **Find the branch.** Use the branch the user named. Otherwise use the default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (e.g. `origin/main`). If that ref isn't set, run `git remote show origin` and read "HEAD branch". With no remote, use the local `main` or `master`.
2. **Get its latest commit.** Run `git fetch origin <branch>`; this only reads from the remote and changes no local branch. Record `GRAPH_SHA` = `git rev-parse origin/<branch>`. If the fetch fails (offline, no remote), use the local branch tip and say so in the plan.
3. **Pick where to build.** Never check out, pull, reset, or stash in the user's working tree for this.
   - **In place:** if the main checkout is already on that branch, at `GRAPH_SHA`, with a clean `git status`, refresh it there.
   - **Graph worktree:** otherwise use `git worktree add --detach .claude/worktrees/ocd-graph GRAPH_SHA`, or `git -C .claude/worktrees/ocd-graph checkout --detach GRAPH_SHA` if it already exists. Build there and reuse it on later runs.
4. **Refresh.** These need no question, because they are local and free:
   - **CBM:** `index_repository` on the chosen path, which updates incrementally if it is already indexed. Then `index_status` to confirm. Record the CBM project name for that path; sub-sessions query that project.
   - **graphify:** `graphify update <path>`, which is AST-only with no LLM. Sub-sessions then pass `--graph <path>/graphify-out/graph.json`.
   - If the graph doesn't exist yet at all, first ask the user (next section). If they agree, build it at `GRAPH_SHA` the same way.
5. **Report it** in the plan message, in one line: "Graph: <tool> at <branch>@<short GRAPH_SHA>". The user can then say "build it from branch X" and you redo steps 1–4 for X.

**When the task base differs from the graph.** Tasks branch from the user's current branch (`BASE_SHA`). If that isn't `GRAPH_SHA`, the graph shows main, not the base:
- Compare the two with `git diff --stat GRAPH_SHA BASE_SHA`.
- Put that line in each contract, so sub-sessions confirm any listed file with a quick read or diff before trusting the graph for it.
- If the difference is large (many changed files in the task's area), say so in the plan and suggest building the graph from the user's branch instead.

## No graph yet: ask the user

When no graph or index exists for this repo, ask once during Ground, before you explore. Exploration is where a graph pays off, and every later sub-session and `/ocd-fix` run reuses it. Use AskUserQuestion, and batch it with any other Ground questions. Offer only what is actually installed (`command -v`, or the MCP tools are present):

| Option | When to offer | Cost |
|---|---|---|
| **Index with codebase-memory-mcp** (Recommended when installed) | CBM is available but this repo isn't indexed | Local and needs no API calls. Usually seconds to a few minutes. Stored outside the repo. |
| **Build a graphify graph** | graphify is installed and `graphify-out/` doesn't exist | The first build runs graphify's own extraction, which may call an LLM API: name that cost. It writes `graphify-out/` into the repo; ask whether to commit or ignore it. Later refreshes (`graphify update .`) are AST-only and free. |
| **Skip** | always | Sub-sessions explore with normal search, which costs more tokens on large repos or long runs. |

How to word the question:
- Say why you're asking: a graph cuts each sub-session's exploration tokens, and the savings grow with the number of sub-sessions, the repo size, and later `/ocd-fix` iterations.
- For a small repo, or a one- or two-task plan, say that skipping is reasonable.
- If neither tool is installed, don't install anything. In one line, say that a code graph tool such as codebase-memory-mcp or graphify would cut tokens on later runs, and continue without one.

After the answer:
- **Yes:** build it, confirm it works with one cheap query (`index_status`, or `graphify query` on a known symbol), then continue Ground with it.
- **Build fails:** report the error in one line and continue without a graph. Don't retry in a loop.
- **No:** record the decision in the ledger, and don't ask again in this chat (including `/ocd-fix`). Ask again only if the user brings it up.

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
3. The graph reflects `GRAPH_SHA` (latest main, or the branch the user chose), not your edits and not necessarily your base. Use `git diff` for your own changes. For files the contract lists as differing between graph and base, read or diff before trusting the graph.
4. **Don't** reindex your worktree, and **don't** run `graphify update` or commit graph artifacts (`graphify-out/`, index files). Parallel branches that each change the graph produce merge conflicts.

## After integration

- Run a ripple check on the combined diff. For each changed public symbol, `trace_path` inbound against the base index lists the callers that existed before. Confirm the integration tests exercise them, alongside the full suite.
- If the project tracks graph files, refresh them once, in the integration worktree: `graphify update .`, and commit that update with the integration. The next `/ocd` run still starts by refreshing from latest main, as described in "Graph source". Mention in the final report that the graph was refreshed.
- If the repo has graphify's git merge driver set up (`graphify hook install`), conflicts on `graph.json` merge automatically. Otherwise, regenerate the graph instead of hand-merging it.

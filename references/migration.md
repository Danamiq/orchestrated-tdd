# Migration mode (`/ocd-migrate`)

Port a codebase, or part of one, from one language or runtime to another (Python → Rust, Ruby → Go, JavaScript → TypeScript, Java → Kotlin, and so on) **without changing what it does**. Everything in SKILL.md still applies: single gateway, worktrees, observed TDD evidence, model routing, knowledge graph, integration verification. This file adds the migration-specific plan, ordering, and safety rules.

The core idea: **the existing implementation is the specification.** Before porting anything, capture its behavior in language-neutral parity tests. Every ported slice must pass the same tests the source passes.

## Contents
- Phase 0 — Decisions (ask the user)
- Phase 1 — Inventory
- Phase 2 — Parity harness first
- Phase 3 — Target foundation
- Phase 4 — Port slice by slice
- Phase 5 — Switch over
- Phase 6 — Integrate and verify
- Migration plan file and resuming
- Translation hazards checklist
- Safety rules

## Phase 0 — Decisions (ask the user)

Ask these in rounds (SKILL.md → "Asking the user"), with your recommendation for each. Don't start porting until the answers that affect correctness are in.

1. **Scope:** which part moves (e.g. "backend only"), and what stays.
2. **Source and target:** language, framework, and runtime versions on both sides; target conventions (package manager, test runner, linter, formatter, build).
3. **Strategy:**
   - **Incremental, strangler pattern.** Recommended. Source and target run side by side; slices move over one at a time behind a router, proxy, flag, or FFI boundary; each step is reversible.
   - **Parallel rewrite, then one switch-over.** Target built alongside; one cutover at the end.
   - **Big-bang.** Only for small codebases.
4. **Boundary:** where the two sides meet and get compared: HTTP API, CLI, message queue, FFI, library interface, or database.
5. **Parity standard:**
   - **Bug-for-bug** (default): identical observable behavior.
   - **Behavior with listed exceptions:** every intended divergence (a fixed bug, a changed error message) is written down and approved.
6. **Data:** shared database and schema unchanged (default), or a schema or format migration too. A schema migration is a separate workstream that needs explicit approval (SKILL.md → "Approval boundaries").
7. **Dependency map:** source library → target library for each dependency. New dependencies need approval, so present the whole map once for a single batched approval.
8. **Non-functional targets:** performance, memory, binary size, startup time. Only if the user cares; otherwise, "no regression beyond X%".
9. **Retiring the source:** deleting source code is destructive. Default: keep it until the user explicitly approves removal after switch-over.

## Phase 1 — Inventory

Use the knowledge graph if one exists (`knowledge-graph.md`, refreshed from latest main). Otherwise use normal search.

- **Modules and dependencies:** the module graph (who imports or calls whom). Detect cycles.
- **Public surface:** routes and endpoints, CLI commands, background jobs, scheduled tasks, events published and consumed, library exports.
- **Side effects:** database reads and writes, files, network calls, caches, environment variables and config, time, randomness.
- **Existing tests:** what is covered, and what isn't.
- **Hazards:** run the translation hazards checklist below over each module.

Then:
- **Order slices leaf-first.** Topologically sort the module graph: modules with no internal dependencies go first, and the most connected core goes last. Break each cycle by moving the cycle together in one slice, or by extracting an interface first.
- **Group slices into waves.** A wave is the slices whose dependencies are already ported. Respect the parallel cap.

## Phase 2 — Parity harness first

This is the migration's RED/GREEN backbone and its permanent regression guard. Build it **before** porting.

1. **Black-box tests at the boundary.** Write them in whichever language suits the boundary; often a neutral runner such as an HTTP test client, a shell or CLI harness, or the target's test framework calling over the boundary. Each test sends inputs and asserts outputs and side effects: response bodies and status codes, CLI stdout, stderr and exit codes, database state, emitted events.
2. **Golden fixtures from the source.** Record the source's actual outputs for representative and edge-case inputs. Normalize nondeterminism explicitly (timestamps, generated IDs, ordering, floating-point formatting), and document each normalization.
3. **Characterization run.** Run the whole parity suite against the **source** and see it pass. A parity test that fails against the source is a wrong test: fix the test, never the source.
4. **Point it at either side.** One switch (an environment variable, a base URL, a binary path) selects source or target. Every slice is then verified against the same suite.

Model tier: Opus. Use Fable if the boundary carries authentication or other security behavior.

## Phase 3 — Target foundation

Run this sequentially, before any slices: one sub-session on the Opus tier.
- Scaffold the target project with its toolchain, test runner, lint, format, and build commands, plus CI hooks if the project has CI.
- Build the switch-over mechanism for the chosen strategy: a router or proxy rule, a feature flag, or FFI bindings. Every slice defaults to the **source**.
- Port shared types and contracts (DTOs, schemas, error types). Generate them from a schema if one exists (OpenAPI, protobuf, JSON Schema) rather than translating by hand.
- Verify: the empty target builds, the parity suite runs against it (all slices are still served by the source), and nothing changed for users.

## Phase 4 — Port slice by slice

Each slice is a task with its own sub-session (or a consolidated group), following the normal contract plus the addendum below:
- **RED:** point that slice's parity tests at the target and watch them fail because the slice isn't implemented there yet.
- **GREEN:** port the slice until those parity tests pass against the target. Also port or rewrite its unit tests in the target's framework.
- **REFACTOR:** make the code idiomatic for the target (ownership, error types, async model) while the parity tests stay green. A literal transliteration is a starting point, not the deliverable.
- **Guards:** the slice's parity tests and its ported unit tests become its regression guards.

Model routing (SKILL.md → "Model routing"), applied to migration:
- **Fable:** authentication, authorization, sessions and tokens, crypto, input validation and parsing at trust boundaries, file and path handling, anything that shells out.
- **Opus:** core domain logic, concurrency and async, FFI or unsafe code, memory and ownership-heavy rewrites, performance-critical paths, modules with many hazards.
- **Sonnet:** straightforward modules (CRUD handlers, DTOs, mappers, formatting helpers), porting unit tests, fixtures.
- **Haiku:** config files, docs, renames, build-script boilerplate.

### Slice contract addendum (append to the task contract)

```
## Migration slice
Source: <files/modules in source language>   Target location: <paths in target project>
Strategy: <strangler | parallel | big-bang>   Parity standard: <bug-for-bug | with exceptions: ...>
Approved dependency map for this slice: <source lib -> target lib>
Hazards found in this slice: <from the checklist, e.g. integer overflow, naive datetimes>
Parity tests for this slice: <names / paths>; run against target with: <cmd / env switch>
Do NOT modify or delete source code. Do NOT change the switch-over default. Do NOT add
dependencies outside the approved map (report BLOCKED instead).
Any behavior you cannot match exactly: stop and report BLOCKED with the exact difference.
Never adjust a parity test or golden file to make the target pass.
```

## Phase 5 — Switch over

- Within the repo: flipping a slice's switch from source to target is a code or config change. Do it per slice only after that slice is VERIFIED and integrated, on the integration branch, with the user's approval of the batch of slices.
- Live systems: deploying, or flipping flags in production, is the user's call. The orchestrator never does it. It can prepare a switch-over checklist: order, rollback step for each slice, what to monitor.
- Optional dual-run (shadow traffic, comparing source and target outputs): propose it when the user's infrastructure supports it. Running it against live systems needs approval.

## Phase 6 — Integrate and verify

On top of SKILL.md step 6:
- Run the **full parity suite against the target**, with every migrated slice switched to the target in the integration worktree. Then run it against the source as well, to prove the source is still intact.
- Run all target unit tests, lint, typecheck, and build, plus the source's own suite (nothing broken during coexistence).
- Check non-functional targets if they were set (a simple benchmark on both sides, same inputs).
- Review against the dependency map and the list of approved divergences. Any unapproved divergence is a failure.

The final report adds a **migration matrix**: one row per slice, with columns for status (`planned`, `ported`, `parity-green`, `switched (repo)`, `source retired`), parity tests and pass counts, approved divergences, tier and model, and follow-ups.

## Migration plan file and resuming

Migrations outlive one chat. Keep a plan file on the migration's integration branch; it is the single source of truth for decisions and progress:
- Location: `docs/migration/<name>.md` by default; ask once if the project has a docs convention.
- Contents: the Phase 0 decisions and approvals, dependency map, slice order and waves, hazards, the parity-suite command, the migration matrix, and open questions.
- Update it at each milestone (wave verified, switch-over, decision made), and commit it with the integration.

Resuming:
- **Same chat:** `/ocd-migrate` (or `/ocd-fix` for a ported slice) reuses the existing slice sub-sessions, as in fix mode.
- **New chat:** `/ocd-migrate continue <name>` reads the plan file, re-verifies the matrix against git and the parity suite, and continues with the next wave in new sub-sessions. Sub-sessions from older chats aren't messaged from here.

## Translation hazards checklist

Check each slice for these. They are where "same code, different language" silently changes behavior.

- **Numbers:** integer width and overflow (wrapping, panicking, or arbitrary precision), integer division and modulo of negative numbers, float formatting and rounding, decimal vs binary float, NaN and infinity handling.
- **Strings:** bytes vs code points vs grapheme clusters, encoding (UTF-8/16), normalization, case folding and locale, slicing and indexing, regex dialect differences.
- **Time:** naive vs zone-aware datetimes, default time zone, DST, clock source, timestamp precision, date formatting and parsing.
- **Errors:** exceptions vs result types, errors that were silently swallowed, error messages and codes that clients depend on, panics and abort semantics.
- **Nulls and absence:** None, null, undefined, and missing keys; empty vs absent; default values.
- **Collections:** iteration order (dicts, maps, sets), sort stability, mutability and aliasing, copy vs reference semantics.
- **Concurrency:** GIL or single-thread assumptions, async runtime differences, thread safety, cancellation, timeouts, connection pooling.
- **Dynamic features:** reflection, metaprogramming, monkey-patching, duck typing, eval, decorators, plugins. These usually need a redesign, so route them to Opus.
- **Serialization:** JSON field order, number precision, null vs omitted fields, date formats, enum representation, casing conventions.
- **I/O and environment:** file paths and separators, line endings, environment-variable parsing, config file formats, working directory, signals and exit codes.
- **Memory and resources:** garbage collection vs ownership, finalizers vs RAII, resource lifetimes, large-data streaming.
- **Third-party behavior:** a replacement library may differ in defaults, validation, or edge cases. Cover each mapped library with parity tests.

## Safety rules

- Never delete, rewrite, or "fix" source code during migration. Changing the source is out of scope unless the user approves it.
- Never weaken a parity test or regenerate a golden file to make the target pass. A wrong golden file is fixed only by re-recording it from the source, and only after the user agrees the old recording was wrong.
- Every intended behavior change is an approved divergence, listed in the plan file.
- New dependencies, schema or data migrations, switching over in live systems, deployment, and retiring source code all need explicit approval.
- Keep both suites green at every integration. A migration that breaks the running system halfway is the failure this mode exists to prevent.

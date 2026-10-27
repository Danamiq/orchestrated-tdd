# TDD cycle for a sub-session

Follow these phases in order. Each phase produces evidence that goes in the final report. Evidence means a command you ran and the output you saw. Writing "tests fail" without running them is a false report, even when the claim is probably right.

## A. Understand

- Read the owned code, its callers, its existing tests, and the project rules named in the contract.
- Pin down the intended behavior and its edge cases: empty/null, boundaries, invalid input, error paths, concurrency, time zones, unicode, permissions, partial failure, as relevant.
- Choose the smallest testable slice. Big tasks become several RED→GREEN loops, not one giant one.
- List the existing behavior that must stay unchanged. Make sure some test covers it, or add a characterization test first and see it pass.

## B. RED — failing tests first

- Write tests for the new behavior **before** touching production code. Use the project's existing framework, file layout, naming, and fixtures. Don't introduce a new test framework.
- Cover the success path plus the relevant negative, boundary, and failure cases.
- Run them. Confirm each new test fails **for the expected reason**: an assertion about the missing behavior, or a missing symbol that the task is supposed to add.
- These are not RED: import errors in unrelated modules, a missing dependency, a misconfigured runner, a typo in the test, or a network or database that is unavailable. Fix the test setup if it is in scope. Otherwise report BLOCKED.
- If a new test passes immediately, the behavior already exists or the test checks nothing. Find out which before continuing.

## C. GREEN — minimum production change

- Write the smallest change that makes the failing tests pass. No speculative abstractions, no unrelated cleanup, no new dependencies without approval.
- Run the targeted tests and confirm they all pass. Record the command and the pass/fail counts.

## D. REFACTOR

- Improve names, structure, and duplication while keeping behavior identical, and match the surrounding conventions.
- Re-run the targeted tests after refactoring. If you changed nothing, say "none", which is fine.

## E. Regression guards (mandatory)

Every new or changed behavior leaves automated tests that fail if that behavior later breaks. Check each item that applies:

- [ ] Primary success path
- [ ] Invalid input and failure handling (errors raised/returned, messages, status codes)
- [ ] Boundaries and important edge cases
- [ ] Neighboring existing behavior the change could disturb
- [ ] The originally failing scenario, for bug fixes. This test must fail on the old code.
- [ ] Contracts that matter here: API shape, persistence and data integrity, permissions and security boundaries, integration with other modules

Rules for guards:
- Assert observable behavior (outputs, state, side effects), not implementation details or mock call counts unless the call *is* the contract.
- Make them deterministic: no real clock, randomness, network, or ordering assumptions without control.
- Coverage padding is not a guard. A test that can't fail when the behavior breaks has no value.
- Prove at least one guard by temporarily breaking the protected code, watching the guard fail, then reverting. Note it in the report.
- Add integration, contract, or end-to-end tests when the risk lives at a boundary unit tests can't see, and the project already has (or the contract allows) that layer.

## F. Verify and hand off

1. Run the targeted tests, then the broader regression suite when feasible.
2. Run lint, typecheck, format check, and build if the project has them.
3. If anything fails, debug and fix it within scope. Writing the code does not make the task COMPLETED.
4. Inspect `git diff <base>...HEAD` and `git status`: no unrelated changes, no accidental deletions, no secrets or credentials, no debug output, no skipped or disabled tests, no unfinished TODOs.
5. Commit on your worktree branch.
6. Write the final report in the contract's format. List every check you did **not** run and why (missing infrastructure, needs credentials, too slow, and so on). If one of those checks matters, report BLOCKED instead of COMPLETED.

## Stack detection hints

Find the commands in the project; don't assume them. Look in this order: the contract, then the project docs, then CI config (`.github/workflows`, `.gitlab-ci.yml`), then manifest scripts. Common signals:

| Signal | Typical runner (verify in scripts first) |
|---|---|
| `package.json` scripts | `npm test` / `pnpm test` / `yarn test`; vitest, jest; `tsc --noEmit`; eslint |
| `pyproject.toml` / `pytest.ini` / `tox.ini` | `pytest`, `uv run pytest`; ruff, mypy, pyright |
| `go.mod` | `go test ./...`, `go vet ./...` |
| `Cargo.toml` | `cargo test`, `cargo clippy`, `cargo fmt --check` |
| `pom.xml` / `build.gradle*` | `mvn test` / `./gradlew test` |
| `*.csproj` / `*.sln` | `dotnet test`, `dotnet build` |
| `pubspec.yaml` | `flutter test` / `dart test`, `flutter analyze` |
| `Gemfile` | `bundle exec rspec` / `rake test` |
| `Makefile` / `justfile` | project targets often wrap all of the above |

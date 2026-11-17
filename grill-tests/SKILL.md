---
name: grill-tests
description: Audits and hardens a test suite by finding redundant tests to delete, consolidating heavy integration tests that boot containers or app hosts, and reporting duplicated test helpers with concrete extraction options. Use when asked to "grill tests", "audit tests", "clean up tests", "find redundant tests", "reduce test time", "speed up slow tests", "consolidate integration tests", "Testcontainers fixture is slow", "duplicated test code", "extract test helpers", or "is this test suite healthy". Produces an analysis report and an approval-gated change plan before editing; never deletes a test without naming the test that still covers the behavior.
---

# Grilling the test suite

Analysis-and-cleanup workflow for an existing test suite. The output is an evidence-backed report
plus a reviewed change plan, then edits.

**Bundled reference:** [`references/SHARED-CONTAINER-FIXTURE.md`](references/SHARED-CONTAINER-FIXTURE.md)
— a concrete **.NET / C# / xUnit v3 / Testcontainers** recipe for rung 2. Read it when the suite boots
containerised databases per test unit: it holds the full fixture implementation, the granularity trap,
and the failure modes. Adapt the naming and the migrator entry point; the structure transfers as-is.

## When to use

- A test suite is slow, flaky, or has grown to the point nobody trusts it.
- Heavy integration tests boot containers, app hosts, or databases and the suite feels too expensive.
- Suspected duplicated tests or duplicated test helpers across multiple test projects.
- Preparing a test suite for review or for CI cost reduction.

## When NOT to use

- **Running tests, filtering, or choosing a command** → use the repository's own test documentation or
  its dedicated test-running skill. Never invent a command; discover it.
- Writing tests for a brand-new feature, or making a failing test pass → edit directly.
- CI/build pipeline problems → the pipeline config files, not the tests.
- Choosing a test framework or migrating frameworks → a separate, much larger decision.

## Inputs required

- **Scope**: whole suite, or specific test projects/paths. Default: whole suite.
- **Intent**: `audit` (report only) or `cleanup` (report + apply approved changes). Default: `audit`.
- If the intent is `cleanup`, confirm before writing to any file outside test code.

## Ground rules

1. **Derive every fact from the repository in front of you.** Do not assume project names, paths,
   frameworks, or commands. Read manifests and test files first; hardcoded guesses are the main way this
   skill goes wrong on a new repo.
2. **Discover the test command, don't prescribe it.** Find it in the repo (README, CONTRIBUTING, CI
   config, build scripts) and reuse it verbatim.
3. **Identify the test framework before recommending idioms.** `[Fact]`/`[Theory]`, `[Test]`/`[TestCase]`,
   `it`/`test`, `describe`/markers — the patterns below translate, the syntax does not.
4. **Coverage is the constraint, not the goal.** Never delete or merge a test unless you can name the
   specific remaining test that still exercises the same behavior. Write that name down.
5. **Analysis precedes edits.** Always produce the report (step 6) and get approval before modifying files.
6. **Never combine tests that assert independent outcomes.** Combining is valid only for sequential steps
   of one workflow or for data variants of one identical assertion path.
7. **A combined test must still localize failure.** Each step asserts separately with a distinct failure
   message. No `try/catch` swallowing multiple asserts, no single aggregate "everything worked" assertion
   at the end.
8. **Do not weaken assertions to make a merge work.** If merging forces you to drop or soften an
   assertion, the merge is invalid — revert to consolidation without combining.
9. **Respect build configurations.** Tests behind conditional compilation (`#if DEBUG`, feature flags,
   build tags, env guards) can be invisible in some configurations. Verify count changes in a
   configuration that actually compiles them.
10. **Editing only test code.** Changing production code, adding a shared test project, or altering the
    build/CI configuration is a separate proposal requiring explicit approval.

## Workflow

### 1. Establish a baseline

Nothing can be claimed as an improvement without a "before". Record it first.

- Confirm required infrastructure is available (container runtime, database, network stubs) before a
  long run — a missing daemon produces failures that look like test defects.
- Run the repo's own build and test commands at the project level first; escalate to a whole-suite run
  only if the repo's documented flow supports it.
- Capture per-project: test count, pass/fail/skip counts, wall-clock duration.

### 2. Inventory the suite

For each test project, collect:

- test class count, individual test count, parameterized/data-driven rows and their counts
- **preparation cost per unit**: does the class start a container, build an app host, apply migrations,
  seed a database, or start an in-process stub server?
- **fixture scope**: is preparation per-method, per-class, per-collection, or per-assembly?
- conditional compilation or environment guards that hide tests

Produce a small table; it becomes the basis for every later recommendation.

### 3. Find redundant tests

Classify each candidate, then decide. Taxonomy:

| Pattern | Smell |
|---|---|
| **Tautology** | Asserts the fake/stub/serializer rather than the code under test (e.g. asserts a stub returns what the stub was told to return). |
| **Duplicate path** | Same production path covered by two tests with different names, usually one fast unit test and one slower integration test. |
| **Boot-only smoke** | Several tests whose only real assertion is "the host started" — one per assembly is enough, not one per controller/endpoint. |
| **Pass-through** | Only proves a property getter or an untransformed DTO mapping works. |
| **Parameterized redundancy** | A parameterized test with a single data row next to an equivalent plain test (or vice versa). |
| **Orphaned fixture** | Test data files, fakes, or helpers with no remaining consumer. |
| **Assertion-free** | Executes a path but asserts nothing meaningful — the most common accidental addition. |

Verdicts: `delete` (redundant), `keep` (distinct coverage — say why), `merge` (step 4), `extract` (step 5).

### 4. Consolidate heavy integration tests

Heavy prep = container startup, app-host/distributed-app boot, migration+seed, or a stub server.
Apply this ladder **in order** and stop at the first rung that removes the cost:

1. **Data variants → one parameterized test.** If one method repeats with different inputs and identical
   assertions, collapse to the framework's data-driven form. Lossless: same paths, less code.
2. **Per-unit prep → one shared assembly-level fixture.** If several classes boot the same
   infrastructure from identical setup, hoist the boot into a single shared fixture and give each unit
   its own **isolated database, schema, or namespace** instead of its own server.
   **This is usually the whole win — prefer it over touching test methods at all.**
   Lossless **only if the existing isolation granularity is preserved**. Measure whether prep is really
   per-class or (common, and easy to misread) per test method before choosing the isolation unit: a
   wrong choice causes apparently random failures, because frameworks give no ordering guarantee.
   For .NET/C#/xUnit v3 suites, the ready-made implementation is in
   [`references/SHARED-CONTAINER-FIXTURE.md`](references/SHARED-CONTAINER-FIXTURE.md).
3. **Prep repeated inside one class → lifecycle hook.** Move boot out of each method into the class
   setup hook, and reset state between tests instead of rebuilding the world.
4. **Sequential steps of one workflow → one method.** Only after 1–3. Combine steps that must run in
   order anyway (create → mutate → assert → teardown). Name the method after the workflow
   (`Queue_SurvivesRestart_AndPreservesOrder`), never `EverythingWorks`. Every step keeps its own
   assertion and message.

If rung 4 is reached, state in the report exactly which distinct behaviors survive and confirm the count
of covered behaviors did not drop.

### 5. Hunt duplicated test code

Compare helpers, fixtures, fakes, base classes, and test data across projects. Then **present options;
do not silently refactor across project boundaries.**

| Option | When it fits | Tradeoff |
|---|---|---|
| **A. Shared test-support project/module** | Fixtures and base classes used by 3+ projects; state must be genuinely shared | New project to add to the build, references, CI impact — needs approval |
| **B. Linked/imported source files** | Small stateless helpers (builders, constants, serializer options, assertions) | Compiles/imports a private copy per assembly; type identity differs per project |
| **C. Leave duplicated + canonical comment** | Small helpers whose duplication is cheaper than the coupling | Drift risk; no enforcement |

Rule of thumb: **stateful fixtures → A; stateless helpers → B is acceptable; trivial one-liners → C.**
Option B is actively wrong for anything holding a live connection, a container handle, or a shared
fixture lifetime.

### 6. Produce the report

One table, no prose padding. Every row must be actionable and justified.

```
| # | Location | Finding | Verdict | Coverage still provided by | Est. saving |
|---|----------|---------|---------|----------------------------|-------------|
| 1 | path/To/Test.cs:42 | duplicate of <other test> | delete | <Project>.<Class>.<Test> | ~2 s |
```

Then three short sections — `Delete`, `Consolidate`, `Extract` — each listing the concrete edits.
Get approval before applying.

### 7. Apply and verify

- Apply only the approved rows, one concern per commit-sized change.
- Re-run the **affected projects only** while iterating, then the full suite once at the end for the
  baseline comparison, using the repo's own commands.
- Confirm the total test count dropped by exactly the number of intentionally deleted tests, and that
  nothing else disappeared. An unexpected extra drop means you deleted or silently skipped coverage.
- Report before/after duration per project touched, and state plainly if a saving was not realized.

## Report template

```markdown
## Baseline
| Project | Tests | Duration | Notes |

## Findings
| # | Location | Finding | Verdict | Covered by | Saving |

## Delete
## Consolidate
## Extract

## Result (after applying)
| Project | Tests before | Tests after | Duration before | Duration after |
```

## Pitfalls

- **Merging is the last resort, not the first.** A shared fixture usually removes the cost with zero
  coverage risk. Jump to rung 2 before touching a single test method.
- **Shared fixtures leak state.** One fixture across many tests means leftovers reach the next test.
  The safe remedy is to preserve the original isolation granularity — a separate database, schema, or
  namespace per unit — not to bolt resets onto tests that never needed them, which changes behaviour
  for any test relying on in-class state. Verify with two consecutive full runs, not one. Failure modes,
  including the per-unique-database connection-pool trap (`53300: sorry, too many clients already`):
  [`references/SHARED-CONTAINER-FIXTURE.md`](references/SHARED-CONTAINER-FIXTURE.md).
- **A merge that hides which step failed is a downgrade.** Reviewers lose the ability to localize a
  regression; that cost is often higher than the seconds saved.
- **Framework filter syntax is not portable.** Filter flags, exit codes for "no tests matched", and
  parameterized-test naming all differ per framework. Read the repo's docs rather than guessing, and do
  not treat "zero tests ran" as success.
- **Environmental flakiness is not a duplicate test.** Health-check timeouts, port collisions, and cold
  container starts are infrastructure. Retry, then fix the environment — never delete the test that
  surfaced a real problem.
- **Do not run a whole-suite integration run while a local stack is up.** Host networking and port
  conflicts produce both fake failures and fake duplicates.
- **Verify count changes in a configuration that compiles the tests.** Conditional compilation can make
  a deletion invisible, or make a healthy suite look empty.
- **License/banner/log noise in test output is not a failure** — and not a reason to change tests.
- **Do not count a test as covered because a similarly-named test exists.** Open it and confirm it
  asserts the same behavior.

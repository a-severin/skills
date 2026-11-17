# Shared Testcontainers fixture: one Postgres server, one fresh database per test

Rung-2 recipe for a **.NET / C# / xUnit v3** suite where every test class starts its own
`Testcontainers.PostgreSql` container. Target: one container per assembly, isolation kept at the
**database** level.

Scope: .NET 8+, xUnit v3, `Testcontainers.PostgreSql`, `Npgsql`, and a journal-based migrator
(DbUp, EF Core `Migrate()`, FluentMigrator — anything that records applied scripts). Substitute your
own names for `Migrations.Setup` / `Migrations.ConfigureDataSource`; everything else transfers
unchanged.

## Contents

1. [When this applies](#1-when-this-applies)
2. [The granularity trap](#2-the-granularity-trap)
3. [Choose the isolation unit](#3-choose-the-isolation-unit)
4. [The fixture](#4-the-fixture)
5. [Migrate a test class](#5-migrate-a-test-class)
6. [Roll out in stages](#6-roll-out-in-stages)
7. [Verify](#7-verify)
8. [xUnit v3 API notes](#8-xunit-v3-api-notes)
9. [Pitfalls](#9-pitfalls)
10. [Proven result](#10-proven-result)

---

## 1. When this applies

You have measured that container startup, not test bodies, dominates runtime, and the start count
scales with test count rather than class count.

Symptoms:

- `private readonly PostgreSqlContainer _container = TestUtils.CreatePostgreSqlContainer();` as a
  field on many test classes, with `StartAsync()` in `InitializeAsync`.
- One class with seven assertions takes 8–12 s on a warm machine.
- Image pull and readiness waits dominate the suite duration.

**Do not apply** when:

- Tests mutate server-level state — `CREATE EXTENSION`, roles, `ALTER SYSTEM SET`, replication
  slots. A shared server makes those order-dependent.
- A test deliberately asserts "no query is issued" through an unreachable DSN such as
  `Host=localhost;Port=59999`. It must keep building its own connection string; do not route it
  through the fixture.
- You would have to change assertions, `MaxThreads`, or reset semantics to make it pass. That is
  re-specification, not consolidation.

## 2. The granularity trap

The single most important fact about xUnit: **the test class is constructed once per test method, so
`InitializeAsync` runs once per test method.** A field declared on the class looks per-class and is
not:

```csharp
// Looks per-class. Actually runs per test method, because xUnit constructs the class
// per test: one container per test, not one per class.
public sealed class OrderServiceTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = TestUtils.CreatePostgreSqlContainer();

    public async ValueTask InitializeAsync() => await _container.StartAsync();
}
```

So the real pre-refactor granularity is **one container per test**. Preserving it means **one
database per test**, not one per class.

Confirm with a count instead of trusting the reading:

```bash
docker events --filter type=container --filter event=create \
  --format '{{.Actor.Attributes.image}}' > /tmp/creates.txt &
dotnet test --project path/To/Some.Tests.csproj -- --filter "FullyQualifiedName~OrderServiceTests"
wait; wc -l /tmp/creates.txt
```

If a class with N tests produces N container creates, granularity is per test.

> **Never default to one database per class.** It is the intuitive reading of "shared fixture" and
> the most common way this refactor introduces flakes. Two tests in the same class that write the
> same row will then race, and xUnit gives no ordering guarantee between them — so it fails randomly,
> reproduces on CI but not locally, and looks like infrastructure flakiness.

## 3. Choose the isolation unit

| Unit | Correct when | Cost | Risk |
|---|---|---|---|
| **Fresh database per test** (default) | Always. Empty journal every time. | Migration run per test | Lowest — behaviour-identical to per-test containers |
| **Fresh database per test from a migrated `TEMPLATE`** | Migrations are the remaining cost, after per-test databases are proven | One migration per run | Low — must guarantee no test writes to the template |
| **Fresh schema per test** in one database | App resolves schema names at runtime, no `search_path` hard-coding | Migration run per test | Medium — cross-schema contamination |
| **Transaction rollback per test** | Exactly one connection, no DDL, no background workers | Near zero | High — silently wrong with a second connection, pooled `DbContext`, or hosted service |
| **Truncate + reseed per test** | Every mutable table enumerable and seeded deterministically | Low | High — drifts silently as the schema grows |

Prefer **fresh database per test**. It is the only option that requires no knowledge of what
individual tests mutate — which matters because the suite will be maintained by people who will not
update your truncation list.

Record the decision and its reason in the fixture's XML doc comment, or the next person will
"optimise" it to one database per class and reintroduce the flakes.

## 4. The fixture

`Tests/Infrastructure/PostgresFixture.cs`:

```csharp
using Npgsql;
using Testcontainers.PostgreSql;

[assembly: AssemblyFixture(typeof(MyApp.Tests.Infrastructure.PostgresFixture))]

namespace MyApp.Tests.Infrastructure;

/// <summary>
/// One PostgreSQL container for the whole test assembly.
/// </summary>
/// <remarks>
/// Isolation is kept at the <b>database</b> level: <see cref="CreateDatabaseAsync"/> creates a
/// uniquely named catalog and runs the full migration journal against it.
///
/// Granularity is unchanged from before the refactor. xUnit constructs a test class and calls
/// <c>InitializeAsync</c> <b>once per test method</b>, so the old code started a container per test;
/// this starts a database per test on a shared server. One database per class would change
/// behaviour and fail at random, because tests in a class are not ordered.
/// </remarks>
public sealed class PostgresFixture : IAsyncLifetime
{
    private const int PurposePrefixLength = 12;

    private readonly SemaphoreSlim _containerLock = new(1, 1);
    private readonly SemaphoreSlim _createDatabaseLock = new(1, 1);

    private PostgreSqlContainer? _container;
    private string? _adminConnectionString;
    private int _databaseCounter;

    /// <summary>Starts the shared container before the first test runs.</summary>
    public async ValueTask InitializeAsync()
    {
        await GetAdminConnectionStringAsync(TestContext.Current.CancellationToken);
    }

    /// <summary>
    /// Creates a uniquely named database, applies every migration to it, and returns a data source
    /// configured the same way as in the tests.
    /// </summary>
    /// <param name="purpose">Class or test name; lands in the database name so logs stay readable.</param>
    /// <param name="ct">Cancellation token.</param>
    public async Task<NpgsqlDataSource> CreateDatabaseAsync(
        string purpose,
        CancellationToken ct = default)
    {
        var adminConnectionString = await GetAdminConnectionStringAsync(ct);

        var index = Interlocked.Increment(ref _databaseCounter);
        var databaseName =
            $"test_{Sanitize(purpose)}_{index:D4}_{Guid.NewGuid().ToString("N")[..8]}";

        // PostgreSQL copies template1 during CREATE DATABASE and locks it, so concurrent
        // CREATE DATABASE statements contend even though each DDL takes milliseconds.
        // Migrations below run OUTSIDE this lock, in parallel over independent catalogs.
        await _createDatabaseLock.WaitAsync(ct);
        try
        {
            await using var admin = new NpgsqlConnection(adminConnectionString);
            await admin.OpenAsync(ct);
            await using var create = admin.CreateCommand();
            create.CommandText = $"CREATE DATABASE \"{databaseName}\"";
            await create.ExecuteNonQueryAsync(ct);
        }
        finally
        {
            _createDatabaseLock.Release();
        }

        Console.WriteLine($"Test database created: {databaseName}");

        var cs = new NpgsqlConnectionStringBuilder(adminConnectionString)
        {
            Database = databaseName,
        }.ConnectionString + ";Include Error Detail=true";

        // Fresh catalog => empty journal => every script is applied, including any script edited
        // in place. Reusing a migrated database silently skips those and the suite runs green
        // against a stale schema.
        //
        // Pooling is disabled for the migration connection ONLY. DbUp and the readiness probe use
        // the raw connection string, and Npgsql pools client-side keyed by connection string —
        // i.e. one never-released pool per unique database. On a shared server (max_connections =
        // 100) those idle connections hit the limit after a few hundred migrations: 53300: sorry,
        // too many clients already. The data source returned below still uses pooled `cs`.
        Migrations.Setup(cs + ";Pooling=false");

        var builder = new NpgsqlDataSourceBuilder(cs);
        Migrations.ConfigureDataSource(builder);
        return builder.Build();
    }

    public async ValueTask DisposeAsync()
    {
        if (_container is not null)
        {
            await _container.DisposeAsync();
            _container = null;
        }

        _adminConnectionString = null;
        _containerLock.Dispose();
        _createDatabaseLock.Dispose();
    }

    private async Task<string> GetAdminConnectionStringAsync(CancellationToken ct)
    {
        // Fast path outside the lock; the cold path re-checks inside it, so the first concurrent
        // touches (Mode = All, MaxThreads = 16) start the container exactly once.
        if (_adminConnectionString is { } running)
        {
            return running;
        }

        await _containerLock.WaitAsync(ct);
        try
        {
            if (_adminConnectionString is null)
            {
                var container = TestUtils.CreatePostgreSqlContainer();
                await container.StartAsync(ct);

                _container = container;
                _adminConnectionString = container.GetConnectionString();
            }

            return _adminConnectionString;
        }
        finally
        {
            _containerLock.Release();
        }
    }

    private static string Sanitize(string purpose)
    {
        var chars = purpose
            .ToLowerInvariant()
            .Select(c => char.IsAsciiLetterOrDigit(c) ? c : '_')
            .ToArray();

        return new string(chars, 0, Math.Min(chars.Length, PurposePrefixLength));
    }
}
```

Implementation notes:

- **`TestUtils.CreatePostgreSqlContainer()` stays the sole image source.** Image tag, version and
  builder options must not be duplicated into the fixture, or two test projects drift onto different
  server versions.
- **Counter + GUID in the name.** `Interlocked` makes collisions structurally impossible while the
  initialisation hook runs on many threads. `purpose` keeps the database identifiable in logs.
- **Migrations outside the create lock.** Migrating inside it serialises the whole suite behind every
  migration and can end up slower than the containers you removed.
- **`Pooling=false` scoped to the migration connection.** The returned data source keeps pooling.

## 5. Migrate a test class

Before — a container per test:

```csharp
public sealed class OrderServiceTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = TestUtils.CreatePostgreSqlContainer();
    private NpgsqlDataSource _ds = null!;

    public async ValueTask InitializeAsync()
    {
        await _container.StartAsync(TestContext.Current.CancellationToken);

        var cs = _container.GetConnectionString() + ";Include Error Detail=true";
        Migrations.Setup(cs);

        var builder = new NpgsqlDataSourceBuilder(cs);
        Migrations.ConfigureDataSource(builder);
        _ds = builder.Build();
    }

    public async ValueTask DisposeAsync()
    {
        await _ds.DisposeAsync();
        await _container.DisposeAsync();
    }
}
```

After — a database per test, same granularity:

```csharp
public sealed class OrderServiceTests : IAsyncLifetime
{
    private readonly PostgresFixture _postgres;
    private NpgsqlDataSource _ds = null!;

    public OrderServiceTests(PostgresFixture postgres)
    {
        _postgres = postgres;
    }

    public async ValueTask InitializeAsync()
    {
        _ds = await _postgres.CreateDatabaseAsync(
            nameof(OrderServiceTests),
            TestContext.Current.CancellationToken);
    }

    public async ValueTask DisposeAsync()
    {
        await _ds.DisposeAsync();
    }
}
```

The diff per class is exactly four hunks plus one removed `using`: constructor injection, the
container field, `InitializeAsync`, `DisposeAsync`, and the now-unused
`using Testcontainers.PostgreSql;`. **Any hunk inside a test body is out of scope** — revert it.

Keep `[assembly: Parallelization(Mode = All, MaxThreads = 16)]` and any `xunit.runner.json` as they
are. Lowering the thread count hides a race and gives back the time you just won.

## 6. Roll out in stages

A failure among twenty changed classes is ambiguous; a failure among three is not.

1. **Fixture + 3 small classes.** Prove correctness, measure one class before and after.
2. **Checkpoints of ~4 classes**, running the whole project after each. A missed `using` does not
   surface until you build.
3. **The largest and most state-sensitive class last, alone.** It carries most of the cost, and a
   failure there must be attributable.
4. **Never migrate a class whose test bodies would need edits.** If it cannot be migrated by changing
   plumbing only, leave it on its own container and note it.

If a class fails after migration, revert **that one class** and report it. Do not adjust the test.

## 7. Verify

Counts — prefer the runner's own discovery over grep. With Microsoft.Testing.Platform the project
path is passed via `--project`; under VSTest it is positional. Use whatever the repository documents.

```bash
dotnet test --project path/To/Tests.csproj --list-tests
dotnet test --project path/To/Tests.csproj
dotnet test --project path/To/Tests.csproj        # run twice
```

- **Test count identical before and after.** In xUnit v3 a `[Theory]` row is a test case, so a merge
  that removes methods but keeps rows correctly leaves the count unchanged.
- **Green twice in a row.** One run proves nothing about leaks under `Mode = All`.

Resource accounting:

```bash
# container creates per run: expect 2 (server + Ryuk) or 1 if the reaper is disabled
docker events --filter type=container --filter event=create \
  --format '{{.Actor.Attributes.image}}' > /tmp/creates.txt &
dotnet test --project path/To/Tests.csproj > /tmp/test-output.txt
wait; sort /tmp/creates.txt | uniq -c

# databases created per run, and their uniqueness
grep 'Test database created:' /tmp/test-output.txt | sort -u | wc -l
```

Cross-check arithmetically, because `docker events` alone misleads — `CREATE DATABASE` emits no
container create event, so a low count does not by itself prove nothing booted:

```
container creates = (total tests − non-database tests) − (databases created by the fixture) + reaper
```

It should come out to the reaper alone. Also confirm the migrator logged one journal-created record
**per database**, i.e. no journal was ever reused.

Report wall-clock for a **single migrated class** before and after. The suite total only moves once
most classes are migrated, so quoting it early understates the change.

## 8. xUnit v3 API notes

- **`AssemblyFixture`** — `[assembly: AssemblyFixture(typeof(PostgresFixture))]` in the fixture file.
  Prefer it over `ICollectionFixture`: collection membership silently serialises the classes in the
  collection, quietly reducing parallelism below `MaxThreads` and disguising a race as a slowdown.
  Assembly scope adds no ordering semantics.
- **Injection** — the fixture arrives by **constructor parameter**. Do not add a static
  service-locator; it makes the dependency invisible and invites the container to outlive the
  assembly.
- **`IAsyncLifetime` in v3 returns `ValueTask`**, not `Task`. Copying v2 signatures fails to compile
  with a confusing interface-implementation error.
- **`TestContext.Current.CancellationToken`** replaces the v2 pattern of capturing a token in the
  constructor.
- **Do not add `ICollectionFixture` "for ordering".** If tests depend on order, the isolation unit is
  wrong — fix that instead.
- **`[assembly: Parallelization]`** belongs in a single file (e.g. `XunitSettings.cs`) and must not
  be changed by this refactor.

## 9. Pitfalls

**Connection-pool exhaustion per unique database — the highest-value trap.** Drivers pool
client-side, keyed by connection string. A unique database per test means a unique connection string
per test, hence a new pool per test. Any code path that opens a connection from a **raw string**
rather than the shared `NpgsqlDataSource` — migration runners and readiness probes are the usual ones
— leaves an idle pool that is never released. On a shared server these accumulate until
`max_connections` is reached.

Symptoms: failures appear mid-run, spread evenly across unrelated tests; the failing set grows with
the number of previously created databases (monotonic in run length); the error is
`Npgsql.PostgresException 53300: sorry, too many clients already`.

Fixes, best first:

1. **`;Pooling=false` on the migration/probe connection only.** One line, fixture-local. The returned
   data source keeps pooling.
2. `await using` every data source so pools are released deterministically in teardown.
3. Cap `Max Pool Size` per connection string.
4. **Not** raising `max_connections` — it masks the leak and hides it until CI runs more databases
   concurrently than your machine did.

**Serialising the fast path.** Guard only the cold path (double-checked, as in section 4). Taking the
lock on every `CreateDatabaseAsync` call bottlenecks the first wave of tests.

**Names derived only from the class name.** Twenty tests from one class then differ only by suffix and
collide the moment the suffix scheme changes. Counter + GUID.

**Editing tests to fit.** If a test needs a different isolation unit, the migration is wrong.

**Adding truncation or resets "for safety".** These break any test that relies on state written by a
sibling test in the same class — common, and invisible until it fails.

**Judging isolation by the name prefix.** Emit the created name and count uniqueness with `sort -u`;
do not infer it.

**Treating migration time as free.** After this refactor, migrations usually become the dominant
remaining cost. That is what the `TEMPLATE` row in section 3 exists for.

## 10. Proven result

One real migration — .NET 10 / xUnit v3 / Testcontainers / Npgsql / DbUp, ~700 tests across 20
container-owning classes, all of which turned out to start a container **per test method**, not per
class:

| Metric | Before | After |
|---|---|---|
| Container starts per run | 510+ | 2 (Postgres + Ryuk) |
| Full-suite duration | 1m 51s | ~21 s |
| Test cases | 705 | 705 (unchanged) |
| Databases created per run | n/a | ~523, zero collisions |

The pooling pitfall in section 9 was hit for real mid-rollout: three unrelated tests failed with
`53300` only after enough databases had accumulated, and it was fixed with `;Pooling=false` on the
migration connection. Nothing was fixed by raising `max_connections`, and no test was modified.

Cost still on the table afterwards: migrations re-ran per test (~523 per run). The `TEMPLATE`
database fast path is the next win, not a bigger fixture.

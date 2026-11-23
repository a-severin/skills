---
name: dotnet-central-package-management
description: >
  Set up Central Package Management (CPM) and a shared build policy in a .NET
  repository: `Directory.Packages.props` owns every package version,
  `Directory.Build.props` owns common compiler settings and analyzer packages,
  and `.editorconfig` owns analyzer severities. Use when asked to centralize
  package versions, migrate to `Directory.Packages.props` / `Directory.Build.props`,
  make a build warning-free, enable `TreatWarningsAsErrors`, or wire up
  Roslynator / Meziantou / SDK analyzers. Triggers: CPM, central package
  management, Directory.Packages.props, Directory.Build.props, PackageVersion,
  централизованные версии пакетов, централизованное управление пакетами,
  анализаторы кода.
---

# Central Package Management + shared build policy for .NET

Two files, two responsibilities. Keep them separate — mixing versions into
`Directory.Build.props` is the most common way this setup decays.

| File | Owns | Does **not** own |
|---|---|---|
| `Directory.Packages.props` | `ManagePackageVersionsCentrally`, one `<PackageVersion>` per package | compiler settings, analyzer *references* |
| `Directory.Build.props` | `Nullable`, `ImplicitUsings`, analysis level/mode, warnings-as-errors, analyzer `<PackageReference>`s | package versions |
| `.editorconfig` | per-rule analyzer severity (`dotnet_diagnostic.XXX.severity`) and style/naming conventions | package versions, build policy |

Analyzer packages are the link between the two: they are referenced from
`Directory.Build.props` but their versions still live in
`Directory.Packages.props`, because CPM covers *all* `PackageReference` items.

## When to use

- A solution has `<PackageReference Include="X" Version="1.2.3" />` scattered
  across many `.csproj` files and versions drift.
- The repository should fail the build on warnings/analyzer findings.
- A new repository needs the same build policy from day one.
- Moving an existing repo onto `Directory.Packages.props`.

Do **not** use this to upgrade packages (that is a separate, reviewable change)
or to configure a package feed / source mapping (that is `NuGet.config`).

## How the import works (don't fight it)

With SDK-style projects both files are imported automatically — never add an
`<Import>` for them:

1. `Directory.Build.props` is imported near the top of `Microsoft.Common.props`
   (search upward from each project directory, first match wins).
2. `Directory.Packages.props` is imported later, by `NuGet.props`, which
   `Microsoft.Common.props` imports at the end.
3. Because both are evaluated before the project body, properties defined in
   either file are visible in every `.csproj` that uses the SDK.

Consequences:

- The **nearest** `Directory.Build.props` / `Directory.Packages.props` wins. A
  nested solution with its own file does **not** inherit the parent one unless
  it explicitly imports it:
  ```xml
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />
  ```
- Placing the files at the **repository root** covers every solution/project
  beneath it. That is the normal choice.
- `ImportDirectoryPackagesProps=false` (or `ImportDirectoryBuildProps=false`)
  can opt a subtree out.

## Template: Directory.Packages.props

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <!-- Optional hardening: reject VersionOverride on individual PackageReference items.
         Leave unset if per-project overrides are legitimately needed. -->
    <CentralPackageVersionOverrideEnabled>false</CentralPackageVersionOverrideEnabled>
  </PropertyGroup>

  <ItemGroup Label="System">
    <PackageVersion Include="System.Text.Json" Version="11.0.0" />
    <PackageVersion Include="System.Net.Http.Json" Version="11.0.0" />
  </ItemGroup>

  <ItemGroup Label="Runtime">
    <PackageVersion Include="Npgsql" Version="10.0.3" />
    <PackageVersion Include="AWSSDK.S3" Version="4.1.1" />
  </ItemGroup>

  <ItemGroup Label="Analyzers">
    <PackageVersion Include="Roslynator.Analyzers" Version="5.0.1" />
    <PackageVersion Include="Roslynator.CodeAnalysis.Analyzers" Version="5.0.1" />
    <PackageVersion Include="Roslynator.Formatting.Analyzers" Version="5.0.1" />
    <PackageVersion Include="Meziantou.Analyzer" Version="3.0.341" />
  </ItemGroup>

  <!-- Test-only analyzers are centralised too, but referenced only by test projects. -->
  <ItemGroup Label="Testing analyzers">
    <PackageVersion Include="xunit.analyzers" Version="2.3.0" />
    <PackageVersion Include="FluentAssertions.Analyzers" Version="0.34.1" />
  </ItemGroup>
</Project>
```

Rules:

- Every package used anywhere in the repo gets exactly **one**
  `<PackageVersion>`. Duplicate `Include` entries are an error.
- Group with `Label`/comments; ordering is cosmetic but a stable grouping keeps
  diffs reviewable.
- A `PackageVersion` alone does **not** add a dependency. Referencing is still
  done per project (or once in `Directory.Build.props` for analyzers).

## Template: Directory.Build.props

```xml
<Project>
    <PropertyGroup>
        <Nullable>enable</Nullable>
        <ImplicitUsings>enable</ImplicitUsings>

        <!-- SDK analyzers: latest ruleset, Recommended set (not All — see rollout) -->
        <AnalysisLevel>latest</AnalysisLevel>
        <AnalysisMode>Recommended</AnalysisMode>

        <!-- Warning-free by policy: compiler + analyzer + NuGet warnings fail the build -->
        <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
        <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
        <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>

        <!-- NU1507 = "multiple package sources, consider package source mapping".
             Keep it a warning, not a build failure. -->
        <WarningsNotAsErrors>$(WarningsNotAsErrors);NU1507</WarningsNotAsErrors>

        <GenerateDocumentationFile>true</GenerateDocumentationFile>
        <!-- 1591: missing XML comment; 1701/1702: assembly binding;
             NU1902/NU1903: NuGet audit vulnerabilities.
             SQL80001 (SQL literal in C# string) comes from a SQL-in-string analyzer —
             remove it if the project does not use one. -->
        <NoWarn>$(NoWarn);1591;1701;1702;SQL80001;NU1902;NU1903</NoWarn>
    </PropertyGroup>

    <!-- Analyzers apply to every project; PrivateAssets=all keeps them out of consumers. -->
    <ItemGroup Label="Analyzers">
        <PackageReference Include="Roslynator.Analyzers" PrivateAssets="all" />
        <PackageReference Include="Roslynator.CodeAnalysis.Analyzers" PrivateAssets="all" />
        <PackageReference Include="Roslynator.Formatting.Analyzers" PrivateAssets="all" />
        <PackageReference Include="Meziantou.Analyzer" PrivateAssets="all" />
    </ItemGroup>
</Project>
```

Property notes:

- `TreatWarningsAsErrors` only promotes **compiler + NuGet** warnings.
  Analyzer diagnostics need `CodeAnalysisTreatWarningsAsErrors`.
- `EnforceCodeStyleInBuild` makes `.editorconfig` style rules run during build;
  combined with warnings-as-errors, style violations become build failures.
- `AnalysisMode=Recommended` enables the recommended subset. `All` is much
  noisier — do not start there on an existing codebase.
- `GenerateDocumentationFile=true` implies CS1591 for every undocumented public
  member. Suppress it if documentation is not part of the policy.
- `NoWarn` handles **compiler** warnings; analyzer suppressions belong in
  `.editorconfig`, not `NoWarn`.

## Template: .editorconfig

Keep `root = true` at the repository root, otherwise a parent directory's file
silently leaks in. Minimal analyzer block:

```ini
root = true

[*.cs]
# Downgrade or silence noisy rules explicitly, with a comment per rule.
dotnet_diagnostic.CA2007.severity = none        # ConfigureAwait — not needed in ASP.NET Core
dotnet_diagnostic.CA1062.severity = none        # null checks — Nullable + ThrowIfNull cover this
dotnet_diagnostic.CA1707.severity = none        # underscores in test names
dotnet_diagnostic.CA1031.severity = suggestion  # catch general exception types — allowed here

# SDK naming rules (CA1700 series) — optional, project-specific
dotnet_analyzer_diagnostic.category-Naming.severity = none
```

- One `dotnet_diagnostic.<ID>.severity` per accepted suppression, with a
  comment saying why. Severities: `none` (off), `silent`, `suggestion`,
  `warning`, `error`.
- Prefer downgrading a rule to `suggestion` over `none` when it is useful in
  review but does not belong in the build gate.
- Style/naming rules (`dotnet_style_*`, `csharp_style_*`, `dotnet_naming_*`)
  are repository taste. Do **not** copy another repo's block verbatim; adopt
  only what the team will enforce.
- Compiler warnings cannot be controlled here — use `NoWarn`.

## Migration procedure

Work in this order so every step leaves a green build.

### 1. Inventory

```bash
# See every package and the versions in use
grep -rn 'PackageReference' --include='*.csproj' . | grep -v '/bin/\|/obj/'

# Check for existing props files / NuGet.config
find . -maxdepth 3 \( -name 'Directory.Build.props' -o -name 'Directory.Packages.props' -o -iname 'nuget.config' \) -not -path './.git/*'
```

Build the list of distinct package IDs. When one package appears with several
versions, choose the **highest** (or the version compatible with all TFMs) and
note it in the change description. Bumping versions is a separate decision.

### 2. Create Directory.Packages.props

Add `ManagePackageVersionsCentrally` and one `PackageVersion` per package,
grouped. Put it at the repository root (or the common ancestor of all projects).

### 3. Strip `Version` from every PackageReference

This is the mandatory CPM step: a `PackageReference` that still carries a
`Version` while CPM is on produces **NU1008** ("Projects that use central
package version management should not define the version on the PackageReference
items"). Restore will fail until every one is removed.

Keep the metadata that is *not* a version:

```xml
<!-- before -->
<PackageReference Include="xunit.analyzers" Version="2.3.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>

<!-- after -->
<PackageReference Include="xunit.analyzers">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>
```

`PrivateAssets`, `IncludeAssets`, `ExcludeAssets`, `GeneratePathProperty` and
`Aliases` stay on the `PackageReference`. Only `Version` (and `VersionOverride`)
is removed.

Run `dotnet restore`. A missing central entry surfaces as **NU1010** ("The
PackageReference items do not have corresponding PackageVersion") — add the
missing `PackageVersion`.

### 4. Create Directory.Build.props — staged

For an existing codebase, do not flip everything on at once:

1. **Phase A — CPM only.** Build must stay identical to before.
2. **Phase B — language defaults.** Add `Nullable` and `ImplicitUsings`; fix
   compile errors.
3. **Phase C — analyzers, not fatal.** Add analyzer packages and
   `AnalysisMode=Recommended`, but leave `TreatWarningsAsErrors` off. Read the
   warning report and decide each rule: fix, downgrade in `.editorconfig`, or
   silence with a comment.
4. **Phase D — enforce.** Turn on `TreatWarningsAsErrors`,
   `CodeAnalysisTreatWarningsAsErrors` and `EnforceCodeStyleInBuild`. Add the
   `WarningsNotAsErrors` / `NoWarn` entries needed for warnings that are not
   actionable in this repo.

For a **new** repository, start at Phase D — there is no legacy debt.

### 5. Analyzer placement

- **All projects:** Roslynator + Meziantou via `Directory.Build.props`, always
  with `PrivateAssets="all"`.
- **Test projects only:** `xunit.analyzers`, `FluentAssertions.Analyzers`, and
  similar — reference them in each test `.csproj` (or a test-specific
  `Directory.Build.props`), never globally, or production projects will be
  told about xUnit rules that do not apply.
- Their versions still go in `Directory.Packages.props`.

### 6. Verify

```bash
# No package version may remain in a project file (must print nothing)
grep -rnE 'PackageReference[^>]*[[:space:]]Version=' --include='*.csproj' .

# Restore and build both configurations
dotnet restore
dotnet build -c Debug
dotnet build -c Release

# Confirm the policy is really active: this should fail on a warning
dotnet build -c Release
```

Additional checks:

- `dotnet list package --outdated` still runs (versions resolve normally).
- The build output has **0 warnings**. If `dotnet build` succeeds with warnings,
  `TreatWarningsAsErrors` is not reaching that project — check for a nested
  `Directory.Build.props` shadowing the root one.
- `dotnet add package X` now writes to `Directory.Packages.props` instead of
  the `.csproj`; confirm by running it and inspecting the diff.

## Gotchas

- **NU1008** — `Version` left on a `PackageReference` under CPM. Remove it.
- **NU1010** — referenced package without a `PackageVersion`. Add one.
- **NU1507** — multiple package sources without source mapping. Excluded from
  warnings-as-errors here; fix it properly with `NuGet.config` source mapping if
  package confusion is a real risk.
- **A nested `Directory.Build.props` replaces the root one** — it is not
  additive. Import the parent explicitly if you need inheritance.
- **`NoWarn` vs `.editorconfig`** are different systems. Compiler warnings →
  `NoWarn` / `WarningsNotAsErrors`; analyzer diagnostics → `dotnet_diagnostic`.
- **`AnalysisMode=All` on day one** produces hundreds of findings and turns a
  refactor into a rewrite. Use `Recommended`, then promote individual rules.
- **Do not put `PackageVersion` items in `Directory.Build.props`** — NuGet reads
  them from `Directory.Packages.props`, and splitting them defeats the single
  source of truth.
- **Do not copy `NoWarn` lists between repos** — entries such as `SQL80001`
  exist because of a specific analyzer that pulls SQL literals out of C#.
  Carry over only codes the target repo actually emits.
- **`netstandard2.0` projects work with CPM** — nothing special, but they
  constrain which package versions are usable, so the centrally chosen version
  must support the lowest TFM in the repo.
- **Central versions are not "latest"** — CPM makes a version single-sourced,
  not automatically upgraded. Upgrades remain explicit changes to
  `Directory.Packages.props`.

## Reference implementation

A working instance of exactly this layout (CPM + shared warning-free build policy
+ Roslynator/Meziantou analyzers) lives in `space-server`:

- `Directory.Packages.props` — `ManagePackageVersionsCentrally`, grouped
  `PackageVersion` items including analyzer packages.
- `Directory.Build.props` — `Nullable`, `ImplicitUsings`,
  `AnalysisLevel=latest`, `AnalysisMode=Recommended`, `TreatWarningsAsErrors`,
  `CodeAnalysisTreatWarningsAsErrors`, `EnforceCodeStyleInBuild`,
  `GenerateDocumentationFile`, `WarningsNotAsErrors=NU1507`, analyzer
  `PackageReference`s with `PrivateAssets="all"`.
- `.editorconfig` — `root = true` plus per-rule severities for the accepted
  suppressions.

Read those three files before adapting the templates, then follow the migration
procedure above. Keep the responsibility split intact: versions in
`Directory.Packages.props`, policy in `Directory.Build.props`, rule severities in
`.editorconfig`.

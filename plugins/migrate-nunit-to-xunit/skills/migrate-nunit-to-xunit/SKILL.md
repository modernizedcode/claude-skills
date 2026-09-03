---
name: migrate-nunit-to-xunit
description: >
  Convert .NET test projects from NUnit 3.x or 4.x to xUnit v3. Use for
  replacing NUnit packages, [TestFixture]/[Test]/[TestCase], Assert.That
  constraint-model and classic assertions, SetUp/TearDown lifecycle,
  TestCaseSource, and NUnit parallelization with xUnit v3 equivalents while
  preserving the current runner (VSTest or MTP) and target framework.
  DO NOT USE FOR: xUnit v2 to v3 upgrades, NUnit version upgrades, migrations
  to MSTest or TUnit, or runner-only VSTest to MTP migrations.
---

# NUnit -> xUnit v3 Migration

Convert NUnit 3.x/4.x tests to xUnit v3 without changing the target framework or test runner. A successful migration builds, discovers the same tests, and preserves pass/fail results and execution semantics.

## Scope

Use this skill only when the project contains NUnit packages or source and the user wants xUnit. If the project already uses xUnit and contains no NUnit tests, report that no framework migration is needed and make no changes.

Do not combine this framework conversion with a target-framework upgrade, an assertion-library rollout across non-NUnit projects, or a VSTest/MTP runner migration. Complete and verify one migration before starting another.

## Response Mode

- **Full migration request:** inspect the project, make the edits, build, and run tests. Do not stop after giving a plan.
- **Focused compile error or API question:** inspect the relevant code and apply only that mapping. Do not narrate the entire workflow.
- **Unsupported target framework:** stop before changing packages. xUnit v3 requires .NET Framework 4.7.2+ or .NET 8+ for test projects; offer a separately approved TFM upgrade or an xUnit v2 target instead.

## Assertion Target

Default to **FluentAssertions** for converted assertions when the repository already references it; this usually unifies style because NUnit repos commonly mix `Assert.That` with `.Should()`. Fall back to native xUnit `Assert` when FluentAssertions is absent and the user does not want it added. Never rewrite already-passing native xUnit or FluentAssertions code in other projects — convert only the NUnit assertions.

FluentAssertions licensing/compat: v8+ requires a commercial licence; stay on 7.x. xUnit v3 integration (correct failure exception type) requires FluentAssertions 7.1.0+.

For detailed mappings and examples, search [`references/mapping-cheatsheet.md`](references/mapping-cheatsheet.md) for constructs actually present in the project and read only the matching sections. Do not load or reproduce the whole reference.

## Fast Path

For a routine project migration, converge in four phases: one batched discovery read/search, one edit pass, one `dotnet test`, and one concise result. Do not:

- list a directory and then reread the same files through another tool
- run separate restore, build, and test commands when `dotnet test` is sufficient
- rerun a passing test command or inspect unchanged files for confirmation

Use an existing CI/test result as the parity baseline when available. Run a new pre-edit baseline when counts are unavailable and the migration contains data-driven tests, lifecycle fixtures, runtime skips, custom attributes, shared state, or other behavior whose parity cannot be established from source alone.

## Workflow

### 1. Establish the baseline

1. In one discovery pass, batch-read the test projects plus `Directory.Build.props`, `Directory.Packages.props`, `global.json`, and CI test invocations, and search the source for the high-risk constructs below.
2. Identify the runner (VSTest via `Microsoft.NET.Test.Sdk` + adapter, or MTP) and preserve it.
3. Record the target frameworks and stop if xUnit v3 does not support them.
4. If a new baseline is required, run the existing test command once and record discovered, passed, failed, and skipped counts.
5. Inventory high-risk constructs before editing:
   - NUnit `[Theory]` + `[Datapoint]`/`[DatapointSource]` — **name-collides with `Xunit.TheoryAttribute` and still compiles after the `using` swap while meaning something completely different**
   - `[TestCaseSource]`, `[TestFixtureSource]`, `[Values]`, `[Range]`, `[Random]`, `[Combinatorial]`, `[Pairwise]`, `[Sequential]`, custom attributes
   - `[OneTimeSetUp]`/`[OneTimeTearDown]`, `[SetUpFixture]`, `[FixtureLifeCycle]`
   - `[Parallelizable]`/`[NonParallelizable]`/`[LevelOfParallelism]`/`[SingleThreaded]` — or their **absence** (see Step 5)
   - `[Order]`, `[Retry]`, `[Repeat]`, `[Explicit]`, `[Apartment]`, `[Platform]`, `[Culture]`, `[SetCulture]`/`[SetUICulture]`, `[DefaultFloatingPointTolerance]`
   - `Assume.That`, `Assert.Ignore`, `Assert.Inconclusive`, `Assert.Pass`, `Assert.Multiple`, `Assert.EnterMultipleScope` (NUnit 4.2+)
   - **`internal`/non-public test classes** — NUnit discovers them, xUnit does not; they vanish from discovery with no failure and no skip
   - `TestContext.*` usage, `StringAssert`/`CollectionAssert`/`FileAssert`/`DirectoryAssert`, `using NUnit.Framework.Legacy` (NUnit 4.0–4.4 home of the classic asserts)

### 2. Replace packages without switching runners

Remove NUnit packages from project files and central package files: `NUnit`, `NUnit3TestAdapter`, `NUnit.Analyzers`, `NUnit.ConsoleRunner`, and NUnit-specific companion packages being replaced.

Add xUnit v3, preserving the detected runner:

```xml
<!-- VSTest: keep the existing Microsoft.NET.Test.Sdk reference; the 3.x adapter builds
     against Microsoft.TestPlatform.ObjectModel 17.13, so bump very old pins (< 17.12)
     if discovery fails -->
<PackageReference Include="xunit.v3" Version="3.2.2" />
<PackageReference Include="xunit.runner.visualstudio" Version="3.1.5" PrivateAssets="all" />
```

In 3.x, `xunit.v3` is the MTP-v1 flavour (`xunit.v3` -> `xunit.v3.mtp-v1`); variants are `xunit.v3.mtp-v1`, `xunit.v3.mtp-v2`, and `xunit.v3.mtp-off`. **VSTest repo:** use plain `xunit.v3` (or `xunit.v3.mtp-off`) and do not set `<TestingPlatformDotnetTestSupport>` or a `global.json` test-runner entry — either one reroutes `dotnet test` to MTP and breaks the runner-unchanged guarantee. **MTP repo:** `xunit.v3.mtp-v1` plus `<TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>` (SDK 8/9) or the `global.json` test-runner setting (SDK 10+).

xUnit v3 test projects are executables — add `<OutputType>Exe</OutputType>` to the project. The `xunit.v3` metapackage brings `xunit.analyzers` transitively; do not add it separately. Replace `global using NUnit.Framework;` (or a `<Using Include="NUnit.Framework" />`) with the repo's xUnit convention — prefer `<Using Include="Xunit" />` if sibling projects use it.

Do not change `TargetFramework`. VSTest loggers (e.g. `JunitXml.TestLogger`) and coverlet keep working unchanged under the VSTest runner.

### 3. Perform the mechanical conversion

Apply the common rewrites first:

| NUnit | xUnit v3 |
|---|---|
| `[TestFixture]` | remove (no class attribute) |
| `[Test]` | `[Fact]` |
| `[TestCase(...)]` | `[Theory]` + `[InlineData(...)]` |
| `[TestCaseSource(nameof(X))]` | `[Theory]` + `[MemberData(nameof(X))]` |
| `[SetUp]` | constructor (fields become `readonly`, `= null!` suppressions go away) |
| `[TearDown]` | `IDisposable.Dispose()` |
| `[Ignore("reason")]` | `[Fact(Skip = "reason")]` |
| `[Category("X")]` | `[Trait("Category", "X")]` |
| `Assert.That(x, Is.EqualTo(y))` | `x.Should().Be(y)` / `Assert.Equal(y, x)` |
| `Assert.That(x, Is.True)` | `x.Should().BeTrue()` / `Assert.True(x)` |
| `Assert.That(x, Is.Null)` | `x.Should().BeNull()` / `Assert.Null(x)` |

### 4. Resolve semantic mappings

Load the mapping cheatsheet for every high-risk construct found in Step 1. These rules are mandatory:

- NUnit `Is.InstanceOf<T>` is assignable-from and maps to FA `BeAssignableTo<T>` / xUnit `Assert.IsAssignableFrom<T>`; NUnit `Is.TypeOf<T>` is exact-type and maps to FA `BeOfType<T>` / xUnit `Assert.IsType<T>`. Do not swap them.
- NUnit `Assert.Throws<T>` is exact-type: FA `ThrowExactly<T>` / xUnit `Assert.Throws<T>`. NUnit `Assert.Catch<T>` permits derived types: FA `Throw<T>` / xUnit `Assert.ThrowsAny<T>`. FA's plain `Throw<T>` permits derived types — using it for `Assert.Throws<T>` loosens the test.
- `Assert.Multiple` and `Assert.EnterMultipleScope` (NUnit 4.2+, the `using`-scope form) both map to FluentAssertions `AssertionScope`; with native xUnit, split into sequential asserts and note the lost report-all-failures semantics.
- Any `internal` or otherwise non-public test class must be made `public`. xUnit discovers only public test classes, so leaving one non-public silently drops every test in it from the run — no failure, no skip, just a lower discovered count.
- Runtime skips (`Assert.Ignore`, `Assert.Inconclusive`, `Assume.That`) map to xUnit v3 `Assert.Skip` / `Assert.SkipUnless(condition, ...)` / `Assert.SkipWhen(...)`. Never convert a runtime skip to the compile-time `[Fact(Skip = ...)]` — that permanently excludes the test where it should have run.
- `Assert.That(collection, Is.EqualTo(expected))` compares element-wise: FA `.Should().Equal(expected)` (ordered). `Is.EquivalentTo` ignores order: FA `.Should().BeEquivalentTo(expected)`. Native xUnit `Assert.Equal` on sequences is element-wise and safe.
- `[TestCase]` arguments become `[InlineData]`; audit `ExpectedResult =` usages — the return-value comparison must become an explicit assertion.
- `TestContext.WriteLine`/`Progress.WriteLine` map to a constructor-injected `ITestOutputHelper` (namespace `Xunit`, not `Xunit.Abstractions`, in v3) or `TestContext.Current.TestOutputHelper`.
- NUnit assertion `message`/`args` parameters have no native equivalent on most xUnit asserts. FluentAssertions: pass them as the `because` arguments — `a.Should().Be(b, "ids differed for {0}", id)`. Native xUnit: only `Assert.True`/`False`/`Fail` take a message. Never drop the message silently — it is often the only thing that makes the failure diagnosable.
- `[DefaultFloatingPointTolerance(x)]` (method/fixture/assembly) silently applies to every `Is.EqualTo` on `float`/`double` in scope. There is no xUnit equivalent — each affected comparison must become explicit: FA `.Should().BeApproximately(expected, x)` / xUnit `Assert.Equal(expected, actual, x)`. Removing the attribute without doing this converts tolerant comparisons to exact ones.
- `[Explicit]` maps to `[Fact(Explicit = true)]` / `[Theory(Explicit = true)]`; explicit tests are skipped unless the runner is told to include them (VSTest RunSettings `Explicit`, MTP `--explicit`).
- `[Retry]`, `[Order]`, `[Repeat]`, `[Apartment]` have no built-in xUnit v3 equivalent — flag each occurrence for an explicit user decision. Never delete the behavior silently.

Apply the mechanical and semantic rewrites in one edit pass when the inventory makes the required mappings clear. Use compiler errors from final verification to drive only unresolved conversions.

### 5. Preserve lifecycle, fixture scope, and parallelization

- `[SetUp]` -> constructor; `[TearDown]` -> `IDisposable`. Async setup/teardown -> `IAsyncLifetime` (v3: `ValueTask InitializeAsync()`, extends `IAsyncDisposable`).
- `[OneTimeSetUp]`/`[OneTimeTearDown]` (per-fixture, shared across that class's tests) -> `IClassFixture<T>`.
- `[SetUpFixture]` -> `[assembly: AssemblyFixture(typeof(T))]` (xUnit v3) when there is one at assembly scope. `[SetUpFixture]` is namespace-scoped — multiple fixtures in different namespaces cannot collapse into one assembly fixture; use collection fixtures per affected group instead.
- `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]` -> delete; instance-per-test is xUnit's only model. Default NUnit lifecycle (single fixture instance) means instance state shared across tests becomes per-test in xUnit — audit mutable instance fields that tests expect to persist.

**Parallelization inverts.** NUnit runs everything on one thread unless `[Parallelizable]` is present; xUnit runs test classes in parallel by default (each class is its own collection). Unless the source opted into parallelism:

- If fixtures share mutable static state, files, databases, or ports, group them with `[Collection("...")]` or disable with `[assembly: CollectionBehavior(DisableTestParallelization = true)]`.
- If the source used `[Parallelizable(ParallelScope.Fixtures)]`, xUnit's default already matches.
- Before applying a parallelization decision, state what the source serialized and how the target preserves it.

### 6. Verify parity

1. Run tests once with the same runner, filter, and configuration used for the baseline.
2. Compare discovered, passed, failed, and skipped counts. Expected difference: NUnit reports `Assert.Inconclusive`/failed `Assume.That` as *inconclusive*, which many loggers count separately; xUnit reports `Assert.Skip` as *skipped* — reconcile those columns rather than treating the shift as a regression.
3. Investigate every difference before declaring completion:
   - missing cases -> `[TestCase]`/`TestCaseSource` conversion, `MemberData` shape
   - an entire class missing -> a test class left `internal`/non-public
   - changed exception behavior -> exact-vs-derived assertion mapping
   - new flaky/shared-state failures -> parallelization inversion (Step 5)
   - silently skipped tests -> runtime-skip conversion
4. Confirm no NUnit package, namespace, attribute, or `Assert.That` remains unless explicitly documented for manual follow-up.

## Completion Criteria

- NUnit version and runner were identified; runner and target framework stayed unchanged
- NUnit packages and source constructs were converted
- Assertion target matches the repo's existing style
- Parallelization decision is explicit
- Build succeeds; test discovery and result counts match the baseline
- Every test class is `public`
- Any unsupported construct (`[Retry]`, `[Order]`, custom attributes) is called out rather than approximated

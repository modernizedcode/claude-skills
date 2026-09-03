# NUnit -> xUnit v3 Mapping Cheatsheet

Comprehensive reference loaded by the `migrate-nunit-to-xunit` skill. Look up specific NUnit constructs and their xUnit v3 equivalents. Assertion rows show both the **FluentAssertions** target (default when the repo uses it; 7.1.0+ required for xunit.v3, stay on 7.x — v8+ needs a commercial licence) and the **native xUnit** target.

## Table of contents

- [1. Test discovery (class + method attributes)](#1-test-discovery-class--method-attributes)
- [2. Data-driven tests](#2-data-driven-tests)
- [3. Assertions](#3-assertions)
- [4. Lifecycle and fixtures](#4-lifecycle-and-fixtures)
- [5. Output / TestContext](#5-output--testcontext)
- [6. Parallelization](#6-parallelization)
- [7. Assembly-level attributes](#7-assembly-level-attributes)
- [8. Packages and project file](#8-packages-and-project-file)
- [9. Companion / extension libraries](#9-companion--extension-libraries)

## 1. Test discovery (class + method attributes)

| NUnit | xUnit v3 |
|---|---|
| `[TestFixture]` | Remove — xUnit has no class attribute |
| `internal` / non-public test class | **Make it `public`** — NUnit discovers non-public fixtures, xUnit does not. Leaving it non-public drops every test in the class from discovery with no failure and no skip; only the discovered count reveals it |
| `[TestFixture(arg)]` (parameterized fixture) | **Manual** — no equivalent; extract the parameter into a base class + derived classes, or a theory |
| `[TestFixtureSource(...)]` | **Manual** — no fixture-level data source; restructure as derived classes or fold the fixture argument into theory rows |
| `[Test]` | `[Fact]` |
| `[Test(Description = "x")]` | `[Fact(DisplayName = "x")]` — note `DisplayName` *replaces* the reported test name (changes CI report keys and name-based filters); use `[Trait("Description", "x")]` when the name must stay stable |
| `[TestCase(...)]` | `[Theory]` + `[InlineData(...)]` |
| NUnit `[Theory]` + `[Datapoint]` / `[DatapointSource]` | **Manual, high risk** — the attribute name collides with `Xunit.TheoryAttribute` and still compiles after the `using` swap while meaning something completely different (no data source, combinatorial semantics lost). Enumerate the datapoint combinations explicitly into `[MemberData]`/`TheoryData`, and convert the `Assume.That` guards inside the body to `Assert.SkipUnless(...)` or drop the excluded rows |
| `[Ignore("reason")]` (attribute) | `[Fact(Skip = "reason")]` / `[Theory(Skip = "reason")]` |
| `[Category("Unit")]` | `[Trait("Category", "Unit")]` |
| `[Property("k", "v")]` | `[Trait("k", "v")]` |
| `[TestOf(typeof(X))]`, `[Author("...")]`, `[Description]` on classes | Metadata only — `[Trait(...)]` or drop, per repo convention |
| `[Explicit]` | `[Fact(Explicit = true)]` / `[Theory(Explicit = true)]` — explicit tests are skipped unless the runner includes them (VSTest RunSettings `Explicit`, MTP `--explicit`) |
| `[Order(n)]` | **Manual** — xUnit deliberately has no ordering; needs `ITestCaseOrderer` or test redesign. Flag it |
| `[Retry(n)]` | **Manual** — no built-in retry (xUnit considers it out of scope); `xRetry.v3` (`[RetryFact(n, delayMs)]`) is the concrete community option. Flag it |
| `[Repeat(n)]` | **Manual** — no equivalent; flag it |
| `[Timeout(ms)]` (method) | `[Fact(Timeout = ms)]` — works for sync and async tests in v3 (the v2 async-only restriction was removed). Timeout signals the test's `CancellationToken` rather than aborting the thread, so a test that never observes the token can still overrun |
| `[Timeout(ms)]` (fixture-level) | No class-level timeout — apply `Timeout` per test method |
| `[Apartment(ApartmentState.STA)]` | No built-in equivalent. `Xunit.StaFact` 3.x supports xUnit v3 3.x (`[StaFact]`, `[WpfFact]`, `[WinFormsFact]`, `[UIFact]` + `Theory` variants) — flag it as a package decision rather than deleting the attribute |
| `[Platform]`, `[Culture]` | Runtime skip: `Assert.SkipUnless(OperatingSystem.IsWindows(), "...")` etc. |
| `[SetCulture("...")]` / `[SetUICulture("...")]` (method/fixture) | **Manual** — set and restore `CultureInfo.CurrentCulture` in the constructor/`Dispose` (or a fixture); no attribute equivalent |
| `[SingleThreaded]` | Covered by xUnit's model — methods within a class never run in parallel; if the intent was cross-class, use a `[Collection]` |
| `[MaxTime(ms)]` | `[Fact(Timeout = ms)]` (semantics differ slightly: MaxTime fails, Timeout cancels — note it) |
| Custom attribute deriving from NUnit types | **Manual** — reimplement against xUnit v3 extensibility; flag it |

## 2. Data-driven tests

| NUnit | xUnit v3 |
|---|---|
| `[TestCase(1, 2)]` | `[InlineData(1, 2)]` |
| `[TestCase(1, 2, ExpectedResult = 3)]` with non-void return | Void test: `[InlineData(1, 2, 3)]` + explicit assertion on the extra parameter. **The implicit return-value comparison must become an explicit assert** |
| `[TestCase(..., TestName = "x")]` | `[InlineData]` has no name parameter. Move the rows to a `[MemberData]` source returning `IEnumerable<ITheoryDataRow>` and name per row: `new TheoryDataRow<int, int>(1, 2).WithTestDisplayName("x")` (v3; the `Label` property customises just the row suffix) |
| `[TestCaseSource(nameof(Cases))]` (static field/property/method returning `IEnumerable`) | `[MemberData(nameof(Cases))]` returning `IEnumerable<object[]>` or `TheoryData<...>` (prefer `TheoryData` — analyzer-checked) |
| `[TestCaseSource(typeof(X), nameof(X.Cases))]` | `[MemberData(nameof(X.Cases), MemberType = typeof(X))]` |
| `TestCaseSource` yielding `TestCaseData` objects | Convert to `TheoryData<...>` rows; `TestCaseData.Returns(...)` becomes an explicit assertion; `.SetName(...)` -> `.WithTestDisplayName(...)` on a `TheoryDataRow` |
| `[ValueSource(nameof(Xs))]` on a parameter | Restructure into `[MemberData]` producing full rows |
| `[Values(1, 2, 3)]` on a parameter | Expand: one `[InlineData]` per combination, or generate rows in `[MemberData]` |
| `[Values]` on a `bool`/enum parameter | Expand all values explicitly |
| `[Range(1, 5)]`, `[Random(n)]` | Generate rows in `[MemberData]` (avoid nondeterministic `Random` — flag it) |
| `[Combinatorial]` (default), `[Pairwise]`, `[Sequential]` | **Manual** — compute combinations in `[MemberData]`; `[Pairwise]` has no equivalent, flag it |

**`TheoryData` example:**

```csharp
public static TheoryData<int, int, int> AdditionCases => new()
{
    { 1, 2, 3 },
    { 2, 2, 4 },
};

[Theory]
[MemberData(nameof(AdditionCases))]
public void Add_ReturnsSum(int a, int b, int expected) =>
    Calculator.Add(a, b).Should().Be(expected);
```

**`[MemberData]` constraints not present in NUnit:** the source member must be **public** *and* **static** (xUnit1017 / xUnit1018) — private static sources, which NUnit's `TestCaseSource` accepts, must be made public. Row values should also be serializable, or the individual cases collapse into one Test Explorer entry and cannot be filtered or re-run individually.

## 3. Assertions

NUnit 4 moved classic asserts (`Assert.AreEqual`) to `ClassicAssert`; both spellings appear below where they differ. In NUnit 4.0–4.4 `ClassicAssert`/`CollectionAssert`/`StringAssert` live in `NUnit.Framework.Legacy` (from 4.5 the classic asserts are back in `NUnit.Framework`) — search for the `Legacy` using too. `Assert.That(x)` with no constraint asserts truth. NUnit 4.2+ also spells multiple-assert blocks as `using (Assert.EnterMultipleScope())` rather than `Assert.Multiple(...)` — search for both.

**Assertion messages.** NUnit's trailing `message`/`args` parameters have no native equivalent on most xUnit asserts. FluentAssertions: pass them as the `because` arguments — `a.Should().Be(b, "ids differed for {0}", id)`. Native xUnit: only `Assert.True`/`Assert.False`/`Assert.Fail` take a message, so either restate the assertion as `Assert.True(cond, msg)` or move the context into the assertion's inputs/`ITestOutputHelper`. Do not drop the message silently — it is often the only thing that makes the failure diagnosable.

### 3.1 Equality, null, reference, boolean

> **Semantic trap:** NUnit `Is.EqualTo` compares across numeric types (`Is.EqualTo(1.0)` matches an `int` 1; `int`/`long`/`byte` compare equal). `Assert.Equal` and FA `.Should().Be()` are type-strict. Comparisons that relied on NUnit's coercion will fail to compile or start failing — cast explicitly, or use `BeApproximately`/`Assert.Equal(e, a, tolerance)` when the intent was numeric-value equality.
>
> Also check for `[DefaultFloatingPointTolerance(x)]` at method/fixture/assembly scope — it silently loosens every `float`/`double` `Is.EqualTo` in scope; each affected comparison must become an explicit `BeApproximately`/tolerance overload.

| NUnit | FluentAssertions | native xUnit |
|---|---|---|
| `Assert.That(a, Is.EqualTo(b))` | `a.Should().Be(b)` | `Assert.Equal(b, a)` |
| `Assert.That(a, Is.Not.EqualTo(b))` | `a.Should().NotBe(b)` | `Assert.NotEqual(b, a)` |
| `Assert.That(a, Is.EqualTo(b).Within(0.01))` | `a.Should().BeApproximately(b, 0.01)` | `Assert.Equal(b, a, 0.01)` |
| `Assert.That(a, Is.EqualTo(b).IgnoreCase)` | `a.Should().BeEquivalentTo(b)` (strings: case-insensitive) | `Assert.Equal(b, a, ignoreCase: true)` |
| `Assert.That(a, Is.EqualTo(b).Using(comparer))` | `a.Should().Be(b)` won't take a comparer — use `a.Should().BeEquivalentTo(b, o => o.Using(comparer))` or assert `comparer.Compare(a, b) == 0` | `Assert.Equal(b, a, comparer)` (`IEqualityComparer<T>`) |
| `Assert.That(a, Is.SameAs(b))` | `a.Should().BeSameAs(b)` | `Assert.Same(b, a)` |
| `Assert.That(x, Is.Null)` | `x.Should().BeNull()` | `Assert.Null(x)` |
| `Assert.That(x, Is.Not.Null)` | `x.Should().NotBeNull()` | `Assert.NotNull(x)` |
| `Assert.That(x, Is.True)` / `Assert.That(x)` | `x.Should().BeTrue()` | `Assert.True(x)` |
| `Assert.That(x, Is.False)` | `x.Should().BeFalse()` | `Assert.False(x)` |
| `Assert.That(x, Is.Default)` | `x.Should().Be(default(T))` | `Assert.Equal(default, x)` |
| `ClassicAssert.AreEqual(e, a)` / NUnit3 `Assert.AreEqual` | `a.Should().Be(e)` | `Assert.Equal(e, a)` |

### 3.2 Type checks

> **Semantic trap:** NUnit `Is.InstanceOf<T>` is **assignable-from** (derived types pass); `Is.TypeOf<T>` is **exact**. Map them separately.

| NUnit | FluentAssertions | native xUnit |
|---|---|---|
| `Assert.That(x, Is.InstanceOf<T>())` | `x.Should().BeAssignableTo<T>()` | `Assert.IsAssignableFrom<T>(x)` |
| `Assert.That(x, Is.TypeOf<T>())` | `x.Should().BeOfType<T>()` | `Assert.IsType<T>(x)` |
| `Assert.That(x, Is.Not.InstanceOf<T>())` | `x.Should().NotBeAssignableTo<T>()` | `Assert.False(x is T)` |
| `Assert.That(x, Is.AssignableTo<T>())` | `x.Should().BeAssignableTo<T>()` | `Assert.IsAssignableFrom<T>(x)` |

### 3.3 Comparison / range

| NUnit | FluentAssertions | native xUnit |
|---|---|---|
| `Is.GreaterThan(n)` | `.Should().BeGreaterThan(n)` | `Assert.True(x > n)` |
| `Is.GreaterThanOrEqualTo(n)` | `.Should().BeGreaterThanOrEqualTo(n)` | `Assert.True(x >= n)` |
| `Is.LessThan(n)` / `Is.LessThanOrEqualTo(n)` | `.Should().BeLessThan(n)` / `BeLessThanOrEqualTo(n)` | `Assert.True(x < n)` etc. |
| `Is.InRange(lo, hi)` | `.Should().BeInRange(lo, hi)` | `Assert.InRange(x, lo, hi)` |
| `Is.Positive` / `Is.Negative` / `Is.Zero` | `.Should().BePositive()` / `BeNegative()` / `Be(0)` | `Assert.True(x > 0)` etc. |
| `Is.NaN` | `.Should().Be(double.NaN)` | `Assert.True(double.IsNaN(x))` |

### 3.4 Strings

| NUnit | FluentAssertions | native xUnit |
|---|---|---|
| `Does.Contain("sub")` | `.Should().Contain("sub")` | `Assert.Contains("sub", s)` |
| `Does.Not.Contain("sub")` | `.Should().NotContain("sub")` | `Assert.DoesNotContain("sub", s)` |
| `Does.StartWith("p")` / `Does.EndWith("s")` | `.Should().StartWith("p")` / `EndWith("s")` | `Assert.StartsWith("p", s)` / `Assert.EndsWith("s", s)` |
| `Does.Match("\\d+")` | `.Should().MatchRegex(@"\d+")` | `Assert.Matches(@"\d+", s)` |
| `Is.Empty` (string) | `.Should().BeEmpty()` | `Assert.Empty(s)` (string is `IEnumerable<char>`; better failure message than `Assert.Equal("", s)`) |
| `Is.Null.Or.Empty` | `.Should().BeNullOrEmpty()` | `Assert.True(string.IsNullOrEmpty(s))` |

### 3.5 Collections

> NUnit `Is.EqualTo` on collections compares **element-wise in order**; `Is.EquivalentTo` compares **ignoring order**. Preserve the distinction.

| NUnit | FluentAssertions | native xUnit |
|---|---|---|
| `Assert.That(c, Is.EqualTo(expected))` | `c.Should().Equal(expected)` (ordered) | `Assert.Equal(expected, c)` (element-wise) |
| `Assert.That(c, Is.EquivalentTo(expected))` | `c.Should().BeEquivalentTo(expected)` (unordered) — note FA compares elements *structurally* (member-by-member), not via `Equals`; for types with meaningful custom equality use `o => o.ComparingByValue<T>()` or sort both and use `.Equal(...)` | Sort both, then `Assert.Equal` — or flag |
| `Has.Property("X").EqualTo(y)` | `obj.X.Should().Be(y)` — or reflection-free restatement per property | `Assert.Equal(y, obj.X)` |
| `Has.Count.EqualTo(n)` / `Has.Length.EqualTo(n)` | `.Should().HaveCount(n)` | `Assert.Equal(n, c.Count)` |
| `Is.Empty` / `Is.Not.Empty` | `.Should().BeEmpty()` / `NotBeEmpty()` | `Assert.Empty(c)` / `Assert.NotEmpty(c)` |
| `Has.Member(item)` / `Does.Contain(item)` | `.Should().Contain(item)` | `Assert.Contains(item, c)` |
| `Has.No.Member(item)` | `.Should().NotContain(item)` | `Assert.DoesNotContain(item, c)` |
| `Has.Exactly(1).Items` | `.Should().ContainSingle()` | `Assert.Single(c)` |
| `Has.Some.Matches<T>(pred)` | `.Should().Contain(x => pred(x))` | `Assert.Contains(c, pred)` |
| `Has.All.Matches<T>(pred)` / `Is.All...` | `.Should().OnlyContain(x => pred(x))` | `Assert.All(c, x => Assert.True(pred(x)))` |
| `Is.Unique` | `.Should().OnlyHaveUniqueItems()` | `Assert.Distinct(c)` |
| `Is.Ordered` / `Is.Ordered.Descending` | `.Should().BeInAscendingOrder()` / `BeInDescendingOrder()` | Manual compare with sorted copy |
| `Is.SubsetOf(super)` | `.Should().BeSubsetOf(super)` | `Assert.Subset(superSet, subSet)` (both args must be `ISet<T>`) |
| `Is.SupersetOf(sub)` | `super.Should().Contain(sub)` | `Assert.Superset(subSet, superSet)` (both args must be `ISet<T>`) |

### 3.6 Exceptions

> **Semantic trap:** NUnit `Assert.Throws<T>` / `Throws.TypeOf<T>` are **exact-type**. NUnit `Assert.Catch<T>` / `Throws.InstanceOf<T>` permit **derived types**. FluentAssertions' plain `Throw<T>` permits derived types — mapping `Assert.Throws<T>` to it loosens the test; use `ThrowExactly<T>`.

| NUnit | FluentAssertions | native xUnit |
|---|---|---|
| `Assert.Throws<T>(() => f())` | `f.Should().ThrowExactly<T>()` — as `Action act = () => f(); act.Should()...` | `Assert.Throws<T>(() => f())` (also exact) |
| `Assert.Catch<T>(() => f())` | `act.Should().Throw<T>()` | `Assert.ThrowsAny<T>(() => f())` |
| `Assert.ThrowsAsync<T>(async () => ...)` | `await act.Should().ThrowExactlyAsync<T>()` | `await Assert.ThrowsAsync<T>(...)` |
| `Assert.CatchAsync<T>(...)` | `await act.Should().ThrowAsync<T>()` | `await Assert.ThrowsAnyAsync<T>(...)` |
| `Assert.That(f, Throws.TypeOf<T>())` | `act.Should().ThrowExactly<T>()` | `Assert.Throws<T>(f)` |
| `Assert.That(f, Throws.InstanceOf<T>())` | `act.Should().Throw<T>()` | `Assert.ThrowsAny<T>(f)` |
| `Throws...With.Message.EqualTo("m")` | `.WithMessage("m")` (supports wildcards) | capture: `var ex = Assert.Throws<T>(f); Assert.Equal("m", ex.Message);` |
| `Throws.ArgumentException.With.Property("ParamName")...` | `.WithParameterName("p")` | capture and assert `ex.ParamName` |
| `Assert.DoesNotThrow(() => f())` | `act.Should().NotThrow()` | Call `f()` directly — an unexpected exception already fails the test |
| `Assert.DoesNotThrowAsync(...)` | `await act.Should().NotThrowAsync()` | `await` it directly |

### 3.7 Multiple assertions, skip, warnings

| NUnit | xUnit v3 target |
|---|---|
| `Assert.Multiple(() => { ... })` | FluentAssertions: `using (new AssertionScope()) { ... }` (`FluentAssertions.Execution`). Native xUnit: sequential asserts — note the lost report-all-failures semantics |
| `Assert.Multiple(async () => { ... })` | `using (new AssertionScope()) { ... }` with awaits inside |
| `using (Assert.EnterMultipleScope()) { ... }` (NUnit 4.2+) | Same target as `Assert.Multiple`: FluentAssertions `using (new AssertionScope()) { ... }`; native xUnit — sequential asserts, and note the lost report-all-failures semantics. The `using`-scope spelling is the one NUnit 4.2+ codebases actually use, so grep for both |
| `Assert.Ignore("reason")` (runtime) | `Assert.Skip("reason")` — **never** `[Fact(Skip=...)]`, which is unconditional |
| `Assert.Inconclusive("reason")` | `Assert.Skip("reason")` |
| `Assume.That(condition)` | `Assert.SkipUnless(condition, "reason")` |
| `Assume.That(x, Is.EqualTo(y))` | `Assert.SkipUnless(Equals(x, y), "reason")` — constraint form needs manual predicate |
| `Assert.Pass("msg")` | `return;` — flag if mid-method control flow depended on the thrown `SuccessException` |
| `Assert.Fail("msg")` | `Assert.Fail("msg")` |
| `Assert.Warn("msg")` | `TestContext.Current.AddWarning("msg")` — a real v3 warning on the test result (VSTest RunSettings `FailWarns=true` escalates warnings to failures for NUnit-like visibility) |
| `await Assert.ThatAsync(() => f(), Is.EqualTo(x))` | Await the value first, then apply the normal mapping: `(await f()).Should().Be(x)` / `Assert.Equal(x, await f())` |

### 3.8 Classic helper classes

| NUnit | FluentAssertions / native xUnit |
|---|---|
| `StringAssert.Contains(sub, s)` | `s.Should().Contain(sub)` / `Assert.Contains(sub, s)` |
| `StringAssert.StartsWith` / `EndsWith` / `IsMatch` | `.Should().StartWith` / `EndWith` / `MatchRegex` |
| `CollectionAssert.AreEqual(e, a)` | `a.Should().Equal(e)` / `Assert.Equal(e, a)` |
| `CollectionAssert.AreEquivalent(e, a)` | `a.Should().BeEquivalentTo(e)` |
| `CollectionAssert.Contains` / `DoesNotContain` | `.Should().Contain` / `NotContain` |
| `CollectionAssert.AllItemsAreUnique` | `.Should().OnlyHaveUniqueItems()` |
| `FileAssert.*`, `DirectoryAssert.*` | **Manual** — assert on `File.Exists`/contents explicitly |

## 4. Lifecycle and fixtures

> **Instance lifecycle inverts.** NUnit creates ONE fixture instance for all tests in the class (unless `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]`); xUnit creates one instance PER TEST. Instance fields no longer carry state between tests — usually a latent-bug fix, but audit tests that relied on accumulation.

| NUnit | xUnit v3 |
|---|---|
| `[SetUp]` | Constructor — fields become `readonly`, `= null!` suppressions go away |
| `[TearDown]` | `IDisposable.Dispose()` |
| async `[SetUp]` / `[TearDown]` | `IAsyncLifetime`: `ValueTask InitializeAsync()` + `ValueTask DisposeAsync()` (v3 signature — extends `IAsyncDisposable`; xUnit v2 used `Task`) |
| `[OneTimeSetUp]` / `[OneTimeTearDown]` | `IClassFixture<TFixture>` — move the shared state into `TFixture` (ctor + `Dispose`), inject via test-class constructor |
| `[SetUpFixture]` (namespace scope) | `[assembly: AssemblyFixture(typeof(TFixture))]` (v3) — ctor runs before any test, `Dispose` after all. `[SetUpFixture]` is namespace-scoped: multiple fixtures in different namespaces cannot collapse into one assembly fixture; use collection fixtures per affected group instead |
| Shared fixture across specific classes | `ICollectionFixture<T>` + `[CollectionDefinition]` + `[Collection("name")]` — note this also serializes those classes |
| `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]` | Delete — already xUnit's model |

**`[OneTimeSetUp]` example:**

```csharp
// NUnit
[TestFixture]
public class DbTests
{
    private DbConnection _conn = null!;
    [OneTimeSetUp] public void Init() => _conn = Open();
    [OneTimeTearDown] public void Cleanup() => _conn.Dispose();
}

// xUnit v3
public sealed class DbFixture : IDisposable
{
    public DbConnection Conn { get; } = Open();
    public void Dispose() => Conn.Dispose();
}

public class DbTests : IClassFixture<DbFixture>
{
    private readonly DbFixture _db;
    public DbTests(DbFixture db) => _db = db;
}
```

## 5. Output / TestContext

| NUnit | xUnit v3 |
|---|---|
| `TestContext.WriteLine(...)` / `TestContext.Out.WriteLine` | Constructor-injected `ITestOutputHelper` (`using Xunit;` — the `Xunit.Abstractions` namespace is v2-only) or `TestContext.Current.TestOutputHelper?.WriteLine(...)` |
| `TestContext.Progress.WriteLine(...)` | `ITestOutputHelper.WriteLine` (no live-progress equivalent) |
| `TestContext.CurrentContext.Test.Name` | `TestContext.Current.Test?.TestDisplayName` |
| `TestContext.CurrentContext.Result.Outcome` | `TestContext.Current.TestState` — only available after the test has finished running (`Test`/`TestOutputHelper` are likewise null outside a test) |
| `TestContext.Error.WriteLine(...)` | `ITestOutputHelper.WriteLine` (no separate error stream) |
| `TestContext.CurrentContext.WorkDirectory` / `TestDirectory` | `AppContext.BaseDirectory` or explicit paths |
| `TestContext.CurrentContext.CancellationToken` | `TestContext.Current.CancellationToken` |
| `TestContext.AddTestAttachment(path)` | `TestContext.Current.AddAttachment(name, ...)` |
| `TestContext.CurrentContext.Random` | **Manual** — seeded `Random` in the test; flag nondeterminism |

## 6. Parallelization

> **The default inverts.** NUnit runs everything single-threaded unless `[Parallelizable]` is applied. xUnit runs **test classes in parallel by default** (each class = one collection; methods within a class stay serial). A migration that ignores this turns shared static/db/file/port state into flaky tests.

| NUnit source | xUnit v3 equivalent |
|---|---|
| No `[Parallelizable]` anywhere, tests share mutable state | `[assembly: CollectionBehavior(DisableTestParallelization = true)]`, or group the sharing classes under one `[Collection("...")]` |
| No `[Parallelizable]`, tests are isolated | Accept xUnit's default (usually a speedup); state the decision |
| `[assembly: Parallelizable(ParallelScope.Fixtures)]` | xUnit's default — nothing to add |
| `[Parallelizable(ParallelScope.Children)]` on a fixture (methods in parallel) | No equivalent — xUnit never parallelizes within a class; accept serial methods |
| `[NonParallelizable]` on specific fixtures | Put those classes in a shared `[Collection]` |
| `[assembly: LevelOfParallelism(n)]` | `[assembly: CollectionBehavior(MaxParallelThreads = n)]` — close but not identical: NUnit caps worker threads across tests, xUnit caps parallel *collections* (`0` = processor count, negative = unlimited) |

## 7. Assembly-level attributes

| NUnit | Disposition |
|---|---|
| `[assembly: Parallelizable(...)]` / `[assembly: LevelOfParallelism(n)]` | Replace per Section 6 |
| `[assembly: Category("x")]` | `[assembly: Trait("Category", "x")]` — v3's `TraitAttribute` targets Assembly, Class and Method; there is no `AssemblyTraitAttribute` in v3 (that was a v2 type) |
| `[assembly: Timeout(ms)]` | No assembly-wide timeout — per-test `Timeout` or flag |
| `[assembly: SetCulture("...")]` / `SetUICulture` | Set culture in an `AssemblyFixture` constructor |

## 8. Packages and project file

**Remove** from `.csproj` / `Directory.Build.props` / `Directory.Packages.props`:

- `NUnit`, `NUnit3TestAdapter`, `NUnit.Analyzers`, `NUnit.ConsoleRunner`, `NUnitLite`

**Add** (VSTest runner — the incremental default):

```xml
<PackageReference Include="xunit.v3" Version="3.2.2" />
<PackageReference Include="xunit.runner.visualstudio" Version="3.1.5" PrivateAssets="all" />
<!-- keep the existing Microsoft.NET.Test.Sdk reference; the 3.x adapter builds against
     Microsoft.TestPlatform.ObjectModel 17.13, so bump very old pins (< 17.12) if discovery fails -->
```

Package shape (3.x): `xunit.v3` is the MTP-v1 flavour (`xunit.v3` -> `xunit.v3.mtp-v1` -> core + assert + analyzers); named variants are `xunit.v3.mtp-v1`, `xunit.v3.mtp-v2` (default from 4.0.0), and `xunit.v3.mtp-off`. For a VSTest repo, use plain `xunit.v3` (or `xunit.v3.mtp-off`) and do **not** set `<TestingPlatformDotnetTestSupport>` or a `global.json` test-runner entry — either one reroutes `dotnet test` to MTP. For an MTP repo, pair `xunit.v3.mtp-v1` with `<TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>` (SDK 8/9) or the `global.json` test-runner setting (SDK 10+).

Project-file changes:

- Add `<OutputType>Exe</OutputType>` — xUnit v3 test projects are executables.
- `xunit.v3` brings `xunit.analyzers` transitively; do not add it separately.
- Swap `global using NUnit.Framework;` / `<Using Include="NUnit.Framework" />` for `<Using Include="Xunit" />` (match sibling-project convention).
- Target framework must be .NET Framework 4.7.2+ or .NET 8+; do not change it during migration.
- VSTest loggers (`JunitXml.TestLogger`), coverlet, and `--blame` keep working under the VSTest runner (`--blame` is VSTest-specific; MTP's equivalents are `--crashdump`/`--hangdump`).

## 9. Companion / extension libraries

| NUnit companion | xUnit v3 equivalent |
|---|---|
| `FluentAssertions` (7.x) / `Shouldly` / `AwesomeAssertions` | Keep — framework-agnostic. FluentAssertions needs 7.1.0+ for xunit.v3 failure integration; v8+ requires a commercial licence |
| `Moq` / `NSubstitute` / `FakeItEasy` | Keep — framework-agnostic |
| `AutoFixture.NUnit3` (`[AutoData]`) | `AutoFixture.Xunit3` — check availability on the feed; otherwise flag |
| `FsCheck.NUnit` | `FsCheck.Xunit.v3` |
| `Verify.NUnit` | `Verify.XunitV3` |
| `NUnit.DeepObjectCompare` | FluentAssertions `BeEquivalentTo` |

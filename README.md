# dotnet-test-migrations

A Claude Code plugin marketplace for .NET migration skills.

## Install

In Claude Code:

```
/plugin marketplace add modernizedcode/claude-skills
/plugin install migrate-nunit-to-xunit@dotnet-test-migrations
```

If the install summary says `Run /reload-plugins to activate.`, run that.

## Plugins

### `migrate-nunit-to-xunit`

Converts .NET test projects from NUnit 3.x/4.x to xUnit v3 while preserving the existing
test runner (VSTest or MTP) and target framework. It handles the mechanical rewrites
(`[Test]` → `[Fact]`, `[TestCase]` → `[Theory]`/`[InlineData]`, `[SetUp]` → constructor)
and, more importantly, the semantic ones that a find-and-replace gets wrong:

- **Parallelisation inverts.** NUnit is serial by default; xUnit runs test classes in
  parallel by default. Shared static state, files, databases, and ports need an explicit
  decision.
- **Non-public test classes disappear.** NUnit discovers `internal` fixtures; xUnit does
  not. No failure, no skip — just a lower discovered count.
- **Exact vs. derived exception types.** `Assert.Throws<T>` is exact; `Assert.Catch<T>`
  permits derived. Mapping both to the same target silently loosens tests.
- **`Is.InstanceOf<T>` vs. `Is.TypeOf<T>`** are assignable-from and exact respectively.
- **Runtime skips** (`Assert.Ignore`, `Assume.That`) must not become compile-time
  `[Fact(Skip = ...)]`, which excludes the test permanently.
- **`[DefaultFloatingPointTolerance]`** silently loosens every float comparison in scope
  and has no xUnit equivalent.

Constructs with no xUnit equivalent (`[Retry]`, `[Order]`, `[Repeat]`, `[Apartment]`,
custom attributes) are flagged for a human decision rather than approximated or deleted.

It finishes by comparing discovered / passed / failed / skipped counts against a baseline.

**Not for:** xUnit v2 → v3 upgrades, NUnit version upgrades, migrations to MSTest or
TUnit, or runner-only VSTest → MTP migrations.

## Use it as a repo-local skill instead

If you'd rather not install a plugin, copy the skill into a project:

```
your-project/
  .claude/
    skills/
      migrate-nunit-to-xunit/
        SKILL.md
        references/mapping-cheatsheet.md
```

Or into `~/.claude/skills/` to make it available everywhere.

## Contributing

Found a NUnit construct it maps wrong, or one it doesn't know about? Open an issue with
the source snippet and what the correct xUnit v3 target is.

## Licence

MIT

---
name: dotnet-10
description: Write, review or upgrade C# and .NET code that targets .NET 10 (net10.0) or C# 14, or upgrade a project from .NET 8 or .NET 9. Use when a project targets net10.0, when the user mentions .NET 10, C# 14, extension blocks or extension properties, the field keyword, null-conditional assignment, file-based apps (dotnet run app.cs), .slnx, or asks to modernize or upgrade .NET code.
---

# .NET 10 and C# 14

.NET 10 is the current long-term support (LTS) release, so most production code targets it. Use this skill so new code uses C# 14 and the .NET 10 APIs correctly, and so upgrades from .NET 8 or 9 catch the breaking changes.

**Status (checked 2026-11-10):** .NET 10 was released November 11, 2025 and is supported until November 14, 2028. C# 14 is the default language version for `net10.0`. EF Core 10 is also LTS and needs the .NET 10 runtime. **.NET 8 and .NET 9 reach end of support on November 10, 2026.** Recommend upgrading any project still on them.

For projects on `net11.0`, also use the `dotnet-11` skill. Everything here still applies there.

## Steps

1. Check the target framework in the `.csproj`, `Directory.Build.props`, or a file-based app. Only use the features below for `net10.0` or later. Check that `<LangVersion>` is not pinned lower.
2. Before using a feature, read the matching reference for its exact shape:
   - [references/csharp-14.md](references/csharp-14.md): extension blocks, `field`, null-conditional assignment, span conversions, lambda modifiers, partial constructors and events, compound assignment operators, `nameof(List<>)`.
   - [references/libraries-sdk.md](references/libraries-sdk.md): new library APIs, file-based apps, CLI and container changes.
   - [references/aspnetcore-efcore.md](references/aspnetcore-efcore.md): ASP.NET Core 10 and EF Core 10.
   - [references/upgrading.md](references/upgrading.md): moving from .NET 8 or 9, with breaking changes and the .NET 9 features worth adopting.
3. If something is not covered, check https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview rather than guessing.

## Quick guide: prefer these in net10.0 code

| Instead of | Write |
| --- | --- |
| A private backing field only used by one property | `set => field = value?.Trim() ?? "";` |
| `if (x is not null) x.Prop = value;` | `x?.Prop = value;` |
| `static class` of `this` extension methods, no way to add properties | `extension(IEnumerable<T> source) { public bool IsEmpty => !source.Any(); }` |
| `(string text, out int result) => ...` just to add `out` | `(text, out result) => int.TryParse(text, out result)` |
| `GroupJoin` plus `SelectMany` plus `DefaultIfEmpty` | `LeftJoin(...)` / `RightJoin(...)` (also translated by EF Core 10) |
| A tiny console project for a script | `app.cs` with `#:package` and `dotnet run app.cs` |
| `dotnet add package X` | `dotnet package add X` (old form still works) |
| `.sln` for a new solution | `.slnx` (the new default) |
| A `System.Linq.Async` package reference | built-in `System.Linq.AsyncEnumerable` (remove the package) |
| Hand-rolled SSE in minimal APIs | `TypedResults.ServerSentEvents(stream, eventType: "x")` |
| Separate EF query filters you cannot turn off one at a time | `HasQueryFilter("SoftDelete", ...)` plus `IgnoreQueryFilters(["SoftDelete"])` |

## Rules

- `field` and `extension` are contextual keywords now. If code already has a member or local named `field`, or a type named `extension`, use `@field`/`this.field` or rename it.
- Span conversions can change which overload is chosen. If a call becomes ambiguous or behaves differently (for example in test assertions on arrays), pass `.AsSpan()` or call the intended method explicitly.
- `x?.P = v` evaluates the right side only when `x` is not null. `x?.P++` and `x?.P--` are not allowed.
- Post-quantum crypto types (`MLDsa`, `SlhDsa`, `CompositeMLDsa`) are marked `[Experimental]` (SYSLIB5006). Use them only when asked.
- When upgrading, read [references/upgrading.md](references/upgrading.md) and tell the user which breaking changes affect them. Do not assume a clean build means a safe upgrade: many changes are behavioral.

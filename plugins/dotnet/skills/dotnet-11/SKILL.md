---
name: dotnet-11
description: Write, review or upgrade C# and .NET code that targets .NET 11 (net11.0) or C# 15. Use when a project targets net11.0, when the user mentions .NET 11, C# 15, union types, closed classes, collection expression with(...), labeled break/continue, runtime async, or asks to modernize .NET code. Covers new language features, library APIs, SDK changes and breaking changes.
---

# .NET 11 and C# 15

.NET 11 and C# 15 are newer than most of Claude's training data. Use this skill so new code uses the new features correctly, and so you do not invent APIs that do not exist.

**Status (checked 2026-11-10):** .NET 11 launches at .NET Conf 2026 (November 10 to 13, 2026). Microsoft's docs were last updated for RC 2, which has go-live support. .NET 11 is a Standard Term Support (STS) release. C# 15 is the default language version for `net11.0`. EF Core 11 ships with it and needs the .NET 11 runtime.

## Steps

1. Check the target framework in the `.csproj` (or `#:property TargetFramework` in a file-based app). Only use the features below when it is `net11.0` or later. If it targets an older version, suggest an upgrade but do not use the features.
2. Check whether the project sets `<LangVersion>`. An explicit older value turns C# 15 off.
3. Before using an API from this skill, read the matching reference file for its exact shape:
   - [references/csharp-15.md](references/csharp-15.md): unions, closed hierarchies, `with(...)` in collection expressions, extension indexers, labeled `break`/`continue`, memory safety preview.
   - [references/libraries.md](references/libraries.md): Process, streams, LINQ joins, System.Text.Json, compression, numerics, crypto, validation, logging.
   - [references/runtime-sdk.md](references/runtime-sdk.md): runtime async, hardware baseline, file-based apps, `dotnet run`/`test`/`sln`, containers.
   - [references/aspnetcore-efcore.md](references/aspnetcore-efcore.md): ASP.NET Core 11 and EF Core 11.
   - [references/breaking-changes.md](references/breaking-changes.md): read this when upgrading a project to .NET 11.
4. If a feature you need is not in these files, check https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/overview rather than guessing.

## Quick guide: prefer these in net11.0 code

| Instead of | Write |
| --- | --- |
| OneOf-style libraries, or a base class plus `_ => throw` arm | `public union Result(Success, Error);` and an exhaustive `switch` |
| `abstract` base record used only for a fixed set of cases | `public closed record Shape;` so the `switch` needs no default arm |
| `new List<T>(capacity) { ... }`, `new HashSet<T>(comparer) { ... }` | `[with(capacity: n), .. items]`, `[with(StringComparer.OrdinalIgnoreCase), "a"]` |
| Boolean flags or `goto` to leave nested loops | `outer: for (...) { ... break outer; }` |
| `Process.Start` plus manual stdout reading | `await Process.RunAndCaptureTextAsync("git", ["status"])` |
| Custom `IEqualityComparer<T>` that compares one key | `EqualityComparer<T>.Create(x => x.Key)` |
| `GroupJoin` plus `SelectMany` plus `DefaultIfEmpty` | `LeftJoin`, `RightJoin` (from .NET 10) or `FullJoin` (.NET 11) |
| `new MemoryStream(Encoding.UTF8.GetBytes(s))` | `new StringStream(s, Encoding.UTF8)` |
| `new MemoryStream(memory.ToArray())` | `new ReadOnlyMemoryStream(memory)` |
| Third-party Zstandard package | `ZstandardStream` in `System.IO.Compression` |

## Rules

- Do not use preview-only features unless the user asks. In .NET 11 these are: the C# memory safety rules (`LangVersion` `preview` plus a feature flag), runtime async (`<Features>runtime-async=on</Features>`), and APIs marked experimental (composite ML-KEM, HPKE, `TlsBufferSession`/`TlsSocketSession`, Blazor AI components, DBSC, `AnchorMode`/`ItemComparer` in Blazor `Virtualize`).
- `union` declarations become structs that box value-type cases into `object? Value`. For hot paths with value types, write a custom union with `HasValue` and `TryGetValue` (see the C# reference).
- A default union value holds `null`. In a `switch` over a union, prefer a `null` arm to `_` or `var`, so the compiler still warns when a new case type is added.
- `closed` only restricts derivation to the declaring assembly, and it is not transitive. A `closed` class is implicitly abstract and cannot also be `sealed`, `static` or `abstract`.
- Do not invent APIs. If you are not sure a member exists in .NET 11, say so and point to the docs.
- When you upgrade a project, read [references/breaking-changes.md](references/breaking-changes.md) and tell the user which changes affect them. The most common surprise is the new x86-64-v2 CPU baseline.

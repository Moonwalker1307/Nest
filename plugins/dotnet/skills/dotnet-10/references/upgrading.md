# Upgrading from .NET 8 or .NET 9 to .NET 10

.NET 8 (LTS) and .NET 9 (STS) both reach end of support on November 10, 2026. .NET 10 (LTS) is supported until November 14, 2028. Move to .NET 10 unless the user wants .NET 11 (STS, same end date as .NET 10).

Full lists: https://learn.microsoft.com/dotnet/core/compatibility/10 and https://learn.microsoft.com/dotnet/core/compatibility/9

## Steps

1. Update `global.json` (if present) to a 10.0 SDK. Change `<TargetFramework>` to `net10.0` in every project, or once in `Directory.Build.props`.
2. Update all `Microsoft.*`, `System.*` and EF Core packages to 10.x together.
3. Remove packages that the framework now provides. NuGet prunes them and reports NU1510; `System.Linq.Async` must be removed by hand because `System.Linq.AsyncEnumerable` is now built in.
4. Remove `<LangVersion>` unless it is set on purpose.
5. Build, run the tests, then check the lists below and tell the user which items apply. Many are behavioral and do not show up as build errors.
6. Suggest the new features worth adopting (end of this file), but keep the upgrade itself small.

## Coming from .NET 8: what changed in .NET 9

- **`BinaryFormatter` is removed.** It always throws. Code or packages that use it need another serializer.
- C# 13 arrived: `params` collections (`params ReadOnlySpan<T>`), the `System.Threading.Lock` type (`lock` on a `Lock` uses its fast path), `\e` escape, `ref`/`unsafe` in iterators and async methods, `allows ref struct`, partial properties, `OverloadResolutionPriorityAttribute`.
- Useful .NET 9 APIs: `CountBy`, `AggregateBy`, `Index()` in LINQ; `Task.WhenEach`; `OrderedDictionary<TKey,TValue>`; `Guid.CreateVersion7()`; `Base64Url`; `SearchValues<string>`; `HybridCache` (`Microsoft.Extensions.Caching.Hybrid`); `MapStaticAssets()` in ASP.NET Core; built-in `Microsoft.AspNetCore.OpenApi` document generation (Swashbuckle is no longer in templates).
- Check the .NET 9 breaking changes page too when skipping a version.

## .NET 10 breaking changes that hit most often

### SDK and NuGet

- A `PackageReference` without a version is now an error (NU1015).
- Framework-provided packages are pruned; direct references to them give NU1510. Remove them, or set `RestoreEnablePackagePruning` to `false`.
- `dotnet restore` audits transitive packages. New vulnerability warnings can fail builds that treat warnings as errors. HTTP audit sources are rejected.
- `dotnet new sln` makes `.slnx`. Tools that only read `.sln` need an update.
- CLI and `dotnet watch` write non-output messages to stderr. Scripts that parse stdout may change.
- `dotnet tool install --local` creates a tool manifest if none exists.

### Language (C# 14)

- `field` and `extension` are contextual keywords. Rename or escape (`@field`, `this.field`, `@extension`).
- Span overloads are chosen more often; some calls become ambiguous. Use `.AsSpan()` or explicit calls.
- See [csharp-14.md](csharp-14.md) for the full list.

### Runtime and libraries

- **The runtime no longer installs default SIGTERM handlers.** The generic host still handles shutdown. Console apps without the host that relied on graceful SIGTERM must use `PosixSignalRegistration`.
- `BackgroundService.ExecuteAsync` runs entirely on a background task, including the synchronous part before the first `await`. Startup ordering can change.
- Configuration keeps keys with `null` values.
- Default trace context propagator is W3C.
- System.Text.Json throws on property name conflicts.
- `MailAddress` rejects consecutive dots.
- HTTP/3 is off by default with `PublishTrimmed`.
- OpenSSL 1.1.1 or later is needed on Linux. The ICU override variable is `DOTNET_ICU_VERSION_OVERRIDE`.
- Generic math shift operators behave consistently across types.

### Containers

- Default images use Ubuntu instead of Debian. Check `apt` package names and paths in Dockerfiles.

### ASP.NET Core

- OpenAPI moved to 3.1 and `Microsoft.OpenApi` 2.0. Document and schema transformers need code changes.
- Cookie auth returns 401/403 for API endpoints instead of redirecting to the login page.
- Blazor: `<NotFound>` fragment unsupported (use `NotFoundPage`), WebAssembly `HttpClient` streaming on by default, standalone WebAssembly environment set by `<WasmApplicationEnvironmentName>`, `BlazorCacheBootResources` removed.
- `UseExceptionHandler` no longer logs exceptions that an `IExceptionHandler` handled.

### EF Core

- `Contains` over collections uses separate parameters, not `OPENJSON`. Usually faster; check query plans for very large lists.
- Migrations no longer run in one shared transaction.
- On Azure SQL or compatibility level 170, JSON columns move from `nvarchar(max)` to `json` in the next migration.
- Check the EF Core 10 breaking changes page, especially for SQLite date handling and Cosmos.

### Desktop

- Apps using both WPF and WinForms must alias `MenuItem` and `ContextMenu`.
- WPF rejects empty `ColumnDefinitions`/`RowDefinitions` and crashes on wrong `DynamicResource` use.

## Worth adopting after the upgrade

- C# 14 `field` properties and `x?.P = v`.
- Extension properties for helpers that were awkward as methods.
- `LeftJoin`/`RightJoin` in LINQ and EF Core.
- `AddValidation()` in minimal APIs.
- Named query filters and `ExecuteUpdateAsync` with normal lambdas in EF Core.
- `JsonSerializerOptions.Strict` for APIs that should reject unknown or duplicate fields.
- File-based apps for scripts and tools.

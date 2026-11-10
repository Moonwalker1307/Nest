# Upgrading to .NET 11: breaking changes to check

Full list: https://learn.microsoft.com/dotnet/core/compatibility/11. C# compiler changes: https://learn.microsoft.com/dotnet/csharp/whats-new/breaking-changes/compiler%20breaking%20changes%20-%20dotnet%2011

## Upgrade steps

1. Change `<TargetFramework>` to `net11.0` and update `global.json` to an 11.0 SDK if the repo pins one.
2. Update `Microsoft.*`, `System.*` and EF Core packages to 11.x together.
3. Remove `<LangVersion>` unless it pins something on purpose. C# 15 is the default.
4. Build, then go through the list below and tell the user which items apply.

## Runtime and deployment

- **CPU baseline:** x86/x64 now needs x86-64-v2. Check old servers, VMs with restricted CPU flags, and emulators. Windows Arm64 needs LSE.
- `DOTNET_RuntimeAsync` and `UNSUPPORTED_RuntimeAsync` environment variables are gone.
- iOS and tvOS: `FileStream` no longer takes advisory locks, so `FileShare` is not enforced. Restore with `System.IO.DisableFileLocking=false` or `DOTNET_SYSTEM_IO_DISABLEFILELOCKING=0`.

## Libraries

- `ZipArchive` checks CRC32 when reading. Corrupt or truncated entries that used to read fine now throw `InvalidDataException`.
- `DeflateStream` and `GZipStream` write a header and footer even for empty payloads, so output bytes for empty input change.
- `double` or `float` to `decimal` and `BigInteger` conversions are now correctly rounded, with no switch to go back. `(decimal)1.23` (a double literal) is no longer exactly `1.23m`. Use `m` literals for decimals.
- `IStartupValidator` is obsolete. Implement `IAsyncStartupValidator`.
- `MemoryCache` metric attribute names changed.
- System.Text.Json: a type-level `JsonIgnoreCondition.Always` is invalid (SYSLIB1226).

## SDK

- `#:property` names in file-based apps cannot contain `:`.
- The `Configuration` environment variable now sets the default build configuration. A CI machine with `Configuration=Release` set builds Release by default.
- The NativeAOT CLI and MSBuild server are on by default. Opt out with `DOTNET_CLI_ENABLEAOT=false` and `DOTNET_CLI_USE_MSBUILD_SERVER=false`.
- "Using launch settings from..." moved to stderr. Scripts that parse stdout may change.
- `dotnet format` folder mode reads only ancestor `.editorconfig` files. Move global settings to `.globalconfig`.

## ASP.NET Core

- OpenAPI documents default to 3.2, and `Microsoft.OpenApi` 3.3.1 has API breaks. Pin `OpenApi3_1` if clients cannot read 3.2.
- QuickGrid sort and page use links in the URL. CSS that targets `button.col-title` must also match `a.col-title`.
- `Virtualize` default `OverscanCount` is 15, not 3.
- Blazor SSR antiforgery validation moved to middleware.
- Endpoint filters run on binding failures. `ThrowOnBadRequest` behaves differently in Development.
- Standalone Blazor WASM: replace the DevServer package with `Microsoft.AspNetCore.Components.Gateway` and remove `inspectUri`.

## EF Core

- SQL Server default compatibility level is 160. Apps on SQL Server 2019 or older must set `UseCompatibilityLevel`.
- Cosmos: `__jObject` removed, unmapped JSON properties are dropped on save, undefined projections throw.
- Full list: https://learn.microsoft.com/ef/core/what-is-new/ef-core-11.0/breaking-changes

# .NET 11 runtime and SDK

Sources: https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/runtime and https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/sdk

## Runtime async (preview)

The runtime, not the compiler, handles `await` suspension and resumption. The compiler stops generating state-machine classes. Results: real call chains in `new StackTrace()`, profilers and debuggers, less overhead, and working breakpoints across `await`. Exception stack traces look the same either way. It works with Native AOT and ReadyToRun. The .NET runtime libraries themselves are already compiled with it.

Turn it on per project only when the user wants it:

```xml
<PropertyGroup>
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

- `net11.0` projects no longer need `<EnablePreviewFeatures>true</EnablePreviewFeatures>` for it.
- Opt out with `<UseRuntimeAsync>false</UseRuntimeAsync>`.
- The `DOTNET_RuntimeAsync` and `UNSUPPORTED_RuntimeAsync` environment variables were removed. Do not suggest them.
- Your source code does not change. Keep writing normal `async`/`await`.

## Hardware baseline (can break deployments)

- x86/x64 needs **x86-64-v2** (adds SSE4.2, POPCNT, SSSE3, CX16) on every OS. Older CPUs fail at start with "The current CPU is missing one or more of the baseline instruction sets."
- ReadyToRun code targets x86-64-v3 (AVX2 and friends) on Windows and Linux. Older v2 CPUs still run, with more JIT work at start-up.
- Windows Arm64 needs armv8.0-a + LSE. Linux Arm64 minimum is unchanged (Raspberry Pi still works).

## Other runtime changes

- JIT: more bounds-check elimination, devirtualization, switch folding, Arm SVE2 intrinsics. `Math.BigMul` on x64 is a single `MUL`.
- Native AOT interface dispatch is faster and smaller.
- In-process crash reports capture the managed stack on Linux, macOS and mobile.
- CoreCLR on WebAssembly is in progress.

## File-based apps (`dotnet run app.cs`)

```csharp
#:include helpers.cs
#:include models/customer.cs
#:include ./libs/MyLibrary.dll                 // .dll becomes a Reference
#:property Company="Contoso Tools"             // quoted values allowed
#:package System.CommandLine@2.0.1 Aliases=CommandLine   // Name=Value metadata

extern alias CommandLine;
```

- `#:package`, `#:project` and `#:ref` accept trailing `Name=Value` metadata.
- `dotnet reference add --file app.cs <project-path>` adds a project reference.
- `dotnet format` works on file-based apps. Native AOT output is reused between builds.
- Breaking: `:` is no longer allowed in `#:property` names.

## CLI

```bash
dotnet run -e ASPNETCORE_ENVIRONMENT=Development -e LOG_LEVEL=Debug

dotnet new slnf --parent-solution MySolution.slnx --name Tests
dotnet sln Tests.slnf add tests/Tests/Tests.csproj
dotnet sln Tests.slnf list
```

- MSBuild properties expand in launch-profile `commandLineArgs`.
- `dotnet watch` integrates with Aspire app hosts, recovers from crashes, and can pick a MAUI or mobile device.
- The `Configuration` environment variable now sets the default for `-c`.
- The NativeAOT CLI host and the MSBuild server are on by default. Opt out with `DOTNET_CLI_ENABLEAOT=false` and `DOTNET_CLI_USE_MSBUILD_SERVER=false`.
- CLI telemetry uses OpenTelemetry. `DOTNET_CLI_TELEMETRY_OPTOUT` still works.
- `dotnet nuget add source` / `update source` take `--min-publish-age-hours`.

## dotnet test (Microsoft.Testing.Platform)

Run-level options go before `--`.

```bash
dotnet test --timeout 90s                        # exit code 3 on timeout
dotnet test --maximum-failed-tests 5             # exit code 13 when hit
dotnet test --results-directory-layout per-module
dotnet test dirs.proj                            # traversal projects
```

- Also: `--no-dependencies`, `--use-current-runtime`/`--ucr`, `--device`, `--test-modules` exclusions, JSON output for `--list-tests`, live display of running tests.
- `DOTNET_TEST_RUNNER=VSTest` or `Microsoft.Testing.Platform`.
- Test projects for Android, iOS, macOS and Mac Catalyst (`androidtest`, `iostest`, `macostest`, `maccatalysttest` templates).
- `dotnet new` test templates support xUnit v3 (on Microsoft.Testing.Platform by default) and NUnit through `--test-runner`.

## Containers

```bash
dotnet publish /t:PublishContainer \
  -p:ContainerRegistry=registry.example.com \
  -p:SOURCE_DATE_EPOCH="$(git log -1 --pretty=%ct)"   # reproducible digests
```

- Multi-arch builds work with Podman.
- Local runtime order: `wslc` on Windows, `container` on macOS, then Docker, then Podman. Set `LocalRegistry` to force one.
- Pushes skip layers when the manifest already exists. `ContainerPushNoCache=true` turns that off.

## Other SDK changes

- `dotnet format` in folder mode only reads `.editorconfig` files in ancestor folders of included files. Use `.globalconfig` for settings that must apply everywhere.
- "Using launch settings from..." now goes to stderr.
- Razor Class Libraries can emit TypeScript into Static Web Assets.

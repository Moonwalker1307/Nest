# .NET 10 libraries, runtime and SDK

Sources: https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/libraries, https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/sdk and https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/runtime

## LINQ

- `LeftJoin` and `RightJoin` on `Enumerable` and `Queryable`. C# query syntax does not support them yet.

  ```csharp
  var rows = students.LeftJoin(departments,
      s => s.DepartmentId, d => d.Id,
      (s, d) => new { s.Name, Department = d?.Name ?? "[NONE]" });
  ```

- `System.Linq.AsyncEnumerable` is part of the core libraries: LINQ over `IAsyncEnumerable<T>` without the `System.Linq.Async` package. Remove that package when upgrading, or names will clash.

## System.Text.Json

```csharp
var options = JsonSerializerOptions.Strict;   // disallow unmapped members and duplicates,
                                              // respect nullable annotations and required ctor params

var lenient = new JsonSerializerOptions { AllowDuplicateProperties = false }; // throw on duplicate keys

var person = await JsonSerializer.DeserializeAsync<Person>(pipe.Reader);     // PipeReader overloads

[JsonSourceGenerationOptions(ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
partial class MyContext : JsonSerializerContext { }
```

`JsonDocumentOptions.AllowDuplicateProperties` does the same for `JsonDocument.Parse`.

## Collections, strings, globalization

```csharp
// OrderedDictionary<TKey, TValue> (added in .NET 9) now returns the index.
if (!dict.TryAdd(key, 1, out int index))
{
    int value = dict.GetAt(index).Value;
    dict.SetAt(index, value + 1);
}

// Compare digits as numbers: "file2" < "file10", "02" == "2".
var comparer = StringComparer.Create(CultureInfo.CurrentCulture, CompareOptions.NumericOrdering);
```

- `ISOWeek.GetWeekOfYear(DateOnly)`, `ISOWeek.GetYear(DateOnly)`, `ISOWeek.ToDateOnly(year, week, day)`.
- `TimeSpan.FromMilliseconds(long)` single-argument overload (works in expression trees).
- Span-based normalization: `GetNormalizedLength`, `IsNormalized`, `TryNormalize` on `ReadOnlySpan<char>`.
- UTF-8 hex: `Convert.FromHexString(ReadOnlySpan<byte>)`, `Convert.TryToHexString`, `Convert.TryToHexStringLower`.

## Compression

Async ZIP APIs: `ZipFile.CreateFromDirectoryAsync`, `ZipFile.ExtractToDirectoryAsync`, `ZipFile.OpenAsync`, `ZipFile.OpenReadAsync`, `ZipArchive.CreateAsync`, `ZipArchiveEntry.OpenAsync`, `CreateEntryFromFileAsync`, `ExtractToFileAsync`. Update mode no longer loads every entry into memory.

## Networking

```csharp
// A Stream over a WebSocket.
using Stream stream = WebSocketStream.Create(webSocket, WebSocketMessageType.Binary, ownsWebSocket: true);

// One message as a stream; the end-of-message is sent on Dispose.
using (Stream message = WebSocketStream.CreateWritableMessageStream(webSocket, WebSocketMessageType.Text))
{
    await JsonSerializer.SerializeAsync(message, payload);
}
```

TLS 1.3 for macOS clients is opt-in with `AppContext.SetSwitch("System.Net.Security.UseNetworkFramework", true)`.

## Cryptography

- Post-quantum: `MLKem`, `MLDsa`, `SlhDsa` (FIPS 203, 204, 205), plus `CompositeMLDsa`. Check the static `IsSupported` first. `MLDsa`, `SlhDsa` and `CompositeMLDsa` are `[Experimental]` (SYSLIB5006).

  ```csharp
  using MLKem key = MLKem.GenerateKey(MLKemAlgorithm.MLKem768);
  ```

- `X509Certificate2Collection.FindByThumbprint(HashAlgorithmName.SHA256, thumbprint)`.
- `PemEncoding.FindUtf8(bytes)`.
- `ExportPkcs12` with `Pkcs12ExportPbeParameters.Pbes2Aes256Sha256`.
- AES key wrap with padding (RFC 5649): `aes.EncryptKeyWrapPadded(...)`, `aes.DecryptKeyWrapPadded(...)`.

## Diagnostics and process

- `ActivitySource(new ActivitySourceOptions(name) { TelemetrySchemaUrl = ... })`, `Meter.TelemetrySchemaUrl`.
- Default trace context propagator is W3C.
- Windows: `ProcessStartInfo.CreateNewProcessGroup = true` to send isolated signals such as Ctrl+Break.
- `System.Numerics.Tensors` is stable, not experimental.

## Runtime

JIT improvements (better inlining, devirtualization of array interface methods, stack allocation of small objects and arrays that don't escape, struct argument codegen), AVX10.2 support, and smaller and faster Native AOT output. No code changes are needed.

## File-based apps

A single `.cs` file runs without a project:

```csharp
#!/usr/bin/env dotnet
#:package Humanizer@2.14.1
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false
#:project ../ClassLib/ClassLib.csproj

Console.WriteLine("Hello".Humanize());
```

```bash
dotnet run app.cs
dotnet publish app.cs              # Native AOT by default
dotnet project convert app.cs      # turn it into a normal project
chmod +x app.cs && ./app.cs        # with the shebang line, on Unix
```

.NET 11 adds `#:include` and more; see the `dotnet-11` skill.

## CLI

- Noun-first commands: `dotnet package add|list|remove`, `dotnet reference add|list|remove`. The old verb-first forms still work.
- `dotnet tool exec <tool>` and `dnx <tool>` run a tool once without installing it.
- `dotnet new sln` creates `.slnx`. Convert an old one with `dotnet sln migrate`.
- `dotnet test` can use Microsoft.Testing.Platform natively, configured in `global.json`:

  ```json
  { "test": { "runner": "Microsoft.Testing.Platform" } }
  ```

- `--cli-schema` prints a command's arguments and options as JSON.
- `--interactive` is on by default in an interactive terminal.
- Tab completion: `dotnet completions script <shell>`.
- Tools can target specific platforms with the `any` RuntimeIdentifier.

## Containers

- `dotnet publish /t:PublishContainer` works for console apps without `<EnableSdkContainerSupport>`.
- `<ContainerImageFormat>OCI</ContainerImageFormat>` (or `Docker`).
- Default .NET images are based on Ubuntu, not Debian.

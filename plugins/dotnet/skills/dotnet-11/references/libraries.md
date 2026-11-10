# .NET 11 library APIs

Source: https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/libraries. Code below is taken from Microsoft's examples. Where only a name is given, Microsoft did not publish a sample; check the API docs for the exact overloads before using it.

## Contents

- Process
- Streams
- Text and regular expressions
- System.Text.Json
- Compression and archives
- LINQ
- Collections, numerics, random, SIMD
- Networking, security and cryptography
- Reflection
- Configuration, options, validation, logging, caching, tracing

## Process

```csharp
// Run, wait, capture stdout and stderr, get the exit code.
ProcessTextOutput result = await Process.RunAndCaptureTextAsync(
    "git", ["status", "--porcelain"]);
Console.WriteLine(result.StandardOutput);
Console.WriteLine($"Exit code: {result.ExitStatus.ExitCode}");

// Start suspended (Windows and macOS), set things up, then resume.
var startInfo = new ProcessStartInfo("worker.exe") { StartSuspended = true };
using var process = Process.Start(startInfo)!;
process.SafeHandle.Resume();

// Send a POSIX signal and wait for the exit status.
if (OperatingSystem.IsLinux() || OperatingSystem.IsMacOS())
{
    using Process p = Process.Start("sleep", "30")!;
    p.Signal(PosixSignal.SIGTERM);
    ProcessExitStatus status = await p.WaitForExitStatusAsync();
}
```

Also new:

- `Process.StartAndForget(...)`: start a child you will not wait for. The handle is released for you.
- `ProcessStartInfo.StartDetached`: detach from the parent's session or console so the child outlives the terminal.
- `ProcessStartInfo.KillOnParentExit`: Windows only.
- `Process.TryGetProcessById(int, out Process)`: returns `false` instead of throwing.

## Streams

Wrap in-memory data as a `Stream` without copying:

```csharp
using Stream config = new StringStream(yamlText, Encoding.UTF8);

ReadOnlyMemory<byte> payload = GetPayload();
using Stream body = new ReadOnlyMemoryStream(payload);
```

- `ReadOnlySequenceStream`: wraps a `ReadOnlySequence<byte>` (for example from System.IO.Pipelines).
- `WritableMemoryStream`: wraps a writable `Memory<byte>` as a fixed-size stream.
- `SafeFileHandle.CreateAnonymousPipe(out readEnd, out writeEnd, asyncRead: true, asyncWrite: false)` and `SafeFileHandle.Type`.

## Text and regular expressions

```csharp
StringBuilder dest = StringBuilder.MoveChunks(source); // source is left empty, no copy

int bad = Utf8.IndexOfInvalidSubsequence(bytes);
bool ok = Utf16.IsValid(chars);

// ^, $ and . treat every Unicode newline as a line end, not only \n.
Regex.Matches(text, @"^line\d$", RegexOptions.Multiline | RegexOptions.AnyNewLine);

// Hex float round trip.
string hex = Math.PI.ToString("X");                    // "0X1.921FB54442D18P+1"
double back = double.Parse(hex, NumberStyles.HexFloat);
```

- Base64: `EncodeToChars`, `EncodeToString`, `EncodeToUtf8`, `DecodeFromChars`, `DecodeFromUtf8`, with array/string-returning overloads and span overloads.
- `Console` honors the `FORCE_COLOR` environment variable.
- `CborReader` can read incrementally: pass `isFinalBlock: false`, check for `CborReaderState.NeedsMoreData`, then call `SlideData(nextChunk, isFinalBlock)`. Needs `CborConformanceMode.Lax`.

## System.Text.Json

C# unions serialize as their contents:

```csharp
public union Reading(int, string);

string json = JsonSerializer.Serialize<Reading>(new("hello")); // "hello"
```

When two cases have the same JSON shape (both objects), pick a classifier:

```csharp
[JsonUnion(TypeClassifier = typeof(JsonUnionTypeStructuralClassifier))]
public union PetUnion(Dog, Cat);
```

F# discriminated unions are supported through reflection only (no source generation, trimming or Native AOT).

Closed hierarchies can infer derived types instead of listing `[JsonDerivedType]`:

```csharp
[JsonPolymorphic(InferClosedTypePolymorphism = true)]
closed record Shape;
record Circle(double Radius) : Shape;
record Square(double Length) : Shape;
```

Open generic derived types work: `[JsonDerivedType(typeof(Derived<>), "derived")]` on `Base<T>`. Open generic converters work: `[JsonConverter(typeof(OptionConverter<>))]`.

JSON Lines output:

```csharp
await JsonSerializer.SerializeAsyncEnumerable(pipeWriter, items, topLevelValues: true);
```

Other additions:

- `options.GetTypeInfo<T>()` and `options.TryGetTypeInfo<T>(out JsonTypeInfo<T>?)`, no cast needed.
- `JsonNamingPolicy.PascalCase`. Naming policy and ignore conditions can now be set on a whole type, and naming policy on a single member. A type-level `JsonIgnoreCondition.Always` is invalid (SYSLIB1226). Check the System.Text.Json docs for the attribute names before using these.
- `Utf8JsonWriter.Reset(stream, options)` can change options when reusing a writer.
- Built-in numeric converters and collection contracts.

## Compression and archives

- `ZstandardStream` and `ZstandardEncoder` in `System.IO.Compression`. No extra package.
- Span-based `ZLibEncoder` (and Deflate/GZip equivalents) with `Compress(source, dest, out consumed, out written, isFinalBlock)` returning `OperationStatus`, plus `Reset()`.
- ZIP: `ZipArchiveEntry.Open(FileAccess)` / `OpenAsync(FileAccess, CancellationToken)`, `ZipArchiveEntry.CompressionMethod`, password overloads for encrypted entries. **CRC32 is now checked when reading; corrupt entries throw `InvalidDataException`.**
- Tar: choose the format with `TarFile.CreateFromDirectory(src, dest, includeBaseDirectory: true, format: TarEntryFormat.Gnu)` (also `Ustar`, `Pax`, async overloads). GNU sparse format 1.0 can be read.

## LINQ

On `Enumerable`, `Queryable` and `AsyncEnumerable`:

```csharp
// FullJoin: every row from both sides; the missing side is default.
foreach (var (product, category) in products.FullJoin(categories, p => p.Category, c => c.Name)) { }

// Join and GroupJoin overloads with no result selector return tuples.
foreach (var (product, order) in products.Join(orders, p => p.Id, o => o.ProductId)) { }
```

`LeftJoin` and `RightJoin` came in .NET 10 and also get tuple overloads. EF Core 11 translates `FullJoin` to `FULL JOIN`.

## Collections, numerics, random, SIMD

```csharp
var byName = EqualityComparer<(string Name, int Age)>.Create(p => p.Name);

var bits = new BitArray(stackalloc byte[] { 0b_0000_0011, 0b_1000_0000 }); // also ReadOnlySpan<bool>/<int>

int i = Random.Shared.NextInteger<int>();
long l = Random.Shared.NextInteger<long>(0L, 100L);
Half h = Random.Shared.NextBinaryFloat<Half>();

Vector128<int> powers = Vector128.CreateGeometricSequence(1, 2); // {1, 2, 4, 8}
(Vector128<int> lo, Vector128<int> hi) = Vector128.Zip(a, b);    // also Unzip, Concat
```

- IEEE 754 decimal floating point: `Decimal32`, `Decimal64`, `Decimal128` in `System.Numerics`.
- Generic `Complex<T>` (works with `float`, `Half`, `BFloat16` and the decimal types).
- `INumberBase<TSelf>.TryParsePartial`: parse a number and report how many characters were used. Good for CSV-style input without substrings.
- `Half` uses hardware FP16 instructions on x64 and Arm64.

## Networking, security and cryptography

- Typed DNS records through `Dns` and the new `DnsResolver` class (Windows, Linux, macOS).
- HTTP request-body compression wrappers, configurable connection eviction, `SocketsHttpHandler.InitialHttp2MaxConcurrentStreams`, `QuicStream.Priority` for HTTP/3.
- `X25519DiffieHellman` (abstract), with `X25519DiffieHellmanCng` on Windows and `X25519DiffieHellmanOpenSsl` elsewhere.
- Unpadded AES Key Wrap: `EncryptKeyWrap` / `DecryptKeyWrap` on `Aes`.
- ML-KEM recipients in `EnvelopedCms`.
- **Experimental:** composite ML-KEM, HPKE, caller-driven TLS (`TlsBufferSession`, `TlsSocketSession`).

## Reflection

- `Type.GetNullableUnderlyingType()`: `typeof(int?)` gives `Int32`, non-nullable types give `null`.
- `MetadataLoadContext.GetLoadContext(assembly)`.

## Configuration, options, validation, logging, caching, tracing

```csharp
// Skip a property when binding configuration.
[ConfigurationIgnore]
public string ComputedKey => Endpoint + ":default";

// Async DataAnnotations validation.
public sealed class ValidVatNumberAttribute : AsyncValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext context) =>
        throw new InvalidOperationException("Validate this attribute with IsValidAsync.");

    protected override async Task<ValidationResult?> IsValidAsync(
        object? value, ValidationContext context, CancellationToken cancellationToken)
    { /* ... */ }
}
await Validator.ValidateObjectAsync(model, context, validateAllProperties: true);

// Async options validation.
public sealed class BackendOptionsValidator : IAsyncValidateOptions<BackendOptions> { /* ValidateAsync + Validate */ }

// Generic source-generated logging methods.
[LoggerMessage(0, LogLevel.Trace, "Received {Code} ({Length} bytes)")]
public static partial void PacketReceived<TCode>(ILogger logger, TCode code, int length)
    where TCode : struct, Enum;

// Rule-based activity tracing.
builder.Services.AddTracing(tracing =>
{
    tracing.EnableTracing(sourceName: "MyCompany.Orders");
    tracing.DisableTracing(sourceName: "MyCompany.Orders", operationName: "HealthCheck");
});
```

- `IAsyncValidatableObject` for async object-level validation.
- `ValidationAttribute.FormatMessage(format, name)`.
- `IStartupValidator` is obsolete; implement `IAsyncStartupValidator`.
- `MemoryCache` with `TrackStatistics = true` publishes OpenTelemetry metrics.
- `MediaTypeNames.Video` constants.

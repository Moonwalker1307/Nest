# ASP.NET Core 10 and EF Core 10

Sources: https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-10.0 and https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew. Breaking changes: https://learn.microsoft.com/aspnet/core/breaking-changes/10/overview and https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/breaking-changes

## ASP.NET Core 10

### Minimal APIs

```csharp
builder.Services.AddValidation();   // DataAnnotations on query, header and body parameters; 400 on failure

app.MapPost("/products", ([Range(1, 1000)] int productId, [Required] string name) => TypedResults.Ok(productId));
app.MapPost("/raw", (Payload p) => TypedResults.Ok()).DisableValidation();

app.MapGet("/heartrate", (CancellationToken ct) =>
    TypedResults.ServerSentEvents(GetHeartRate(ct), eventType: "heartRate"));
```

Validation also works on nested types and records when they are marked for validation (see the docs for `[ValidatableType]`).

### OpenAPI

- Documents are OpenAPI 3.1 by default and use `Microsoft.OpenApi` 2.0. Transformers must change: `OpenApiAny` is gone (use `JsonNode`, for example `["temperatureC"] = 0`) and `OpenApiSchema.Nullable` is gone. This applies even if you pick 3.0.
- Choose the version with `builder.Services.AddOpenApi(o => o.OpenApiVersion = OpenApiSpecVersion.OpenApi3_0);`.
- XML doc comments flow into the document.

### Blazor

- `[PersistentState]` on a component property keeps its value from prerendering into interactive rendering:

  ```razor
  @code {
      [PersistentState]
      public List<Movie>? Movies { get; set; }

      protected override async Task OnInitializedAsync() => Movies ??= await MovieService.GetMoviesAsync();
  }
  ```

- Not-found pages: `NavigationManager.NotFound()` and `<Router NotFoundPage="typeof(Pages.NotFound)">`. The `<NotFound>` render fragment is no longer supported. Without the Blazor router, use `app.UseStatusCodePagesWithReExecute("/not-found", createScopeForStatusCodePages: true)`.
- WebAssembly: boot resources are preloaded, `blazor.boot.json` is inlined into `dotnet.js`, `HttpClient` response streaming is on by default (`ReadAsStreamAsync` returns a stream without sync reads).
- Standalone WebAssembly environment comes from `<WasmApplicationEnvironmentName>`, not the `Blazor-Environment` header or `launchSettings.json`.
- Same-page `NavigateTo` no longer scrolls to the top.

### Identity and auth

- Passkeys (WebAuthn/FIDO2) in ASP.NET Core Identity. The Blazor Web App template includes them.
- Cookie auth returns 401/403 instead of redirecting for API endpoints (`[ApiController]`, JSON minimal APIs, `TypedResults` endpoints, SignalR), detected through `IApiEndpointMetadata`. Override `OnRedirectToLogin`/`OnRedirectToAccessDenied` to keep redirects.

### Other

- Kestrel's memory pool evicts unused memory automatically.
- `UseExceptionHandler` no longer logs diagnostics for exceptions an `IExceptionHandler` handled. Set `SuppressDiagnosticsCallback = context => false` to restore.

## EF Core 10

LTS, supported until November 2028. Needs the .NET 10 runtime.

### Queries

```csharp
// LEFT JOIN / RIGHT JOIN.
var rows = context.Students.LeftJoin(context.Departments,
    s => s.DepartmentId, d => d.Id,
    (s, d) => new { s.FirstName, Department = d.Name ?? "[NONE]" });

// ExecuteUpdateAsync with a normal lambda, so you can use if statements.
await context.Blogs.ExecuteUpdateAsync(s =>
{
    s.SetProperty(b => b.Views, 8);
    if (nameChanged) s.SetProperty(b => b.Name, "foo");
});

// Named query filters that can be turned off one at a time.
modelBuilder.Entity<Blog>()
    .HasQueryFilter("SoftDelete", b => !b.IsDeleted)
    .HasQueryFilter("Tenant", b => b.TenantId == tenantId);

var all = await context.Blogs.IgnoreQueryFilters(["SoftDelete"]).ToListAsync();
```

- Collections in `Contains` are sent as separate padded parameters (`IN (@ids1, @ids2, ...)`) by default, not one JSON parameter. Change with `UseParameterizedCollectionMode(...)`, or per query with `EF.Constant(ids)`.
- Split queries add the key to `ORDER BY` so results are consistent.
- SQL parameter names are simpler (`@city`, not `@__city_0`).
- An analyzer warns about string concatenation in `FromSqlRaw`.

### Complex types and JSON

```csharp
modelBuilder.Entity<Customer>(b =>
{
    b.ComplexProperty(c => c.ShippingAddress);                   // table splitting
    b.ComplexProperty(c => c.Details, d => d.ToJson());          // JSON column, collections allowed
});

await context.Blogs.ExecuteUpdateAsync(s =>
    s.SetProperty(b => b.Details.Views, b => b.Details.Views + 1));   // bulk update inside JSON
```

- Complex types can be optional (`Address?`, needs at least one required property) and can be structs.
- Prefer complex types over owned entities for JSON and table splitting: they have value semantics.

### SQL Server and Azure SQL

- `SqlVector<float>` with `[Column(TypeName = "vector(1536)")]` and `EF.Functions.VectorDistance("cosine", b.Embedding, vector)`.
- The `json` data type is used with `UseAzureSql` or compatibility level 170. Existing `nvarchar` JSON columns change to `json` in the next migration unless you pin the column type.
- Named default constraints: `HasDefaultValueSql("GETDATE()", "DF_Post_Created")` or `UseNamedDefaultConstraints()`.

### Cosmos DB

Full-text search (`EF.Functions.FullTextContains`, `FullTextScore`), hybrid search with `EF.Functions.Rrf`, vector search no longer experimental (`IsVectorProperty`, `IsVectorIndex`).

### Behavior changes

- Migrations are no longer wrapped in one transaction (the EF 9 behavior was reverted).
- Inlined constants are logged as `?` unless `EnableSensitiveDataLogging` is on.
- Microsoft.Data.Sqlite changed how `DateTime`, `DateTimeOffset` and UTC values are handled.

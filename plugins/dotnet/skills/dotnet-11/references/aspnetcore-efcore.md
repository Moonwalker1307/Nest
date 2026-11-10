# ASP.NET Core 11 and EF Core 11

Sources: https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-11 and https://learn.microsoft.com/ef/core/what-is-new/ef-core-11.0/whatsnew. Migration guide: https://learn.microsoft.com/aspnet/core/migration/100-to-110

## ASP.NET Core 11

### Minimal APIs and validation

- C# unions work in JSON request and response bodies, SignalR's JSON protocol, JS interop and persistent state. They do not work for route, query, header or form binding.
- Async validation: derive from `AsyncValidationAttribute` and override `IsValidAsync`, or implement `IAsyncValidatableObject.ValidateAsync` (returns `IAsyncEnumerable<ValidationResult>`). Register with `builder.Services.AddValidation();`.
- Validation messages are localized when you call `AddLocalization()` and `AddValidation()`.
- Endpoint filters now run on binding failures.
- `[ShortCircuit]` or `[ShortCircuit(404)]` on controllers and endpoints.
- `ValidatableTypeAttribute` and `SkipValidationAttribute` are no longer experimental. Remove `ASP0029` suppressions.

### OpenAPI

- Documents are **OpenAPI 3.2 by default**. Older tools may need `options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_1`. `Microsoft.OpenApi` is now 3.3.1, which has its own breaking changes.
- `MapMethods("/search", ["QUERY"], ...)` documents HTTP QUERY.
- File results map to `type: string, format: binary`.
- `TypedResults.ServerSentEvents(...)` emits `itemSchema` in 3.2.
- `[Obsolete]` sets `deprecated: true`.
- `Microsoft.AspNetCore.Grpc.Swagger` is deprecated.

### Blazor

- `<DisplayName For="@(() => Model!.ProductionDate)" />` reads `[Display]` or `[DisplayName]`.
- `<BasePath />` renders `<base href>`.
- `NavigateTo(url, new NavigationOptions { RelativeToCurrentUri = true })`, also on `NavLink`. `NavigationManager.GetUriWithFragment`.
- Static SSR: `[SupplyParameterFromTempData]`, `[CascadingParameter] ITempData`, `[SupplyParameterFromSession]` (needs `AddSession()` and `UseSession()`), `<CacheView VaryByQuery="..." ExpiresAfter="...">`, client-side validation on by default with `DataAnnotationsValidator`.
- `Virtualize`: variable item heights, `InitialItemIndex`, `ScrollToItemAsync(index)`. Default `OverscanCount` changed from 3 to 15. `AnchorMode` is experimental (`ASP0030`).
- `QuickGrid`: sort and page state in the URL by default (rendered as links), `Sortable="true"` on `PropertyColumn`, `OnRowClick`. Use unique `QueryParameterNameOptions` when several grids share a page.
- `app.MapRazorComponents<App>().WithBrowserOptions(o => ...)` configures client start-up from C#.
- `<EnvironmentView Include="Development">`.
- Circuits: `Circuit.RequestCircuitPauseAsync`, auto pause on hidden tabs (`Microsoft.AspNetCore.Components.Server.AutoPause`), auth refresh without reconnecting.
- `IHostedService` runs in Blazor WebAssembly. WASM configuration reads environment variables.
- Standalone WASM uses `Microsoft.AspNetCore.Components.Gateway` instead of the WebAssembly DevServer; remove `inspectUri` from `launchSettings.json`.
- The Blazor Web App template no longer calls `app.UseAntiforgery()`; protection is automatic.
- Async form validation: `EditContext.RegisterAsyncFieldValidator`, `IsValidationPending(field)`.
- Experimental: Blazor AI components (`Microsoft.AspNetCore.Components.AI`).

### SignalR, auth and other

- SignalR auth refresh on server (`EnableAuthenticationRefresh`, hub `OnAuthenticationRefreshedAsync`) and clients (`WithAuthenticationRefresh`). Cancelling the token passed to `InvokeAsync` cancels the hub call.
- Identity uses `TimeProvider`. Passkey names come from the AAGUID.
- `IAuthorizationRequirementData` now works on hubs, MVC controllers and `AuthorizeView`.
- `dotnet user-jwts create --file app.cs` for file-based apps.
- `dotnet dev-certs https --trust` trusts in both WSL and Windows.
- Native OpenTelemetry HTTP server attributes from the `Microsoft.AspNetCore` source.
- Experimental: Device Bound Session Credentials (`AddDeviceBoundSession`).

## EF Core 11

Needs the .NET 11 runtime. It does not run on .NET 10 or .NET Framework.

### Queries

```csharp
var results = await context.Customers
    .FullJoin(context.Orders, c => c.Id, o => o.CustomerId,
              (c, o) => new { Customer = c, Order = o })
    .ToListAsync();

var top = await context.Blogs.MaxByAsync(b => b.Posts.Count());   // MinByAsync too

var hasField = await context.Blogs
    .Where(b => EF.Functions.JsonPathExists(b.JsonData, "$.OptionalInt"))
    .ToListAsync();
```

- Better SQL for split queries and `GroupBy`. Redundant `CAST`s are removed.
- An analyzer flags `ToAsyncEnumerable` on EF queries; use `AsAsyncEnumerable`.

### Modeling

```csharp
modelBuilder.Entity<Animal>().UseTptMappingStrategy()
    .ComplexProperty(a => a.Details, b => b.ToJson());            // complex/JSON on TPT and TPC

modelBuilder.Entity<MyEntity>().Property(e => e.Details.Description).HasMaxLength(500);

modelBuilder.Entity<Customer>().HasKey(c => c.CustomerId.Value);  // keys on complex properties
modelBuilder.Entity<Customer>().HasIndex(c => c.Address.PostalCode);
modelBuilder.Entity<Order>().HasIndex("Items[].Sku");             // JSON collection path
```

### SQL Server

- **Default compatibility level is now 160 (SQL Server 2022).** Set `o.UseCompatibilityLevel(...)` for older servers.
- Vector search (SQL Server 2025, EF API may change): `HasVectorIndex(b => b.Embedding, "cosine")`, `.VectorSearch(b => b.Embedding, embedding, "cosine").OrderBy(r => r.Distance).Take(5).WithApproximate()`. `SqlVector<T>` columns are not loaded unless projected.
- Full-text: `HasFullTextCatalog`, `HasFullTextIndex(...).UseKeyIndex(...).UseCatalog(...)`, `ContainsTable(...)` / `FreeTextTable(...)` returning `Rank`.
- With compatibility level 170, `Contains` on JSON primitive collections uses `JSON_CONTAINS`. `EF.Functions.JsonContains(...)`, JSON indexes.
- Temporal period columns can map to CLR properties: `HasPeriodStart(e => e.PeriodStart)`.
- `EF.Functions.DateTrunc()`, `DateTimeOffset.ToOffset()`, and more date translations.

### SQLite and Cosmos DB

- SQLite: ordered `string.Join` aggregation (SQLite 3.44+). `UInt128` parameters (reading not supported yet).
- Cosmos: complex types, indexing policy (`HasIndex`, vector and full-text indexes), transactional batches (`AutoTransactionBehavior`), `BulkExecutionEnabled()`, session tokens. The serializer moved to System.Text.Json. Breaking: the `__jObject` shadow property is gone, unmapped JSON properties are not kept, undefined projections throw.

### Migrations and tools

- `dotnet ef database update <Name> --add` creates and applies a migration in one step.
- `.ExcludeForeignKeyFromMigrations()` keeps the relationship but not the database constraint.
- The model snapshot records the latest migration ID, so parallel branches conflict visibly.
- `.config/dotnet-ef.json` stores default options. `--context "*"` targets every context.
- The EF CLI writes logs to stderr; stdout carries only output.

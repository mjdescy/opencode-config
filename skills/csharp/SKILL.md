---
name: csharp
description: Use when working with C# code, .cs files, .csproj, .slnx, .sln, or .NET projects. Covers conventions, patterns, preferred Nuget packages, and tooling for C# development.
---

# C# Conventions

## Code Style & Naming

| Symbol Kind | Style | Example |
|---|---|---|
| Class, struct, enum, namespace | PascalCase | `OrderProcessor` |
| Interface | PascalCase with `I` prefix | `IOrderProcessor` |
| Private field | camelCase with `_` prefix | `_orderRepository` |
| Parameter, local variable | camelCase | `orderCount` |
| Method | PascalCase | `ProcessOrder()` |
| Constant | PascalCase | `MaxRetryCount` |
| Boolean member/variable | Prefix `Is`, `Has`, `Can` | `IsActive` |

- Use `var` when the type is obvious from the RHS; explicit types otherwise
- Prefer expression-bodied members (`=>`) for single-expression methods/properties; block body `{ }` for multi-statement logic
- Prefer `init` properties over `set`; mark fields `readonly` when assigned only in constructor
- Prefer `readonly struct` or `record` for value-type data

## Project Structure

- `dotnet 10+` solution with a root `.slnx` file; layout: `src/`, `tests/`
- One type per file; file name matches type name
- File-scoped namespaces (`namespace MyApp;`); folder structure mirrors namespace
- Separate class library for domain logic, referenced from UI/console/web projects
- Use project references, not file linking or shared projects
- Organize by feature folders (e.g. `Features/Orders/`) rather than by layer
- Member order within a file: constants → static fields → instance fields → constructors → properties → public methods → private/internal methods → nested types

## Design Patterns & Principles

- Constructor injection for DI; prefer `Microsoft.Extensions.DependencyInjection`
- Use `IAsyncEnumerable<T>` for streaming async results
- Prefer `System.Text.Json` over Newtonsoft for new code
- Use primary constructors (C# 12+) when appropriate:

  ```csharp
  public class OrderService(IOrderRepository repo)
  {
      public Order Get(int id) => repo.Get(id);
  }
  ```

- Prefer pattern matching over `if`-`is`-cast chains:

  ```csharp
  string result = obj switch
  {
      string s => $"Text: {s}",
      int i => $"Number: {i}",
      _ => "Unknown"
  };
  ```

- Use `record` types for immutable data carriers
- Use null-conditional (`?.`, `??`, `??=`) and collection expressions (`int[] nums = [1, 2, 3];`)
- When a private method operates primarily on a parameter of another type, write it as a **private static extension method** — improves discoverability and enables LINQ-style chaining
- Keep methods small and focused (0–3 parameters; use a parameter object beyond that)
- Avoid `out`/`ref` parameters; prefer tuples or result objects
- Avoid `#region`, magic numbers/strings, and "Utility"/"Helper" class names
- Comments explain **why**, not **what**; XML doc comments on public/internal APIs only

## Testing

- xUnit test framework; NSubstitute or Moq for mocking
- Arrange-Act-Assert pattern; one assertion per test when practical
- Unit tests for all public methods and critical private methods

## Tooling

| Command | Purpose |
|---|---|
| `dotnet format` | Code formatting |
| `dotnet analyzers` | Code quality enforcement |
| `dotnet watch` | Rapid dev feedback |
| `dotnet test` | Run tests |
| `dotnet build` | Build solution |
| `dotnet run` | Run application |

## Common Libraries

| Library | Use Case |
|---|---|
| `CommandLineParser` | CLI argument parsing |
| `System.Text.Json` | JSON serialization |
| `Microsoft.Extensions.DependencyInjection` | DI container |
| `Serilog` | Logging |
| `FluentValidation` | Input validation |
| `ClosedXML` | Excel (.xlsx) manipulation |
| `DuckDB.NET` | Embedded analytical database |

## Console App API Design

- Use subcommands for different operations (`myapp generate`, `myapp review`)
- Limit positional arguments to one (e.g. input file); use options (`--input`, `--output`) for everything else
- Provide clear help text, consistent option naming, and meaningful exit codes
- Always include `--json` for machine-readable output
- Prefer `init` subcommand for configuration/project setup
- Support both file input and stdin

## Security & Performance

### Security Checklist
- **SQL Injection**: Always parameterize SQL queries. Never concatenate user input into SQL strings.
- **Hardcoded secrets**: Never store API keys, connection strings, or passwords in source code. Use `User Secrets`, environment variables, or a secrets manager.
- **Input validation**: Validate all external input. Use `FluentValidation` or `System.ComponentModel.DataAnnotations`.
- **XSS/ Injection**: Encode output for the target context (HTML, JSON, SQL). Use `System.Web.HttpUtility` or Razor's default encoding.
- **Dependency scanning**: Run `dotnet list package --vulnerable` to check for known vulnerabilities.
- **Secure defaults**: Opt in to security features explicitly (e.g., require HTTPS, anti-forgery tokens).

### Performance Checklist
- **Boxing**: Avoid boxing value types. Use generic collections (`List<int>` not `ArrayList`). Prefer `IEquatable<T>` on structs.
- **Allocations**: Prefer `ArrayPool<T>` for temporary large arrays. Use `StringBuilder` for string concatenation in loops. Use `record struct` for small, short-lived value types.
- **Async**: Never use `async void` (except for event handlers). Always `await` or `Task.WhenAll` async calls. Use `ConfigureAwait(false)` in library code. Use `ValueTask<T>` for hot-path async methods that often complete synchronously.
- **LINQ**: Be aware of multiple enumeration. Use `.ToList()` or `.ToArray()` to materialize if iterating multiple times. Prefer `Any()` over `Count() > 0`.
- **DbContext**: Use short-lived DbContext instances. Avoid tracking too many entities. Use `AsNoTracking()` for read-only queries. Use `ExecuteUpdate`/`ExecuteDelete` for bulk operations (EF Core 7+).
- **HttpClient**: Use `IHttpClientFactory` and typed clients. Never wrap `HttpClient` in a `using` block.

## Data Pipeline Patterns

### DuckDB.NET
```csharp
using DuckDB.NET.Data;

await using var conn = new DuckDBConnection("DataSource=:memory:");
await conn.OpenAsync();

await using var cmd = conn.CreateCommand();
cmd.CommandText = "SELECT count(*) FROM read_csv_auto('data.csv')";
var count = (long)(await cmd.ExecuteScalarAsync()!);
```

- Prefer DuckDB over SQLite for analytical/OLAP workloads
- Use `read_csv_auto`, `read_parquet`, `read_json_auto` for direct file queries
- Parameterize all query strings with `DuckDBParameter`
- Use `COPY` for bulk data loading
- Use `CREATE OR REPLACE TABLE ... AS SELECT` for ETL patterns

### ClosedXML (Excel)
```csharp
using ClosedXML.Excel;

using var workbook = new XLWorkbook();
var ws = workbook.Worksheets.Add("Sheet1");
ws.Cell("A1").Value = "Hello";
ws.Cell("B1").Value = 42;
ws.RangeUsed()?.SetAutoFilter();
workbook.SaveAs("output.xlsx");
```

- Use ClosedXML's fluent API where possible
- Handle empty cells with `.IsEmpty()` checks
- Use `TryGetValue<T>()` for type-coercion-safe reads
- Prefer `XLWorkbook` in a `using` statement to ensure proper disposal
- For large files, stream rows with `IXLRangeRows` rather than loading all cells at once

### System.Text.Json
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = false
};
var json = JsonSerializer.Serialize(data, options);
```

- Prefer `System.Text.Json` over Newtonsoft for new code
- Use `[JsonPropertyName("name")]` for property mapping
- Use `JsonSerializerOptions` with `PropertyNamingPolicy = JsonNamingPolicy.CamelCase`
- Use `Utf8JsonWriter` for high-performance streaming write

### Async Data Access
- Use `await using` for `IDisposable` resources in async context
- Use `IAsyncEnumerable<T>` for streaming large result sets from databases
- Use `await foreach` to consume `IAsyncEnumerable<T>` sequences
- Prefer `Npgsql` for PostgreSQL, `Microsoft.Data.SqlClient` for SQL Server
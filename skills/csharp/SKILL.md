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
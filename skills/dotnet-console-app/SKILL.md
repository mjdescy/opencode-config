---
name: dotnet-console-app
description: Use when scaffolding a new .NET 10 console app with a library, unit tests, MIT license, single-file release builds, and CLI verbs. Derived from the report-populator project conventions.
---

# .NET 10 Console App Blueprint

Use these conventions when scaffolding a new .NET 10 solution from scratch. This blueprint encodes the structure, build settings, package choices, and testing patterns proven in the `report-populator` project.

## Solution Structure

```
<repo>/
├── <name>.slnx
├── .gitignore                   (generated via `dotnet new gitignore`)
├── LICENSE                      (MIT)
├── README.md
├── src/
│   ├── <Name>.Console/          (console app, thin — CLI parsing only)
│   │   ├── <Name>.Console.csproj
│   │   └── Program.cs
│   └── <Name>.Library/          (class library — all business logic)
│       ├── <Name>.Library.csproj
│       └── ... .cs files
└── tests/
    ├── <Name>.Library.Tests/    (xUnit — unit + integration tests)
    │   ├── <Name>.Library.Tests.csproj
    │   └── ... .cs files
    └── <Name>.Console.Tests/    (xUnit — CLI argument parsing tests)
        ├── <Name>.Console.Tests.csproj
        └── ... .cs files
```

### Principles
- Console project is **thin**: only CLI parsing. All real work lives in the library.
- Console project references the library via `<ProjectReference>`.
- Test projects reference the project they test via `<ProjectReference>`.
- One type per file; file name matches type name.
- File-scoped namespaces throughout.

## Scaffolding Commands (run in order)

```sh
dotnet new sln -n <name>
dotnet new gitignore

dotnet new classlib -n <Name>.Library -o src/<Name>.Library --framework net10.0
dotnet new console  -n <Name>.Console -o src/<Name>.Console --framework net10.0
dotnet new xunit    -n <Name>.Library.Tests -o tests/<Name>.Library.Tests --framework net10.0
dotnet new xunit    -n <Name>.Console.Tests -o tests/<Name>.Console.Tests --framework net10.0

dotnet sln <name>.slnx add src/<Name>.Library src/<Name>.Console tests/<Name>.Library.Tests tests/<Name>.Console.Tests
```

Remove default `Class1.cs` and `UnitTest1.cs` after scaffolding.

## Project Files — Required Properties

### Console `.csproj`

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net10.0</TargetFramework>
  <ImplicitUsings>enable</ImplicitUsings>
  <Nullable>enable</Nullable>
  <AssemblyName><exe-name></AssemblyName>
  <Version>0.1.0</Version>
</PropertyGroup>

<PropertyGroup Condition="'$(Configuration)' == 'Release'">
  <PublishSingleFile>true</PublishSingleFile>
  <SelfContained>true</SelfContained>
  <DebugType>none</DebugType>
</PropertyGroup>
```

### Library `.csproj`

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <ImplicitUsings>enable</ImplicitUsings>
  <Nullable>enable</Nullable>
  <Version>0.1.0</Version>
</PropertyGroup>

<PropertyGroup Condition="'$(Configuration)' == 'Release'">
  <DebugType>none</DebugType>
</PropertyGroup>
```

### Test `.csproj`

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <ImplicitUsings>enable</ImplicitUsings>
  <Nullable>enable</Nullable>
  <IsPackable>false</IsPackable>
</PropertyGroup>
```

## NuGet Packages

| Project | Package | Version |
|---------|---------|---------|
| Console | `CommandLineParser` | 2.9.x |
| Library | `ClosedXML` | 0.104.x |
| Tests (both) | `coverlet.collector` | 6.0.x |
| Tests (both) | `Microsoft.NET.Test.Sdk` | 17.x |
| Tests (both) | `xunit` | 2.9.x |
| Tests (both) | `xunit.runner.visualstudio` | 3.x |
| Tests (both) | `<Using Include="Xunit" />` | |

## CLI Design

- Use `CommandLineParser` with verbs (`[Verb("name")]`).
- `init` verb for generating sample/template files.
- One positional `[Value(0)]` argument per verb; use options for additional args.
- `Main` returns `int` for exit codes (`0` = success, non-zero = failure).
- Print errors to `stderr`, normal output to `stdout`.

```csharp
public static class Program
{
    public static int Main(string[] args)
    {
        return Parser.Default.ParseArguments<RunOptions, InitOptions>(args)
            .MapResult(
                (RunOptions opts) => RunCommand(opts),
                (InitOptions opts) => InitCommand(opts),
                _ => 1);
    }
}
```

## Data Types

- Use `record` for immutable data carriers (e.g. config records).
- Use a thin config class wrapping `List<T>` with `{ get; init; }`.
- Prefer primary constructors on records:

```csharp
public sealed record Mapping(
    string SourcePath,
    string DestinationPath,
    string Worksheet,
    string CellAddress
);
```

## Testing Conventions

- **Framework**: xUnit. No mocking library needed for self-contained E2E tests.
- **Pattern**: AAA (Arrange-Act-Assert).
- **Temp directories**: Use `Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString())` + `try`/`finally` for cleanup.
- **Integration tests**: Create real files (`.xlsx` via ClosedXML) in temp dirs, exercise the library directly, assert on output.
- **CLI tests**: Call `Program.Main(["verb", "args"])` and assert exit codes and side effects.
- **Collection expressions**: Use `["a", "b"]` not `new[] { "a", "b" }`.
- **Empty input tests**: Assert both no exception *and* that output files/worksheet exist.

## Config File Conventions

- **Serialization**: `System.Text.Json` with `PropertyNamingPolicy = JsonNamingPolicy.CamelCase`.
- **Format**: Root is a bare JSON array `[ ... ]`, not an object with a named property. This simplifies generation from tabular data (e.g. Nushell `$table | to json`).
- **Relative paths**: Resolve against the config file's directory, not CWD. Absolute paths pass through unchanged.

```json
[
  {
    "sourceFilePath": "data/source.xlsx",
    "destinationFilePath": "output/dest.xlsx",
    "destinationWorksheet": "Sheet1",
    "destinationCellAddress": "A4"
  }
]
```

## Build & Publish

```sh
dotnet publish src/<Name>.Console/<Name>.Console.csproj -c Release
```

Output is a single self-contained executable with no `.pdb` files.

## Commit Strategy

- `git init` after scaffolding.
- Commit after each major step: scaffold, packages, library impl, console impl, tests, license, README.
- Commit messages are imperative and concise.

## Example `AGENTS.md`

Copy this into a new repo's `AGENTS.md` after scaffolding:

```markdown
---
name: Default Agent
---

## Instructions

- Use `dotnet build` after every file change to verify compilation.
- Run `dotnet test` after implementing features or adding tests.
- Write unit tests for all public methods.
- Add XML doc comments (`<summary>`) on all public types and methods.
- Use file-scoped namespaces and collection expressions.
- Private fields use `_camelCase` naming.
```

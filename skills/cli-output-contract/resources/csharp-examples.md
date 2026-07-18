# C# Implementation (System.CommandLine)

Use `System.CommandLine` (the official .NET library) to implement the output
contract. It has first-class support for `--help`, `--version`, exit codes,
and is the recommended parser for new .NET CLIs.

> **Note:** The `dotnet-console-app` skill uses `CommandLineParser` (API-compatible name).
> That library works identically — the API shown below applies to both.

## Global Options

Define `--quiet`, `--json`, and `--no-color` as global options on the root
command so every subcommand inherits them:

```csharp
using System.CommandLine;

public static class CliOptions
{
    public static readonly Option<bool> QuietOption = new(
        aliases: ["--quiet", "-q"],
        description: "Suppress all non-error output.");

    public static readonly Option<bool> JsonOption = new(
        aliases: ["--json"],
        description: "Output as structured JSON.");

    public static readonly Option<bool> NoColorOption = new(
        aliases: ["--no-color"],
        description: "Disable colored output.");
}
```

## Output Helper

Encapsulate the three-mode decision in a single class so every command doesn't
duplicate the logic:

```csharp
public sealed class OutputWriter
{
    private readonly bool _quiet;
    private readonly bool _json;
    private readonly TextWriter _stdout;
    private readonly TextWriter _stderr;
    private readonly JsonSerializerOptions _jsonOptions;

    public OutputWriter(
        bool quiet,
        bool json,
        TextWriter? stdout = null,
        TextWriter? stderr = null)
    {
        _quiet = quiet;
        _json = json;
        _stdout = stdout ?? Console.Out;
        _stderr = stderr ?? Console.Error;
        _jsonOptions = new JsonSerializerOptions
        {
            WriteIndented = true,
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        };
    }

    /// <summary>Write a line of human-readable text (default mode only).</summary>
    public void Text(string line)
    {
        if (!_quiet && !_json)
            _stdout.WriteLine(line);
    }

    /// <summary>Write a JSON result to stdout (JSON mode only).</summary>
    public void JsonResult<T>(T payload)
    {
        if (_json)
        {
            var json = JsonSerializer.Serialize(payload, _jsonOptions);
            _stdout.WriteLine(json);
        }
    }

    /// <summary>Write a warning to stderr (all modes).</summary>
    public void Warn(string message)
    {
        _stderr.WriteLine($"warning: {message}");
    }

    /// <summary>Write an error to stderr (all modes).</summary>
    public void Error(string message)
    {
        _stderr.WriteLine($"error: {message}");
    }
}
```

## Wiring It Up

```csharp
using System.CommandLine;

class Program
{
    static async Task<int> Main(string[] args)
    {
        var root = new RootCommand("My CLI tool");

        // Add global options
        root.AddGlobalOption(CliOptions.QuietOption);
        root.AddGlobalOption(CliOptions.JsonOption);
        root.AddGlobalOption(CliOptions.NoColorOption);

        // Register subcommands / handlers
        root.SetHandler((quiet, json, noColor) =>
        {
            var output = new OutputWriter(quiet, json);

            // Respect NO_COLOR
            if (noColor || Environment.GetEnvironmentVariable("NO_COLOR") is { Length: > 0 })
                DisableColor();

            output.Text("Processing...");

            if (json)
            {
                output.JsonResult(new { status = "ok", data = new[] { 1, 2, 3 } });
            }

            return 0;
        },
        CliOptions.QuietOption,
        CliOptions.JsonOption,
        CliOptions.NoColorOption);

        return await root.InvokeAsync(args);
    }

    static void DisableColor()
    {
        Console.ForegroundColor = ConsoleColor.Gray; // no-op equivalent
    }
}
```

## Recommended Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `System.CommandLine` | 2.0.x (latest) | Argument parsing, help, `--version` |
| `System.Text.Json` | (built-in) | JSON serialization |

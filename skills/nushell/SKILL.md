---
name: nushell
description: Use when writing Nushell scripts (.nu files). Covers conventions, structure, and style for Nushell scripting.
---

# Nushell Conventions

## Script Structure

- Every script has a `main` command that serves as the entry point
- Break logic into several small, focused commands (functions)
- Each command does one thing. Keep them short (5-15 lines).
- Define commands in dependency order — called commands before callers, or use `export def` / `def --env` as needed
- Use `export def` when the command is intended for reuse by other scripts, `def` for internal helpers

```nushell
def main [] {
    let data = load-data
    let processed = process-data $data
    save-results $processed
}

def load-data [] -> list {
    open input.json | from json
}

def process-data [data: list] -> list {
    $data | each { |row|
        { name: ($row.name | str title-case), value: ($row.value | into int) }
    }
}

def save-results [data: list] {
    $data | to json | save --force output.json
}
```

## Naming

| Kind | Convention | Example |
|------|-----------|---------|
| Commands | `kebab-case` | `load-data`, `process-input` |
| Variables | `snake_case` | `$input_file`, `$max_retries` |
| CLI flags | `kebab-case` | `--dry-run`, `--input-path` |
| Parameter variables | `snake_case` | `data: list`, `$dry_run`, `$input_path` |
| Environment variables | `SCREAMING_SNAKE_CASE` | `$env.MY_APP_HOME` |
| File names | `kebab-case.nu` | `build-pipeline.nu` |

## Style

- Use pipes (`|`) for data flow, avoid deeply nested `do` blocks
- Prefix unused variables with `_` (e.g., `|_row|`)
- Use `let` over `mut` — avoid mutation unless performance requires it
- Use `$env` sparingly; prefer explicit parameters
- Use `--flags` for optional configuration, positional args for required inputs
- Return data through the pipeline, not `print`
- Use `error make` for error handling, not `exit 1`
- Prefer structured data (records, lists, tables) over string parsing
- Use `each`, `where`, `select`, `group-by`, `sort-by` over raw loops
- Use `try`/`catch` for known-fallible operations
- Fail early with guard clauses to keep the happy path unindented: validate inputs at the top, then proceed without nesting
- Always prefix external commands with `^` (e.g., `^rg`, `^git`, `^ls`) to avoid ambiguity with Nushell internal commands

```nushell
def process-file [path: path, --dry-run] {
    let content = open $path | from json
    if $dry_run {  # Nushell converts --dry-run to $dry_run internally
        print $"Would process ($content | length) rows"
        return
    }
    $content | each { |row|
        { id: $row.id, status: "processed" }
    }
}
```

## Data Types & Conversions

- `string`, `int`, `float`, `bool`, `datetime`, `filesize`, `glob`, `path`
- `record` (key-value), `list` (ordered), `table` (list of records with same keys)
- Explicit conversions: `$x | into int`, `$x | into string`, `$x | into datetime`
- Use `describe` to inspect types during debugging

## Error Handling

```nushell
def safe-read [path: path] {
    let result = try { open $path } catch { |e|
        error make {
            msg: $"Failed to read ($path): ($e.msg)"
            label: { text: $"Could not open file" }
        }
    }
    $result
}
```

## CLI Scripts

- Parse CLI args via `def main [...rest: string]` and manual parsing or use `nu ...`
- Return exit code 0 on success, non-zero on failure

```nushell
def main [--input-path: path, --output-path: path] {
    if ($input_path | is-empty) or ($output_path | is-empty) {
        error make { msg: "Both --input-path and --output-path are required." }
    }
    run-pipeline $input_path $output_path
}
```

## Testing

- Use `nu --test` (Nushell 0.98+) — it discovers all commands annotated with `#[test]` and runs them, reporting pass/fail per test
- Name test commands `test_<description>` for discoverability
- Group tests at the bottom of the file, mirroring the commands they test
- Cover happy path and edge cases (empty input, missing file, invalid data) in separate tests

### Assertions

| Command | Purpose |
|---------|---------|
| `assert` | Assert a condition is true |
| `assert equal` | Assert two values are equal |
| `assert not equal` | Assert two values differ |
| `assert greater` | Assert left > right |
| `assert greater or equal` | Assert left >= right |
| `assert less` | Assert left < right |
| `assert less or equal` | Assert left <= right |
| `assert length` | Assert a list/table has N elements |
| `assert str contains` | Assert a string contains a substring |
| `assert error` | Assert a block raises an error |

### Example

```nushell
#[test]
def test_process_data_transforms_rows [] {
    let input = [{ name: "alice", value: "42" }]
    let actual = process-data $input
    assert equal $actual [{ name: "Alice", value: 42 }]
}

#[test]
def test_process_data_handles_empty_list [] {
    let actual = process-data []
    assert equal $actual []
}

#[test]
def test_load_data_errors_on_missing_file [] {
    assert error { load-data "nonexistent.json" }
}
```

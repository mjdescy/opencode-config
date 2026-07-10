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
- Use `rm --force` over `rm`

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

- Use `nu --test` or inline test commands with `#[test]` attribute (Nushell 0.98+)
- Write small helper commands that return data for easy assertion

## Tooling

| Command | Purpose |
|---------|---------|
| `nu -c "..."` | Run inline script |
| `nu script.nu` | Run a script file |
| `nu --commands "..."` | Run commands and exit |
| `nu --stdin` | Read input from stdin |
| `nu --test` | Run tests in script |

# Bash Implementation (getopt / argh)

For shell scripts, use `getopt` (POSIX) or `argbash` (code generator) to handle
flag parsing. Since Bash lacks structured types, implement the output contract
with careful printf discipline.

## Pattern with `getopt`

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- Argument parsing ---
QUIET=false
JSON=false
NO_COLOR=false

OPTS=$(getopt -o qh --long quiet,json,no-color,help -n "$0" -- "$@")
if [ $? -ne 0 ]; then exit 1; fi
eval set -- "$OPTS"

while true; do
    case "$1" in
        -q|--quiet)    QUIET=true; shift ;;
        --json)        JSON=true; shift ;;
        --no-color)    NO_COLOR=true; shift ;;
        -h|--help)     echo "Usage: $0 [--quiet|-q] [--json] [--no-color]"; exit 0 ;;
        --)            shift; break ;;
        *)             echo "Internal error!" >&2; exit 1 ;;
    esac
done

# Respect NO_COLOR
if [ "$NO_COLOR" = true ] || [ -n "${NO_COLOR:-}" ]; then
    NO_COLOR=true
fi

# --- Output helpers ---
write_text() {
    if [ "$QUIET" = false ] && [ "$JSON" = false ]; then
        printf '%s\n' "$1"
    fi
}

write_json() {
    if [ "$JSON" = true ]; then
        printf '%s\n' "$1"
    fi
}

write_warning() {
    printf 'warning: %s\n' "$1" >&2
}

write_error() {
    printf 'error: %s\n' "$1" >&2
}

# --- Main logic ---
write_text "Processing..."

if [ "$JSON" = true ]; then
    write_json '{"status":"ok","data":[1,2,3]}'
fi

exit 0
```

## Pattern with `argbash` (recommended for complex CLIs)

Use [argbash](https://argbash.dev/) to generate the parsing boilerplate:

```bash
# my-cli.m4 — input for argbash
# ARG_OPTIONAL_BOOLEAN([quiet],[q],[Suppress non-error output])
# ARG_OPTIONAL_BOOLEAN([json],[],[Output as structured JSON])
# ARG_OPTIONAL_BOOLEAN([no-color],[],[Disable colored output])
# ARG_HELP([My CLI tool])
# ARGBASH_GO
```

Then generate: `argbash my-cli.m4 -o my-cli`

## Key Points for Bash CLIs

| Requirement | How to satisfy |
|---|---|
| `--quiet` / `-q` | Boolean flag, gate `echo`/`printf` calls |
| `--json` | Boolean flag, `echo '{...}'` — use `jq` to build complex payloads |
| Errors to stderr | `>&2` redirection on every error message |
| Exit codes | `exit 0` / `exit 1` |
| `--help` / `-h` | `getopt` handles this, or just `printf` usage and exit 0 |
| `NO_COLOR` | Check `-n "${NO_COLOR:-}"` |

## Building JSON with `jq`

For non‑trivial JSON output, pipe through `jq`:

```bash
if [ "$JSON" = true ]; then
    jq -n '{status: "ok", data: [1,2,3]}'
fi
```

This automatically handles quoting and structure.

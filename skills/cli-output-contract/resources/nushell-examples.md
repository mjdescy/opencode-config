# Nushell Script Implementation

For CLIs written directly in Nushell scripts (`.nu` files), implement the
output contract using `parse` and conditional flow.

## Pattern

Wrap the entire script body in a `def --wrapped` so `--quiet`, `--json`, and
`--help` work naturally with Nushell's argument passing.

```nu
# my-cli.nu
#
# My CLI tool — demonstrates the output contract in pure Nushell.

export def main [
    --quiet (-q)                    # Suppress all non-error output
    --json                          # Output as structured JSON
    input?: string                  # Optional positional argument
    ...rest                         # Forwarded to subcommands / skip flags
] {
    let quiet = $quiet
    let json = $json

    # --- Output helpers ---

    def write-text [msg: string] {
        if not $quiet and not $json {
            print $msg
        }
    }

    def write-json [payload: any] {
        if $json {
            $payload | to json --pretty 2 | print
        } else if not $quiet {
            # fallback human display (avoid duplicating with write-text calls)
        }
    }

    def write-warning [msg: string] {
        print -e $"warning: ($msg)"
    }

    def write-error [msg: string] {
        print -e $"error: ($msg)"
    }

    # --- Main logic ---

    # Honour NO_COLOR (Nushell has built-in support; just signal it)
    if ($env | get -i NO_COLOR | is-not-empty) {
        $env.NU_DISABLE_COLORS = true
    }

    # Respect --quiet/--json at the top level
    if $json {
        write-json { status: "ok", message: "hello from my-cli" }
    } else {
        write-text "Hello from my-cli"
    }
}

# --- Subcommand example ---
export def "main sub" [
    --quiet (-q)
    --json
    name: string
] {
    if $json {
        { status: "ok", name: $name } | to json --pretty 2 | print
    } else if not $quiet {
        print $"Hello, ($name)!"
    }
}
```

## Key Points for Nushell CLIs

| Requirement | How to satisfy |
|---|---|
| `--quiet` / `-q` | Gate `print` calls (and never use `print` implicitly). |
| `--json` | `$payload \| to json --pretty 2 \| print` |
| Errors to stderr | `print -e "error: ..."` |
| Exit codes | `exit 0` / `exit 1` |
| `--help` | Automatic with `def --wrapped` + `--help` |

## Running the Script

```nu
# Default (text)
source my-cli.nu
my-cli

# Quiet
my-cli --quiet

# JSON
my-cli --json

# Subcommand
my-cli sub --json "world"
```

## TTY Detection

Nushell doesn't expose `isatty` directly, but you can pass through to a
subprocess or use `(term size).columns` as a heuristic. For most scripts,
honouring `--no-color` and `NO_COLOR` is sufficient.

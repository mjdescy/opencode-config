---
name: cli-output-contract
description: Mandatory output-mode contract for every CLI or console app, in any language (C# / .NET, Rust, Python, Nushell, PowerShell, Bash). Default = clean text, --quiet/-q = silent, --json = structured JSON. Incorporates clig.dev guidelines.
---

# CLI Output Contract

Every command-line interface in this ecosystem **MUST** implement these three
output modes. This skill encodes that contract along with the essential
guidelines from [clig.dev](https://clig.dev/).

---

## Output Contract (Hard Requirements)

| Mode | Flag | Behaviour |
|---|---|---|
| Default (human text) | *(none)* | Clean, readable text to **stdout**. No progress spinners, no ANSI art, no logging noise. |
| Quiet | `--quiet` / `-q` | **No output to stdout whatsoever.** Exit code signals success/failure. Errors still go to stderr. |
| JSON | `--json` | Valid structured JSON to **stdout**. Must be parseable by `jq`. Errors go to stderr so stdout is always clean JSON. |

### Rules

1. Every command accepts `--quiet`, `-q`, and `--json` at the top level.
   Subcommands inherit these flags.
2. The three modes are **mutually exclusive**. If `--quiet` and `--json` are both
   passed, `--json` wins and `--quiet` is ignored (JSON is machine-readable,
   which is the point of quiet).
3. If the `--json` flag is present, **every** piece of output that would normally
   go to stdout is bundled into the JSON. No loose text lines leak out.
4. All human messaging (warnings, progress, info) goes to **stderr** in all
   three modes. In JSON mode, warnings may optionally be included in the JSON
   payload under a `"warnings"` key, but they *also* go to stderr.
5. In JSON mode, if the command produces a structured result, output a JSON
   **object** (not an array) with at minimum a `"status"` field. For
   list-oriented commands, wrap results in an object key (e.g. `{ "items": [...] }`).
   This leaves room for metadata fields (`"count"`, `"warnings"`, etc.) without
   breaking backward compatibility.

---

## Nushell‑First Design

Since these CLIs will often be called from Nushell, honour these additional
properties:

- **Default text output** must be line-oriented so `| lines`, `| each`, and
  `| where` work naturally.
- **JSON output** must interop with `from json`.
- **Exit codes** must be numeric (0 = success, non-zero = failure) so Nushell's
  `try`/`catch` and `$env.LAST_EXIT_CODE` work correctly.
- **Stderr** output should not pollute pipelines. The shell separates streams;
  the CLI must not print diagnostics to stdout.

---

## Essential clig.dev Guidelines (Must Follow)

These are the non-negotiable rules from [clig.dev](https://clig.dev/) that every
CLI in this ecosystem **must** follow.

### The Basics

- **Use an argument‑parsing library.** Never hand‑roll flag parsing.
- **Exit code 0 on success, non-zero on failure.** Map distinct exit codes to
  distinct failure modes (e.g. 1 = user error, 2 = runtime error).
- **Primary output → stdout.** Machine-readable output also goes to stdout.
- **Logs, warnings, errors → stderr.** Piped commands must not be contaminated.
- **Display output on success, but keep it brief.** If you change state, tell
  the user. Err on the side of *less* output.

### Help

- `-h` / `--help` at any level shows full help.
- Running with no args (when args are required) shows **concise** help +
  instruction to pass `--help` for full details.
- `help` subcommand works: `myapp help`, `myapp help subcommand`.
- Lead with **examples** in help text.
- Use terminal‑independent formatting (no raw escape sequences in piped output).

### Arguments & Flags

- **Prefer flags to positional args.**
- **Full‑length versions for all flags** (`--quiet` not just `-q`).
- Use **standard flag names** (`--help`, `--version`, `--quiet`, `--json`,
  `--force`, `--dry-run`, `--no-input`, `--output`, `--debug`).
- Never require a prompt. Always provide a flag/arg equivalent.
- Confirm before destructive actions. Support `--force` to skip confirmation.
- Accept `-` for stdin/stdout when file I/O is involved.

### Output (beyond the contract)

- **Human-readable first.** Use TTY detection to adjust output format.
- Use `--plain` for a script‑friendly plain-text mode when normal output
  uses multi-line formatting.
- **Color with intention.** Disable color when:
  - stdout/stderr is not a TTY (check individually),
  - `NO_COLOR` env var is set,
  - `TERM=dumb`,
  - `--no-color` is passed.
- No animations / progress bars when stdout is not a TTY.
- **Use a pager** (`less -FIRX`) when output is long and both stdin & stdout
  are TTYs.
- Don't print log‑level labels (`ERR`, `WARN`) to stderr by default. Save that
  for `--debug`/`--verbose`.

### Errors

- **Rewrite errors for humans.** Suggest the fix. Example: *"Can't write to
  file.txt. Try `chmod +w file.txt`."*
- **Signal‑to‑noise is crucial.** Group repeated errors under one header.
- Put the most important info at the **end** of error output.
- For unexpected errors, offer debug info + a link to report bugs.
- **Never print a stack trace to the user by default.** Show one in `--debug`.

### Interactivity

- **Only prompt if stdin is a TTY.** In scripts, error out and tell the user
  which flag to pass.
- **`--no-input`** disables all prompts.
- **Let the user escape.** Ctrl-C must always work.
- When prompting for secrets, **don't echo** the input.

### Subcommands

- If your tool has subcommands, apply all the above rules consistently across
  every subcommand.
- Use noun‑verb hierarchy: `docker container create`.
- Share global flags (like `--quiet`, `--json`) across all subcommands.

---

## Language‑Specific Implementation

For concrete examples, see the files in `resources/`:

| Language | Resource file | Recommended library |
|---|---|---|
| C# (.NET) | `resources/csharp-examples.md` | `System.CommandLine` |
| Rust | `resources/rust-examples.md` | `clap` (derive) |
| Python | `resources/python-examples.md` | `argparse` / `click` / `typer` |
| Nushell | `resources/nushell-examples.md` | Built‑in `def --wrapped` + `parse` |
| PowerShell | `resources/powershell-examples.md` | `param()` block + advanced functions |
| Bash | `resources/bash-examples.md` | `getopt` / `argbash` |

When implementing in a language not listed here, map the output contract to
that language's best‑in‑class argument‑parsing library (see
[clig.dev's recommendations](https://clig.dev/#the-basics)).

---

## Enforcement Checklist

Use this when reviewing any CLI PR:

- [ ] `--quiet` / `-q` suppresses all stdout? (stderr still flows)
- [ ] `--json` produces valid JSON to stdout, errors to stderr?
- [ ] `-h` / `--help` works at every level?
- [ ] `--help` + `--json` together? (help should win — help is a meta-action)
- [ ] Exit code 0 on success, non-zero on failure?
- [ ] Errors on stderr, output on stdout?
- [ ] `NO_COLOR` honoured?
- [ ] No prompts when stdin is not a TTY?
- [ ] `--no-input` honoured?
- [ ] Destructive actions require confirmation or `--force`?

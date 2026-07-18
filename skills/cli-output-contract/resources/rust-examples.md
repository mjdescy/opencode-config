# Rust Implementation (clap)

Use `clap` with the `derive` feature — it generates `--help`, `--version`, and
exit codes automatically, and makes it easy to add global flags.

## Cargo.toml

```toml
[package]
name = "my-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## Global Options Struct

```rust
#[derive(clap::Parser)]
#[command(name = "my-cli", version)]
pub struct Cli {
    /// Suppress all non-error output
    #[arg(short = 'q', long = "quiet", global = true)]
    pub quiet: bool,

    /// Output as structured JSON
    #[arg(long = "json", global = true)]
    pub json: bool,

    /// Disable colored output
    #[arg(long = "no-color", global = true)]
    pub no_color: bool,

    #[command(subcommand)]
    pub command: Option<Commands>,
}
```

## Output Helper

```rust
pub struct OutputWriter<W: std::io::Write> {
    quiet: bool,
    json: bool,
    stdout: W,
    stderr: W,
}

impl<W: std::io::Write> OutputWriter<W> {
    pub fn new(quiet: bool, json: bool, stdout: W, stderr: W) -> Self {
        Self { quiet, json, stdout, stderr }
    }

    /// Write a line of human-readable text (default mode).
    pub fn text(&mut self, line: impl AsRef<str>) {
        if !self.quiet && !self.json {
            writeln!(self.stdout, "{}", line.as_ref()).ok();
        }
    }

    /// Write a JSON result to stdout (JSON mode).
    pub fn json_result<T: serde::Serialize>(&mut self, payload: &T) {
        if self.json {
            let json = serde_json::to_string_pretty(payload).unwrap();
            writeln!(self.stdout, "{json}").ok();
        }
    }

    /// Write a warning to stderr (all modes).
    pub fn warn(&mut self, message: impl AsRef<str>) {
        writeln!(self.stderr, "warning: {}", message.as_ref()).ok();
    }

    /// Write an error to stderr (all modes).
    pub fn error(&mut self, message: impl AsRef<str>) {
        writeln!(self.stderr, "error: {}", message.as_ref()).ok();
    }
}
```

## Wiring It Up

```rust
use clap::Parser;

fn main() {
    let cli = Cli::parse();

    // Respect NO_COLOR
    if cli.no_color || std::env::var_os("NO_COLOR").is_some_and(|v| !v.is_empty()) {
        // Disable ANSI — e.g. set an env var consumed by a formatting lib
        std::env::set_var("NO_COLOR", "1");
    }

    let mut output = OutputWriter::new(cli.quiet, cli.json, std::io::stdout(), std::io::stderr());

    match &cli.command {
        None => {
            output.text("Running default command...");
            // ...
        }
        Some(cmd) => {
            // handle subcommands
        }
    }
}
```

## Recommended Crates

| Crate | Purpose |
|-------|---------|
| `clap` (derive) | Argument parsing, help, version |
| `serde` + `serde_json` | JSON serialization |
| `is-terminal` | TTY detection for pager/color decisions |

---
name: rust
description: Use when working with Rust code, .rs files, Cargo.toml, Cargo.lock, or Rust projects. Covers conventions, patterns, preferred crates, and tooling for Rust development.
---

# Rust Conventions

## Code Style & Naming

| Symbol Kind | Style | Example |
|---|---|---|
| Types (struct, enum, trait) | PascalCase | `OrderProcessor` |
| Functions, methods, locals | snake_case | `process_order` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Statics | SCREAMING_SNAKE_CASE | `GLOBAL_CONFIG` |
| Module names | snake_case | `order_processing` |
| Type parameters | concise PascalCase | `T`, `E`, `Item` |
| Lifetimes | single lowercase | `'a`, `'ctx` |
| Cargo features | kebab-case | `serde-support` |
| Builder methods | `set_*`, `with_*`, `build` | `.with_name("foo").build()`

- `use` statements grouped: `std` → external crates → `crate`/`super`, separated by blank lines
- Import items, not modules (`use std::collections::HashMap`), unless avoiding name clashes
- Prefer `Self` over repeating the type name in `impl` blocks
- Derive `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Default` where semantically correct (in that order)
- Omit `return` for trailing expressions; use `return` only for early exits
- Use `#[must_use]` on functions returning `Result` or values that should not be discarded

## Project Structure

```
my-crate/
├── Cargo.toml
├── Cargo.lock
├── src/
│   ├── lib.rs          # library root, re-exports
│   ├── main.rs         # binary entry point (if applicable)
│   ├── config.rs
│   ├── error.rs
│   └── cli.rs
├── tests/              # integration tests
│   └── integration_test.rs
├── benches/            # benchmarks
├── examples/           # example binaries
└── build.rs            # build script (if needed)
```

- Use a workspace (`[workspace]`) for multi-crate projects; workspace-level `Cargo.toml` for shared config
- One `lib.rs` and optional `main.rs`; keep `lib.rs` as a thin re-export surface
- Module tree mirrors filesystem; declare with `mod` only in `lib.rs`/`main.rs`
- Use `pub use crate::foo::Bar` to control public API shape, not `pub mod` re-exports from leaf modules

## Error Handling

```rust
use std::fmt;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    Validation { field: String, reason: String },
    Io(std::io::Error),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::NotFound(id) => write!(f, "resource not found: {id}"),
            Self::Validation { field, reason } => {
                write!(f, "validation error on {field}: {reason}")
            }
            Self::Io(inner) => write!(f, "I/O error: {inner}"),
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::Io(inner) => Some(inner),
            _ => None,
        }
    }
}
```

- Define domain-specific error types; avoid bare `Box<dyn Error>` in public APIs
- Implement `Display` + `Error` manually for app errors, or use `thiserror` derive for brevity:
  ```rust
  #[derive(thiserror::Error, Debug)]
  pub enum AppError {
      #[error("resource not found: {0}")]
      NotFound(String),
  }
  ```
- Use `anyhow::Result` in binaries/tests for convenience; use custom error types in libraries
- Use `.context()` / `.with_context()` from `anyhow` or `eyre` to enrich errors
- Prefer `map_err` for one-off conversions, `From` impls for systematic conversions
- Handle `Result` explicitly — avoid `.unwrap()` / `.expect()` in production code (use only in tests and infallible positions)

## Patterns & Design

- Use `Option<T>` for nullable values, never raw pointers or sentinel values
- Use `Result<T, E>` for fallible operations
- Favor `match` and `if let` over index-based access; avoid `[0]` on slices
- Use the builder pattern for structs with many optional fields:
  ```rust
  #[derive(Default)]
  pub struct ConfigBuilder {
      timeout: Option<Duration>,
      retries: Option<usize>,
  }

  impl ConfigBuilder {
      pub fn timeout(mut self, dur: Duration) -> Self {
          self.timeout = Some(dur);
          self
      }
      pub fn build(self) -> Config {
          Config {
              timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
              retries: self.retries.unwrap_or(3),
          }
      }
  }
  ```
- Use `From`/`TryFrom` for conversions, not custom `to_*` methods (except `to_string()`)
- Prefer iterators (`map`, `filter`, `fold`, `collect`) over manual loops where clarity is equal
- Use `impl Trait` in argument position for generic parameters; turbofish `::<>` only when inference fails
- Use `Arc<str>` or `Box<str>` over `Arc<String>` / `Box<String>` for read-only strings
- Use `Cow<'_, str>` when a function sometimes borrows, sometimes owns a string
- Use `#[non_exhaustive]` on public enums and structs in libraries to allow future variants/fields
- Avoid `unsafe` unless necessary for FFI or performance; wrap unsafe blocks in safe abstractions

## Async & Concurrency

- Use `tokio` as the async runtime; `async-std` is acceptable for stdlib-aligned projects
- Prefer `tokio::sync` channels (`mpsc`, `oneshot`, `broadcast`) over `std::sync::mpsc`
- Use `tokio::spawn` with `JoinHandle` for fire-and-forget or joinable tasks
- For shared state, prefer `tokio::sync::RwLock` or `std::sync::Arc<Mutex<T>>`; minimize lock scope
- Use `tokio::select!` for racing futures or timeouts
- Prefer `Stream`/`futures::StreamExt` over channel-based iteration for async sequences
- Mark async fn signatures with concrete return types, not `impl Future`

## Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_process_order_success() {
        let result = process_order("valid_input");
        assert!(result.is_ok());
    }

    #[test]
    fn test_process_order_invalid_input() {
        let result = process_order("");
        assert!(result.is_err());
    }
}
```

- Unit tests in a `#[cfg(test)] mod tests` at the bottom of each source file
- Integration tests in `tests/` directory — one file per integration scenario
- Name tests `test_<description>`; use `#[should_panic]` sparingly (prefer `assert!(…)`)
- Use `assert_eq!`, `assert_ne!`, `assert!` over manual `if`-`panic`
- Use `Result<()>` return type in tests for `?` propagation:
  ```rust
  #[test] fn test_foo() -> Result<()> { ... }
  ```
- Use `#[tokio::test]` for async tests
- Use `rstest` for parameterized tests:
  ```rust
  #[rstest]
  #[case(1, 2, 3)]
  #[case(0, 0, 0)]
  fn test_add(#[case] a: i32, #[case] b: i32, #[case] expected: i32) { ... }
  ```
- Prefer table tests with rstest over multiple `#[test]` functions

## Crates & Dependencies

| Category | Preferred Crate | Notes |
|---|---|---|
| Error derive | `thiserror` | Domain error types in libraries |
| Error reporting | `anyhow` | Convenient error handling in apps/binaries |
| Async runtime | `tokio` | Full-featured, de facto standard |
| Serialization | `serde` + `serde_json` | Only JSON; add `serde_yaml`, `toml` per need |
| CLI | `clap` | Derive API (`#[derive(Parser)]`) |
| HTTP client | `reqwest` | Async, TLS bundled |
| HTTP server | `axum` | ergonomic, tower-based |
| Logging | `tracing` | Structured, async-aware; use `tracing-subscriber` |
| Database | `sqlx` | Async, compile-time checked queries |
| DateTime | `chrono` | Timezone-aware |
| UUID | `uuid` | With `v4`/`v7` features |
| Random | `rand` | De facto standard |
| Regex | `regex` | Compiled at runtime |
| Testing | `rstest` | Parameterized tests |

## Tooling

| Command | Purpose |
|---|---|
| `cargo build` | Build project |
| `cargo check` | Type-check without codegen (faster than build) |
| `cargo test` | Run unit + integration tests |
| `cargo clippy` | Lint with Clippy (use `-- -D warnings` to fail on warnings) |
| `cargo fmt` | Format code (`rustfmt`) |
| `cargo doc --open` | Build docs and open in browser |
| `cargo add <crate>` | Add dependency |
| `cargo outdated` | Check for outdated deps |
| `cargo audit` | Check for security advisories |
| `cargo nextest` | Faster test runner with better reporting |
| `cargo expand` | Expand macros for debugging |
| `cargo watch` | Auto-run on file changes |

## Cargo.toml Conventions

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"
license = "MIT OR Apache-2.0"
description = "A short, concise description."

[dependencies]
serde = { version = "1", features = ["derive"] }

[dev-dependencies]
tempfile = "3"

[features]
default = ["std"]
std = []
```

- Edition `2021` (minimum); migrate to `2024` when stable
- Set `rust-version` to MSRV
- License `MIT OR Apache-2.0` dual-license for libraries
- Use caret requirements (`"1.2"` = `^1.2`) — specify minimum compatible version, avoid exact pins
- Keep feature flags additive; never make a feature disable functionality
- Use `[lints]` table or `clippy.toml` for repo-level lint config

# Python Implementation (argparse / click / typer)

Python has several mature argument‑parsing libraries. Use whichever suits the
project — the output contract is the same regardless.

## Option A: `argparse` (stdlib, no dependencies)

```python
#!/usr/bin/env python3
"""My CLI tool — demonstrates the output contract with argparse."""

import argparse
import json
import sys


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="My CLI tool")
    parser.add_argument("--quiet", "-q", action="store_true",
                        help="Suppress all non-error output")
    parser.add_argument("--json", action="store_true",
                        help="Output as structured JSON")
    parser.add_argument("--no-color", action="store_true",
                        help="Disable colored output")
    return parser


class OutputWriter:
    def __init__(self, quiet: bool, json_mode: bool):
        self.quiet = quiet
        self.json = json_mode

    def text(self, line: str) -> None:
        if not self.quiet and not self.json:
            print(line)

    def json_result(self, payload: object) -> None:
        if self.json:
            print(json.dumps(payload, indent=2, default=str))

    def warn(self, message: str) -> None:
        print(f"warning: {message}", file=sys.stderr)

    def error(self, message: str) -> None:
        print(f"error: {message}", file=sys.stderr)


def main() -> int:
    parser = build_parser()
    args = parser.parse_args()

    # Respect NO_COLOR
    if args.no_color or os.environ.get("NO_COLOR"):
        os.environ["NO_COLOR"] = "1"

    output = OutputWriter(args.quiet, args.json)

    output.text("Processing...")
    output.json_result({"status": "ok", "data": [1, 2, 3]})

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Option B: `click` (decorator‑based, popular)

```python
import json
import sys
import click


class OutputWriter:
    def __init__(self, quiet: bool, json_mode: bool):
        self.quiet = quiet
        self.json = json_mode

    def text(self, line: str) -> None:
        if not self.quiet and not self.json:
            click.echo(line)

    def json_result(self, payload: object) -> None:
        if self.json:
            click.echo(json.dumps(payload, indent=2, default=str))

    def warn(self, message: str) -> None:
        click.echo(f"warning: {message}", err=True)

    def error(self, message: str) -> None:
        click.echo(f"error: {message}", err=True)


@click.group()
@click.option("--quiet", "-q", is_flag=True, help="Suppress non-error output")
@click.option("--json", is_flag=True, help="Output as structured JSON")
@click.option("--no-color", is_flag=True, help="Disable colored output")
@click.pass_context
def cli(ctx, quiet, json, no_color):
    """My CLI tool."""
    ctx.ensure_object(dict)
    ctx.obj["output"] = OutputWriter(quiet, json)
    if no_color or os.environ.get("NO_COLOR"):
        os.environ["NO_COLOR"] = "1"


@cli.command()
@click.pass_context
def run(ctx):
    output = ctx.obj["output"]
    output.text("Running...")
    output.json_result({"status": "ok"})


if __name__ == "__main__":
    cli()
```

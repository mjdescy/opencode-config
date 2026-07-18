# PowerShell Implementation (Advanced Functions)

PowerShell has built‑in `param()` blocks with `[switch]` parameters, automatic
`-h`/`-?` help, and `$?` / `$LASTEXITCODE` for exit code discipline.

## Pattern

```powershell
<#
.SYNOPSIS
    My CLI tool
.DESCRIPTION
    Demonstrates the output contract in PowerShell.
.PARAMETER Quiet
    Suppress all non-error output.
.PARAMETER Json
    Output as structured JSON.
#>
function Invoke-MyCli {
    [CmdletBinding()]
    param(
        [switch]$Quiet,
        [switch]$Json,
        [switch]$NoColor
    )

    # Respect NO_COLOR
    if ($NoColor -or $env:NO_COLOR) {
        $env:NO_COLOR = '1'
    }

    # --- Output helpers ---
    function Write-Text([string]$Message) {
        if (-not $Script:Quiet -and -not $Script:Json) {
            Write-Output $Message
        }
    }

    function Write-JsonResult($Payload) {
        if ($Script:Json) {
            $Payload | ConvertTo-Json -Depth 10 | Write-Output
        }
    }

    function Write-WarningMsg([string]$Message) {
        Write-Warning $Message
    }

    function Write-ErrorMsg([string]$Message) {
        Write-Error $Message
    }

    # --- Main logic ---
    Write-Text "Processing..."

    if ($Json) {
        Write-JsonResult @{ status = "ok"; data = @(1, 2, 3) }
    }

    return 0
}

# Entry point (when invoked as a script, not dot-sourced)
$exitCode = Invoke-MyCli @args
exit $exitCode
```

## Key Points for PowerShell CLIs

| Requirement | How to satisfy |
|---|---|
| `--quiet` / `-q` | `[switch]$Quiet` in `param()` block. PowerShell also accepts `-q`. |
| `--json` | `[switch]$Json` — output via `ConvertTo-Json` |
| Errors to stderr | `Write-Error` (goes to stderr by default) |
| Warnings to stderr | `Write-Warning` |
| Exit codes | `exit 0` / `exit $LASTEXITCODE` |
| `--help` / `-?` | Automatic via `[CmdletBinding()]` |
| `--no-color` | `$PSStyle.OutputRendering = [System.ConsoleColor]::Gray` or `$env:NO_COLOR` |

## As a Script File

Save as `my-cli.ps1` with the function at the bottom for readability:

```powershell
#!/usr/bin/env pwsh
param(
    [switch]$Quiet,
    [switch]$Json,
    [switch]$NoColor
)

# ... (same body as above, just inline)
```

## TTY Detection

```powershell
$isTTY = [Console]::IsOutputRedirected -eq $false
```

Use this to skip pagers / progress in non‑interactive contexts.

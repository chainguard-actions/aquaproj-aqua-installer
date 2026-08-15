<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.5** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: The env var `$AQUA_OPTS` (sourced from `inputs.aqua_opts`, which is caller-controlled) is expanded unquoted in the shell command `run: aqua i $AQUA_OPTS`. An attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) could be injected to execute arbitrary commands. The fix is to quote the expansion: `aqua i "$AQUA_OPTS"`.

Locations:

- `action.yaml:77`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted `$AQUA_OPTS` expansion in `hardened/action/action.yaml` at line 77. Changed `run: aqua i $AQUA_OPTS` to `run: aqua i "$AQUA_OPTS"` to prevent shell metacharacter injection from attacker-controlled `inputs.aqua_opts` values. The variable was already correctly sourced via the step's `env:` block; only the quoting in the shell command needed to be fixed.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed all three steps in action.yaml that wrote AQUA_ROOT_DIR to $GITHUB_PATH without sanitization:
1. Bash (non-Windows): Expanded to a multi-line block that captures the resolved path into a variable, strips newlines/CRs with `printf '%s' "$_aqua_bin" | tr -d '\n\r'`, then writes the sanitized value to $GITHUB_PATH.
2. Bash (Windows): Same approach as above for the Windows default path variant.
3. PowerShell (Windows): Used `-replace '[\r\n]', ''` to strip newlines from the resolved AQUA_ROOT_DIR value before writing to $env:GITHUB_PATH via Add-Content.
In all cases, AQUA_ROOT_DIR (an environment variable inherited from the calling workflow and therefore untrusted) is now sanitized before being written to GITHUB_PATH, preventing newline injection attacks.


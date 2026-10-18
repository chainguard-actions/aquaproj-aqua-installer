<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: The shell variable $AQUA_OPTS is expanded unquoted in the run command `aqua i $AQUA_OPTS`. This variable is sourced from `inputs.aqua_opts` (set via `env: AQUA_OPTS: ${{ inputs.aqua_opts }}`). An attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) would be parsed by the shell, enabling command injection. The fix is to quote the expansion: `aqua i "$AQUA_OPTS"`.

Locations:

- `action.yaml:84`

### github-env-injection (severity: high)

Three `run:` steps write the inherited process env var `AQUA_ROOT_DIR` to `$GITHUB_PATH` without sanitization. `AQUA_ROOT_DIR` is not set by the action itself — it is inherited from the calling workflow's environment, making it a workflow-controlled (untrusted) value. A calling workflow could set `AQUA_ROOT_DIR` to a value containing newline characters, injecting additional arbitrary entries into `$GITHUB_PATH`. The fix is to sanitize the value before writing: `safe=$(printf '%s' "$AQUA_ROOT_DIR" | tr -d '\n\r')` and use `$safe` in the echo. Affected steps: (1) bash step for non-Windows writing `${AQUA_ROOT_DIR:-...}/bin >> $GITHUB_PATH`; (2) bash step for Windows writing `${AQUA_ROOT_DIR:-...}/bin >> $GITHUB_PATH`; (3) PowerShell step writing `$env:AQUA_ROOT_DIR` via `Out-File` to `$env:GITHUB_PATH`.

Locations:

- `action.yaml:43`
- `action.yaml:48`
- `action.yaml:54`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all four locations in action.yaml:
1. Non-Windows bash step (line 43): sanitize AQUA_ROOT_DIR with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_PATH.
2. Windows bash step (line 48): same sanitization for AQUA_ROOT_DIR.
3. PowerShell step (line 54): use `-replace '[\r\n]', ''` to strip newlines from AQUA_ROOT_DIR before writing to $GITHUB_PATH.
4. aqua install step (line 84): replaced unquoted `$AQUA_OPTS` expansion with xargs-based tokenization into a bash array, then `aqua i "${opts[@]}"` to prevent shell metacharacter injection while preserving multi-flag support.


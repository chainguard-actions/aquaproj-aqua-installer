<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.5** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: The `run:` step `aqua i $AQUA_OPTS` expands the shell variable `$AQUA_OPTS` without double-quoting it. `AQUA_OPTS` is sourced directly from `${{ inputs.aqua_opts }}` via the `env:` block, making it workflow-controllable. An attacker can supply shell metacharacters (e.g. `;`, `|`, `&`, `$(...)`) in `inputs.aqua_opts` to inject arbitrary commands. The fix is to quote the expansion: `aqua i "$AQUA_OPTS"`.

Locations:

- `action.yaml:82`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the unquoted `$AQUA_OPTS` expansion in `action.yaml` at line 82. Replaced `aqua i $AQUA_OPTS` with a safe xargs-based tokenization approach that builds a bash array from the options string (respecting shell quoting within the value), then expands it as `aqua i "${opts[@]}"`. This prevents shell metacharacter injection while correctly handling multiple space-separated flags in the `aqua_opts` input. The `AQUA_OPTS` env var was already sourced from `${{ inputs.aqua_opts }}` in the env: block.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed all three steps in action.yaml that wrote environment-inherited variables ($AQUA_ROOT_DIR, $XDG_DATA_HOME) to $GITHUB_PATH without sanitization:

1. Linux bash step (was line 44): Now captures the path into `aqua_bin_dir`, sanitizes with `printf '%s' "$aqua_bin_dir" | tr -d '\n\r'` into `safe_dir`, then writes `safe_dir` to $GITHUB_PATH.

2. Windows bash step (was line 48): Same approach — captures path, sanitizes with `tr -d '\n\r'`, writes sanitized value.

3. Windows PowerShell step (was line 53): Captures path into `$aquaBinDir`, sanitizes with `-replace '[\r\n]', ''` into `$safeDir`, then writes `$safeDir` to $GITHUB_PATH.

All three fixes follow the prescribed pattern: compute the value first, sanitize it in a separate step (not nested), then write the sanitized value to the special environment file.


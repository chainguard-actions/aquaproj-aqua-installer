<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.2** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (b) violation: The `run: aqua i $AQUA_OPTS` step in action.yaml expands the shell variable `$AQUA_OPTS` without double-quoting it. `AQUA_OPTS` is sourced directly from `inputs.aqua_opts` (a user-controlled input). An attacker calling this composite action can supply shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) in `aqua_opts` to inject arbitrary shell commands. The fix is to quote the expansion: `run: aqua i "$AQUA_OPTS"`.

Locations:

- `action.yaml:83`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the unquoted `$AQUA_OPTS` expansion in the `aqua i` step. Since `aqua_opts` is a list of command-line options, used the xargs-based array tokenization pattern: tokenize `$AQUA_OPTS` into an `opts` array using `printf '%s' "$AQUA_OPTS" | xargs printf '%s\0'` with a null-delimited read loop, then expand as `aqua i "${opts[@]}"`. This safely handles multiple options and quoted arguments while preventing shell injection from user-controlled input.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed all three $GITHUB_PATH injection vulnerabilities in action.yaml:

1. Non-Windows bash step (was line 45): Now computes the aqua bin path into a variable, pipes it through `tr -d '\n\r'` to strip newlines, then writes the sanitized value to "$GITHUB_PATH".

2. Windows bash step (was line 51): Same pattern — computes path into variable, sanitizes with `printf '%s' ... | tr -d '\n\r'`, then writes to "$GITHUB_PATH".

3. Windows PowerShell step (was line 57): Now uses `-replace '[\r\n]', ''` to strip carriage returns and newlines from the computed path before writing it to $env:GITHUB_PATH via `Add-Content`.

All three steps now prevent newline injection that could allow an attacker-controlled environment variable (AQUA_ROOT_DIR, XDG_DATA_HOME, or HOME) to inject additional entries into the runner's PATH.


<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Three steps in action.yaml write the inherited environment variable $AQUA_ROOT_DIR (and $XDG_DATA_HOME) to $GITHUB_PATH without sanitization. These variables are inherited from the calling workflow and are therefore workflow-controlled (untrusted). A caller could set AQUA_ROOT_DIR to a value containing newline characters, injecting arbitrary entries into $GITHUB_PATH. The required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`) is absent before each write.

- Line 45 (bash, Linux): `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`
- Line 51 (bash, Windows): `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`
- Line 57 (pwsh, Windows): `$(if($env:AQUA_ROOT_DIR) {echo $env:AQUA_ROOT_DIR} ...) | Out-File -FilePath $env:GITHUB_PATH`

Locations:

- `action.yaml:45`
- `action.yaml:51`
- `action.yaml:57`

### script-injection (severity: high)

Rule (b) violation: The `run:` step `aqua i $AQUA_OPTS` (line 88 of action.yaml) expands the shell variable $AQUA_OPTS without double-quoting it. $AQUA_OPTS is sourced from `inputs.aqua_opts` (via the `env:` block), which is caller-controlled. An attacker-supplied value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) would be interpreted by the shell, enabling command injection. The fix is to quote the expansion: `aqua i "$AQUA_OPTS"`.

Locations:

- `action.yaml:88`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed all four locations:
1. Line 45 (Linux bash): Sanitized AQUA_ROOT_DIR/XDG_DATA_HOME with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_PATH.
2. Line 51 (Windows bash): Same sanitization pattern for AQUA_ROOT_DIR before writing to $GITHUB_PATH.
3. Line 57 (Windows pwsh): Used PowerShell `-replace '[\r\n]', ''` to strip newlines from AQUA_ROOT_DIR before writing to $GITHUB_PATH.
4. Line 88 (script-injection): Quoted `$AQUA_OPTS` as `"$AQUA_OPTS"` in `aqua i "$AQUA_OPTS"` to prevent shell metacharacter injection from caller-controlled input.


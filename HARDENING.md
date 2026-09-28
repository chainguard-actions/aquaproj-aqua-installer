<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v3.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v3.1.2** was hardened automatically. 1 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: The `run: aqua i $AQUA_OPTS` step uses an unquoted shell variable `$AQUA_OPTS` that is sourced from `inputs.aqua_opts` via the `env:` block (`AQUA_OPTS: ${{ inputs.aqua_opts }}`). An attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) in `inputs.aqua_opts` could cause command injection. The variable must be double-quoted: `aqua i "$AQUA_OPTS"`. Note: since `aqua_opts` is an optional input with a default of `-l` and is used as a flag argument (not a positional argument), the guarded form `${AQUA_OPTS:+"$AQUA_OPTS"}` is not applicable here — straightforward double-quoting is the correct fix.

Locations:

- `action.yaml:75`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted shell variable in action.yaml line 75: changed `aqua i $AQUA_OPTS` to `aqua i "$AQUA_OPTS"`. The `AQUA_OPTS` variable is already properly sourced through the `env:` block, so double-quoting it in the shell command prevents shell metacharacter injection from attacker-controlled `inputs.aqua_opts` values.


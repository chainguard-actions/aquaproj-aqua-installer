<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Composite step writes the inherited, workflow-controlled env vars AQUA_ROOT_DIR / XDG_DATA_HOME to $GITHUB_PATH with no printf '%s' ... | tr -d '\n\r' cleanup first: `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`.

Locations:

- `action.yaml:45`

### github-env-injection (severity: high)

Windows bash step writes the inherited env var AQUA_ROOT_DIR to $GITHUB_PATH with no newline cleanup first: `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`.

Locations:

- `action.yaml:51`

### github-env-injection (severity: high)

pwsh step writes the inherited $env:AQUA_ROOT_DIR to $env:GITHUB_PATH through Out-File with no CR/LF cleanup first. An embedded newline could inject extra PATH entries.

Locations:

- `action.yaml:57`

### script-injection (severity: high)

Sub-rule (b): an env var built from inputs.aqua_opts is expanded unquoted: `run: aqua i $AQUA_OPTS`. Word splitting and glob expansion apply to the value supplied by the workflow. Quote it, for example `aqua i ${AQUA_OPTS:+"$AQUA_OPTS"}`, or parse it into an array safely.

Locations:

- `action.yaml:88`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection

**Notes:**

All four findings are fixed in action.yaml. Nothing was run to test the changes.

- **Writes to $GITHUB_PATH (3 findings):** The two bash steps now save the aqua bin path to a variable, strip newlines and carriage returns with `printf '%s' ... | tr -d '\n\r'`, and then write it to "$GITHUB_PATH". The pwsh step strips them with `-replace "[`r`n]", ""` before `Out-File`.
- **`aqua_opts` (1 finding):** I didn't use the suggested `${AQUA_OPTS:+"$AQUA_OPTS"}`. `aqua_opts` is a list of options, and that form would pass the whole list to aqua as one argument. Instead the value is split on whitespace into an array with `read -r -d '' -a args <<< "$AQUA_OPTS" || true`, inside an `if [ -n ... ]` check, and the step runs `aqua i "${args[@]}"`. That matches how the original unquoted `$AQUA_OPTS` was split, except that wildcard characters are no longer expanded into file names. Quotes in the value are still treated as plain text, as before.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

In action.yaml, the 'aqua i $AQUA_OPTS' step now splits AQUA_OPTS into an array with `read -r -d '' -a opts <<< "$AQUA_OPTS" || true` (guarded by -n) and runs `aqua i "${opts[@]}"`. It still splits on whitespace and newlines like upstream's unquoted $VAR, so the default '-l' and multi-flag values work as before, but glob characters are no longer expanded.


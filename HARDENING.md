<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.5** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Composite step writes a path built from the inherited env vars AQUA_ROOT_DIR and XDG_DATA_HOME, which the calling workflow controls, to $GITHUB_PATH without sanitizing it first (no printf '%s' | tr -d '\n\r'): `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`

Locations:

- `action.yaml:45`

### github-env-injection (severity: high)

Windows bash step writes a path built from the inherited env var AQUA_ROOT_DIR to $GITHUB_PATH without stripping newlines: `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`

Locations:

- `action.yaml:51`

### github-env-injection (severity: high)

Windows pwsh step writes the inherited $env:AQUA_ROOT_DIR straight to $env:GITHUB_PATH with Out-File and does not strip newlines first, so a value containing newlines can inject extra PATH entries.

Locations:

- `action.yaml:57`

### script-injection (severity: high)

Sub-rule (b): the env var AQUA_OPTS comes from inputs.aqua_opts but is expanded without quotes: `run: aqua i $AQUA_OPTS`. The shell word-splits and glob-expands the input value.

Locations:

- `action.yaml:89`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Bash GITHUB_PATH writes (non-Windows and Windows): the path is built into a variable, newlines are removed with printf '%s' | tr -d '\n\r', and the result is written to the quoted "$GITHUB_PATH". Pwsh step: CR/LF are removed with -replace before Out-File. Upstream's behaviour is kept as it was, including that it does not add /bin when AQUA_ROOT_DIR is set. aqua i: the unquoted $AQUA_OPTS is now split into a bash array with read -r -d '' -a, wrapped in an empty-value check and followed by || true, then passed as "${args[@]}". It splits on the same whitespace as before but no longer expands globs.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

In action.yaml, the `aqua i $AQUA_OPTS` step now reads AQUA_OPTS into an array with `read -r -d '' -a` (guarded by an emptiness check, with `|| true`) and runs `aqua i "${opts[@]}"`. Like upstream's unquoted $AQUA_OPTS, it still splits on whitespace, so multi-option values such as '-l -a' keep working. Unlike upstream, it no longer expands globs.


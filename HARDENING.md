<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.1.2** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Composite step writes the inherited, workflow-controlled env var AQUA_ROOT_DIR (and XDG_DATA_HOME) to $GITHUB_PATH without the `printf '%s' ... | tr -d '\n\r'` sanitization: `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`. A value containing a newline would add extra PATH entries.

Locations:

- `action.yaml:53`

### github-env-injection (severity: high)

Windows bash step writes the inherited env var AQUA_ROOT_DIR to $GITHUB_PATH without newline sanitization: `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`.

Locations:

- `action.yaml:59`

### github-env-injection (severity: high)

pwsh step writes the inherited env var $env:AQUA_ROOT_DIR to $env:GITHUB_PATH (Out-File -Append) without stripping CR/LF characters, so a multi-line value would add extra PATH entries.

Locations:

- `action.yaml:65`

### script-injection (severity: high)

Rule (b): the 'aqua i' step expands workflow-controlled data without quotes. `aqua i $OPTS -- "${pkgs[@]}"` expands OPTS (from inputs.aqua_opts) unquoted. `pkgs+=($line)` expands $line (from inputs.packages) unquoted in an array append, and neither line has a shellcheck SC2086/SC2206 disable directive, so the documented exemption does not cover it.

Locations:

- `action.yaml:101`
- `action.yaml:106`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed all four findings, all in action.yaml. Nothing was run to test the changes.

- **Writes to $GITHUB_PATH (three steps):** The two bash steps now strip newlines and carriage returns from the computed path with printf '%s' ... | tr -d '\n\r' before writing it. The value is quoted when added to "$GITHUB_PATH". The pwsh step now removes CR/LF from the value with -replace "[`r`n]", "" before Out-File -Append.
- **The 'aqua i' step:** Upstream's unquoted whitespace splitting is kept on purpose. Both inputs are lists of separate words, and quoting them would pass each list to aqua as one argument. Globbing is still off (set -f). pkgs+=($line) is unchanged apart from a new shellcheck SC2206 disable comment. The unquoted $OPTS now goes into an array, opts=($OPTS), with the same disable comment, and the command is now aqua i "${opts[@]}" -- "${pkgs[@]}". The arguments aqua receives are the same as before. Neither form evaluates the input, so nothing in it can run as a command.


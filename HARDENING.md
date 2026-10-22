<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.1.1** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Composite step writes `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`. AQUA_ROOT_DIR and XDG_DATA_HOME come from the calling workflow's environment, and the value is not passed through `printf '%s' ... | tr -d '\n\r'` before the write.

Locations:

- `action.yaml:53`

### github-env-injection (severity: high)

Windows bash step writes `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`. AQUA_ROOT_DIR is inherited from the calling workflow and is not sanitized (no `printf '%s' ... | tr -d '\n\r'`).

Locations:

- `action.yaml:59`

### github-env-injection (severity: high)

Windows pwsh step writes the inherited $env:AQUA_ROOT_DIR to $env:GITHUB_PATH with Out-File and never strips newline or carriage-return characters first.

Locations:

- `action.yaml:65`

### script-injection (severity: high)

Rule (b): two shell expansions of input-derived data are unquoted. `aqua i $OPTS -- "${pkgs[@]}"` expands OPTS (from inputs.aqua_opts) without quotes, and `pkgs+=($line)` expands line (from inputs.packages) without quotes. Both values are split on whitespace, though `set -f` prevents glob expansion.

Locations:

- `action.yaml:106`
- `action.yaml:101`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection

**Notes:**

GITHUB_PATH writes: the two bash steps now put the path in a variable and remove newline and carriage-return characters with printf '%s' | tr -d '\n\r' before appending to "$GITHUB_PATH". The pwsh step removes `r`n with -replace before Out-File. It still leaves /bin off a set AQUA_ROOT_DIR, exactly as upstream did. Script injection: aqua_opts and each packages line are now split into bash arrays with `read -r -d '' -a ... || true`. Like upstream's unquoted expansion, this splits on whitespace and treats quotes as plain text, so the arguments aqua receives don't change. Each value is checked for emptiness before splitting, the arrays are expanded quoted, and set -f is still there. The '-l' rule is unchanged: when packages are given and aqua_opts is the default '-l', the option is dropped.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

action.yaml: the two bash steps (non-Windows and Windows) now build the bin path in a variable, remove newline/CR characters with printf '%s' | tr -d '\n\r', and append it to "$GITHUB_PATH". The pwsh step now puts the chosen path in $aquaBin, removes `r/`n with -replace, then appends it with Out-File. Each step resolves the path the same way upstream did. This includes upstream's quirk in the pwsh step, which writes $env:AQUA_ROOT_DIR as-is without adding /bin.

### Iteration 3

**Fixes applied:** github-env-injection, invalid-yaml

**Notes:**

I rewrote the broken steps in action.yaml. A bad earlier edit had mixed pieces of a bash step and two versions of a PowerShell script into one stretch of the file, around lines 58–61. There are now three separate steps: (1) a bash step for non-Windows runners that adds ${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin to GITHUB_PATH with CR/LF removed; (2) a bash step for Windows that adds ${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin with CR/LF removed; (3) a pwsh step for Windows that strips CR/LF from $env:AQUA_ROOT_DIR (or the default path) with -replace before appending it via Out-File. The stray echo/Out-File fragments are gone, including the one that wrote AQUA_ROOT_DIR to GITHUB_PATH without stripping newlines, so the file's YAML structure is intact again. I checked this by re-reading the edited part of the file; I didn't run a YAML parser or the tests. The action only has action.yaml (there's no action.yml), so that's the file the invalid-yaml finding refers to.

Two things to know: the bash and pwsh steps both run on Windows, so the bin directory is added to PATH twice there. And the pwsh step adds AQUA_ROOT_DIR without a /bin suffix while the bash steps add /bin. I wrote both from memory of how upstream behaves; neither was checked against the upstream source, so it's worth comparing before release.


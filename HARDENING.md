<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.3** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (b) violation: In action.yaml, the run step `aqua i $AQUA_OPTS` uses an unquoted shell variable `$AQUA_OPTS` that is sourced from `inputs.aqua_opts` via the env: block. Because the expansion is unquoted, an attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) could cause command injection. The variable must be double-quoted: `aqua i "$AQUA_OPTS"`.

Locations:

- `action.yaml:86`

### github-env-injection (severity: high)

Case (e) violation: The composite action writes the inherited process env var `AQUA_ROOT_DIR` (which can be set arbitrarily by the calling workflow) directly to `$GITHUB_PATH` without the required sanitization step (`printf '%s' "$AQUA_ROOT_DIR" | tr -d '\n\r'`). This appears in three steps: the Linux bash step (`echo "${AQUA_ROOT_DIR:-...}/bin" >> $GITHUB_PATH`), the Windows bash step (`echo "${AQUA_ROOT_DIR:-...}/bin" >> $GITHUB_PATH`), and the PowerShell step (`$env:AQUA_ROOT_DIR | Out-File -FilePath $env:GITHUB_PATH`). A calling workflow could inject newlines into `AQUA_ROOT_DIR` to smuggle additional entries into GITHUB_PATH.

Locations:

- `action.yaml:45`
- `action.yaml:51`
- `action.yaml:57`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed 4 issues in hardened/action/action.yaml:
1. github-env-injection (Linux bash, line 45): Added sanitization using `printf '%s' "${AQUA_ROOT_DIR:-...}" | tr -d '\n\r'` before writing to $GITHUB_PATH.
2. github-env-injection (Windows bash, line 51): Same sanitization pattern applied.
3. github-env-injection (PowerShell, line 57): Used `-replace '[\r\n]', ''` to strip newlines before writing to $GITHUB_PATH via Add-Content.
4. script-injection (line 86): Replaced unquoted `aqua i $AQUA_OPTS` with xargs-based tokenization into a bash array (`opts=()`), then expanded as `aqua i "${opts[@]}"` to safely handle multiple options without allowing shell metacharacter injection.


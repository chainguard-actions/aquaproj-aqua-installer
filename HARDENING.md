<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.1** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In action.yaml, two composite action steps write inherited process environment variables ($AQUA_ROOT_DIR and $XDG_DATA_HOME) to $GITHUB_PATH without the required sanitization step (printf '%s' ... | tr -d '\n\r'). These env vars are not computed in the same run block — they are referenced from the calling workflow's environment, making them untrusted inputs. An attacker-controlled calling workflow could set AQUA_ROOT_DIR or XDG_DATA_HOME to a value containing newlines, injecting arbitrary entries into GITHUB_PATH.

Step 1 (Linux): `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`
Step 2 (Windows bash): `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`

Fix: sanitize before writing, e.g.:
```
safe=$(printf '%s' "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" | tr -d '\n\r')
echo "$safe" >> "$GITHUB_PATH"
```

Locations:

- `action.yaml:37`
- `action.yaml:41`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed two steps in action.yaml that wrote environment-variable-derived paths (AQUA_ROOT_DIR, XDG_DATA_HOME) directly to $GITHUB_PATH without sanitization. Both the Linux step (runner.os != 'Windows') and the Windows bash step (runner.os == 'Windows') now use `safe=$(printf '%s' "<path>" | tr -d '\n\r')` followed by `echo "$safe" >> "$GITHUB_PATH"` to strip any embedded newlines before writing, preventing GITHUB_PATH injection attacks.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the unquoted `$AQUA_OPTS` expansion in the `aqua i` step. Replaced `aqua i $AQUA_OPTS` with an xargs-based array tokenization pattern that safely splits the caller-controlled `aqua_opts` input into individual arguments while preventing shell metacharacter injection. The fix uses `printf '%s' "$AQUA_OPTS" | xargs printf '%s\0'` piped into a null-delimited read loop to build an `opts` array, then expands it as `aqua i "${opts[@]}"`. A guard on `[ -n "$AQUA_OPTS" ]` prevents xargs from emitting an empty token when the input is empty.


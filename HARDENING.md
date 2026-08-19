<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.4** was hardened automatically. 0 finding(s) were identified and resolved across 1 iteration(s).

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted $AQUA_OPTS in `aqua i $AQUA_OPTS` at line 84 of action.yaml. Since aqua_opts is a multi-token flags argument (default: "-l"), replaced the bare expansion with xargs-based array tokenization: the value is split quote-awaredly into an opts array via `printf '%s' "$AQUA_OPTS" | xargs printf '%s\0'` with a NUL-delimited read loop, then expanded as `"${opts[@]}"`. A guard `if [ -n "$AQUA_OPTS" ]` prevents xargs from emitting an empty token when the variable is empty.


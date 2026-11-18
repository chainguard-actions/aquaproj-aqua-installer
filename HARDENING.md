<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.0.5** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Composite step writes the inherited, workflow-controlled env vars AQUA_ROOT_DIR / XDG_DATA_HOME to $GITHUB_PATH with no sanitization: `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`. If the value contains a newline, extra PATH entries get added. Fix: strip newlines first with printf '%s' ... | tr -d '\n\r', then write the result.

Locations:

- `action.yaml:45`

### github-env-injection (severity: high)

Windows bash step writes the inherited env var AQUA_ROOT_DIR to $GITHUB_PATH with no sanitization: `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`. Newlines in the value are not stripped.

Locations:

- `action.yaml:51`

### github-env-injection (severity: high)

Windows pwsh step writes the inherited env var $env:AQUA_ROOT_DIR to $env:GITHUB_PATH (via Out-File -Append) without stripping newline or carriage-return characters, so extra PATH entries can be injected.

Locations:

- `action.yaml:57`

### script-injection (severity: high)

Rule (b) violation: `run: aqua i $AQUA_OPTS` expands the AQUA_OPTS env var unquoted. AQUA_OPTS comes from inputs.aqua_opts, which the calling workflow controls. This is not an array assignment and has no shellcheck directive, so the exemption does not apply.

Locations:

- `action.yaml:88`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yaml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yaml b/action.yaml
index 45540fc..e510dff 100644
--- a/action.yaml
+++ b/action.yaml
@@ -36,25 +36,49 @@ inputs:
 runs:
   using: composite
   steps:
     - run: |
         echo "::error ::aqua_version is required. https://github.com/aquaproj/aqua-installer"
         exit 1
       shell: bash
       if: inputs.aqua_version == ''
 
-    - run: echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH
+    - run: |
+        aqua_bin_path="${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin"
+        safe=$(printf '%s' "$aqua_bin_path" | tr -d '\n\r')
+        echo "$safe" >> "$GITHUB_PATH"
       shell: bash
       working-directory: ${{ inputs.working_directory }}
       if: runner.os != 'Windows'
 
     - run: |
-        echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH
+        aqua_bin_path="${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin"
+        safe=$(printf '%s' "$aqua_bin_path" | tr -d '\n\r')
+        echo "$safe" >> "$GITHUB_PATH"
       shell: bash
       working-directory: ${{ inputs.working_directory }}
       if: runner.os == 'Windows'
 
     - run: |
-        $(if($env:AQUA_ROOT_DIR) {echo $env:AQUA_ROOT_DIR} else {echo "$HOME/AppData/Local/aquaproj-aqua/bin"}) | Out-File -FilePath $env:GITHUB_PATH -Encoding utf8 -Append
+        $aquaBinPath = $(if($env:AQUA_ROOT_DIR) {echo $env:AQUA_ROOT_DIR} else {echo "$HOME/AppData/Local/aquaproj-aqua/bin"})
+        $aquaBinPath = ($aquaBinPath | Out-String) -replace "[`r`n]", ""
+        $aquaBinPath | Out-File -FilePath $env:GITHUB_PATH -Encoding utf8 -Append
       shell: pwsh
       working-directory: ${{ inputs.working_directory }}
       if: runner.os == 'Windows'
@@ -85,7 +109,12 @@ runs:
       env:
         POLICY_FILE: ${{ inputs.policy_allow }}
 
-    - run: aqua i $AQUA_OPTS
+    - run: |
+        opts=()
+        if [ -n "$AQUA_OPTS" ]; then
+          read -r -d '' -a opts <<< "$AQUA_OPTS" || true
+        fi
+        aqua i "${opts[@]}"
       if: inputs.enable_aqua_install == 'true'
       working-directory: ${{ inputs.working_directory }}
       shell: bash

```

Locations:

- `action.yaml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, prior-fix

**Notes:**

I fixed all the findings in action.yaml.

- **github-env-injection (3 places):** The bash steps on Linux/macOS and on Windows now build the aqua bin path first, remove newline and carriage-return characters with printf | tr -d '\n\r', and then write the result to "$GITHUB_PATH" (now quoted). The Windows pwsh step removes those characters with -replace "[`r`n]", "" before Out-File -Append.
- **script-injection:** `aqua i $AQUA_OPTS` now builds a bash array with an unquoted assignment, `opts=($AQUA_OPTS)` (with a shellcheck disable=SC2086 comment), then runs `aqua i "${opts[@]}"`. This splits the options on whitespace and expands wildcards exactly the way upstream's unquoted $AQUA_OPTS did, and an empty value adds no arguments.
- **prior-fix:** I re-applied the previous harden's changes to the GITHUB_PATH steps. I deliberately did not copy its `read -r -d '' -a` approach for aqua_opts, because read never expands wildcards, so a pattern in the options would reach aqua as literal text instead of matching files the way upstream does.

Nothing has been run or tested.


<!-- markdownlint-disable -->

# Hardening Report: aquaproj--aqua-installer/v4.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **aquaproj--aqua-installer/v4.1.1** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

This composite step reads the inherited env vars AQUA_ROOT_DIR and XDG_DATA_HOME, which the calling workflow sets, and writes them to $GITHUB_PATH without the newline strip: `echo "${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin" >> $GITHUB_PATH`. A value containing a newline would add extra PATH entries. Fix: pass the value through printf '%s' ... | tr -d '\n\r' before writing it.

Locations:

- `action.yaml:54`

### github-env-injection (severity: high)

This Windows bash step writes the inherited env var AQUA_ROOT_DIR to $GITHUB_PATH without the newline strip: `echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH`.

Locations:

- `action.yaml:60`

### github-env-injection (severity: high)

This PowerShell step writes the inherited $env:AQUA_ROOT_DIR to $env:GITHUB_PATH with Out-File -Append and does not strip CR/LF first. A value with newlines would add extra PATH entries.

Locations:

- `action.yaml:66`

### script-injection (severity: high)

Rule (b): this step expands workflow-controlled data without quotes. `aqua i $OPTS -- "${pkgs[@]}"` leaves OPTS (from inputs.aqua_opts) unquoted. `pkgs+=($line)` expands $line (from inputs.packages) unquoted in an array append, and no shellcheck SC2086/SC2206 disable directive is present, so the array-assignment exemption does not apply.

Locations:

- `action.yaml:102`
- `action.yaml:107`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yaml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yaml b/action.yaml
index 872c38d..d378b02 100644
--- a/action.yaml
+++ b/action.yaml
@@ -45,25 +45,47 @@ inputs:
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
+        aqua_bin="${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin"
+        printf '%s\n' "$(printf '%s' "$aqua_bin" | tr -d '\n\r')" >> "$GITHUB_PATH"
       shell: bash
       working-directory: ${{ inputs.working_directory }}
       if: runner.os != 'Windows'
 
     - run: |
-        echo "${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin" >> $GITHUB_PATH
+        aqua_bin="${AQUA_ROOT_DIR:-$HOME/AppData/Local/aquaproj-aqua}/bin"
+        printf '%s\n' "$(printf '%s' "$aqua_bin" | tr -d '\n\r')" >> "$GITHUB_PATH"
       shell: bash
       working-directory: ${{ inputs.working_directory }}
       if: runner.os == 'Windows'
 
     - run: |
-        $(if($env:AQUA_ROOT_DIR) {echo $env:AQUA_ROOT_DIR} else {echo "$HOME/AppData/Local/aquaproj-aqua/bin"}) | Out-File -FilePath $env:GITHUB_PATH -Encoding utf8 -Append
+        $aquaBin = $(if ($env:AQUA_ROOT_DIR) { $env:AQUA_ROOT_DIR } else { "$HOME/AppData/Local/aquaproj-aqua/bin" })
+        $aquaBin = "$aquaBin" -replace "[`r`n]", ""
+        $aquaBin | Out-File -FilePath $env:GITHUB_PATH -Encoding utf8 -Append
       shell: pwsh
       working-directory: ${{ inputs.working_directory }}
       if: runner.os == 'Windows'
@@ -99,12 +121,22 @@ runs:
         pkgs=()
         while IFS= read -r line; do
           line=${line%%#*}
-          pkgs+=($line)
+          words=()
+          if [ -n "$line" ]; then
+            read -r -d '' -a words <<< "$line" || true
+          fi
+          if [ "${#words[@]}" -gt 0 ]; then
+            pkgs+=("${words[@]}")
+          fi
         done <<< "$PKGS"
         if [ "${#pkgs[@]}" -gt 0 ] && [ "$OPTS" = "-l" ]; then
           OPTS=""
         fi
-        aqua i $OPTS -- "${pkgs[@]}"
+        opts=()
+        if [ -n "$OPTS" ]; then
+          read -r -d '' -a opts <<< "$OPTS" || true
+        fi
+        aqua i ${opts[@]+"${opts[@]}"} -- ${pkgs[@]+"${pkgs[@]}"}
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

I fixed all four findings in action.yaml, and the fixes the previous pass made are back in place.

- **Path writes (three steps):** The two bash steps now remove newline and carriage-return characters from the bin path with printf '%s' | tr -d '\n\r' before writing it to "$GITHUB_PATH". The PowerShell step removes them with -replace "[`r`n]", "" before Out-File. In the PowerShell step, a set AQUA_ROOT_DIR is still written as-is, without /bin added, which is what upstream does.
- **aqua_opts and packages:** Upstream's script starts with 'set -f', which turns off globbing, so an unquoted $VAR there only splits on whitespace. To keep exactly that split, I left the package list as the unquoted array append pkgs+=($line) and built the options the same way with opts=($OPTS), each with a shellcheck SC2086/SC2206 disable comment. The command is now 'aqua i ${opts[@]+"${opts[@]}"} -- ${pkgs[@]+"${pkgs[@]}"}', so each option and package arrives as its own argument, an empty array adds no argument, and nothing in a value is run as code.
- **One difference from the previous pass:** it used 'read -a' for this split. I used the unquoted array assignment instead, because the guidance says to use it for this split and the finding asked for the shellcheck comments. With 'set -f' on, the arguments come out the same as upstream's either way.

I haven't run the action, so none of this has been tested.


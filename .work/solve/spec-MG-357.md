# MG-357: Fix gather exit code propagation and upload coordination

## Problem

1. **`gatherCommand` masks exit codes**: No `pipefail`, and `$?` reflects `tee`'s exit code, not the gather binary's. The `fi | tee -a` at the end also masks exit codes.
2. **`uploadCommand` only checks process presence**: Uses `pgrep -a gather` to detect termination, but never checks whether gather succeeded.
3. **No success gate**: Upload/obfuscation proceeds unconditionally after gather terminates, even on failure or crash.

## Solution

### Mechanism: Exit code marker file

Gather container writes its exit code to `/must-gather/.gather-exit-code` on the shared volume. Upload container reads this file after detecting gather termination to decide whether to proceed.

### Changes

#### 1. Fix `gatherCommand` (template.go)
- Add `set -o pipefail` so `$?` reflects the gather binary's exit code through `tee`
- Remove the broken `fi | tee -a` piping
- Write the exit code to `/must-gather/.gather-exit-code`
- Use `(exit $status)` to set `$?` without exiting the main shell (so `obfuscateChownSuffix` still runs)
- Timeout (124/137) behavior preserved: writes "0" and exits 0

#### 2. Fix `uploadCommand` (template.go)
- After the pgrep wait loop, check the marker file:
  - File exists with "0" -> proceed with upload
  - File exists with non-zero -> skip upload, exit 1
  - File missing (crash) -> skip upload, exit 1
- `uploadCommandDirect` (obfuscate.source mode) unchanged

#### 3. Custom command wrapping (template.go)
- Always wrap custom commands in bash to write the marker file
- For obfuscation: marker write added between `"$@"` and `obfuscateChownSuffix`
- For non-obfuscation: wrap in bash with marker write

#### 4. Tests (template_test.go)
- Update existing tests for new command format (pipefail prefix, wrapped custom commands)
- Add dedicated test for gather exit code marker and upload gate logic

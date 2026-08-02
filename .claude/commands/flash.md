Watch the most recent GitHub Actions build for the current branch, wait for it to complete, download the firmware, then flash left and right halves automatically by waiting for the NICENANO bootloader volume.

Steps:
1. Run `gh run list --branch $(git rev-parse --abbrev-ref HEAD) --limit 1 --json databaseId,status,conclusion,displayTitle` to get the latest run ID and status for the current branch.
2. If the run is `in_progress` or `queued`, run `gh run watch <id>` (with a 5 minute timeout) to wait for completion.
3. Check the final conclusion. If it is not `success`, report the failure and stop.
4. Remove any previous download: `rm -rf ~/Downloads/zmk-flash`
5. Download artifacts: `gh run download <id> --dir ~/Downloads/zmk-flash`
6. Unzip any zip files: `find ~/Downloads/zmk-flash -name "*.zip" -exec unzip -o {} -d ~/Downloads/zmk-flash/firmware \;`
7. Find the left and right .uf2 file paths:
   - LEFT: the file matching `*corne_left*.uf2`
   - RIGHT: the file matching `*corne_right*.uf2`
8. Detect the OS to determine how to flash:
   - If `uname -r` contains `microsoft` (WSL), use the WSL flash method below.
   - Otherwise (macOS), use `/Volumes/NICENANO` as the mount path.

   **WSL flash method** (use in steps 9 and 10):
   WSL2 does not auto-mount USB drives, so use PowerShell's Copy-Item directly.
   Convert the WSL file path to a Windows path with `wslpath -w`, then poll until
   the NICENANO volume appears and copy via PowerShell:
   ```bash
   PS=/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe
   WIN_SRC=$(wslpath -w "<uf2_path>")
   DRIVE=""
   while [ -z "$DRIVE" ]; do
     DRIVE=$($PS -Command "Get-Volume -FileSystemLabel NICENANO -ErrorAction SilentlyContinue | Select-Object -ExpandProperty DriveLetter" | tr -d '\r\n')
     sleep 1
   done
   $PS -Command "Copy-Item -Path '$WIN_SRC' -Destination '${DRIVE}:\'"
   while [ -n "$($PS -Command "Get-Volume -FileSystemLabel NICENANO -ErrorAction SilentlyContinue | Select-Object -ExpandProperty DriveLetter" | tr -d '\r\n')" ]; do sleep 1; done
   ```

9. Flash left half:
   - Output this message in bold so the user sees it before approving:
     **ACTION REQUIRED: Double-click reset on the LEFT half to enter bootloader mode, then approve the next command.**
   - In a SINGLE Bash call (so it only needs one approval), wait for the volume, copy, and wait for it to disappear.
   - macOS: `until [ -d /Volumes/NICENANO ]; do sleep 1; done && cp "<left_uf2>" /Volumes/NICENANO/ ; while [ -d /Volumes/NICENANO ]; do sleep 1; done`
   - WSL: use the WSL flash method from step 8 with `<left_uf2>` path.
   - Tell the user the left half is flashed.
10. Flash right half:
   - Output this message in bold so the user sees it before approving:
     **ACTION REQUIRED: Double-click reset on the RIGHT half to enter bootloader mode, then approve the next command.**
   - In a SINGLE Bash call (so it only needs one approval), wait for the volume, copy, and wait for it to disappear.
   - macOS: `until [ -d /Volumes/NICENANO ]; do sleep 1; done && cp "<right_uf2>" /Volumes/NICENANO/ ; while [ -d /Volumes/NICENANO ]; do sleep 1; done`
   - WSL: use the WSL flash method from step 8 with `<right_uf2>` path.
   - Tell the user the right half is flashed.
11. Report success: both halves flashed. Note: this repo's `build.yaml` does not build a `settings_reset` artifact. If you need to wipe BLE pairing, add a `settings_reset` shield build to `build.yaml` (see the keyball44 config for an example) and flash the resulting `settings_reset-nice_nano_v2-zmk.uf2`.

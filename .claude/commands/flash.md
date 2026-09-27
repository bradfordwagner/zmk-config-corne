Watch the most recent GitHub Actions build for the current branch, wait for it to complete, download the firmware, then flash left and right halves automatically by waiting for the NICENANO bootloader volume.

Steps:
1. Run `gh run list --branch $(git rev-parse --abbrev-ref HEAD) --limit 1 --json databaseId,status,conclusion,displayTitle` to get the latest run ID and status for the current branch.
2. If the run is `in_progress` or `queued`, run `gh run watch <id>` (with a 5 minute timeout) to wait for completion.
3. Check the final conclusion. If it is not `success`, report the failure and stop.
4. Remove any previous download: `rm -rf ~/Downloads/zmk-flash`
5. Download artifacts: `gh run download <id> --dir ~/Downloads/zmk-flash`
6. Unzip any zip files: `find ~/Downloads/zmk-flash -name "*.zip" -exec unzip -o {} -d ~/Downloads/zmk-flash/firmware \;`
7. Find the left and right .uf2 file paths:
   - LEFT: the file matching `*corne_left*.uf2`, excluding `*with_studio*`
     (build.yaml also produces `corne_left_with_studio`; use that one instead
     only if the user asks for ZMK Studio)
   - RIGHT: the file matching `*corne_right*.uf2`
8. Detect the OS to determine how to flash:
   - If `uname -r` contains `microsoft` (WSL), use the WSL flash method below.
   - Otherwise (macOS), use `/Volumes/NICENANO` as the mount path.

   **macOS notes (apply in steps 9 and 10):**
   - Recent macOS blocks writes to removable USB volumes until the terminal app
     running Claude Code is granted access under System Settings → Privacy &
     Security → Files and Folders (or Full Disk Access). If `cp` fails with
     `Permission denied` (EACCES) on the FIRST attempt, tell the user to grant
     that permission, restart the terminal, and re-run.
   - Even with permission granted, `cp` can hit `Permission denied` from a race
     when the volume dir appears before macOS finishes mounting it read-write, so
     wait with `sleep 2` after the volume appears and retry a few times.
   - A `could not copy extended attributes ... Device not configured` (or
     `Input/output error`) is the SUCCESS signature: the uf2 data was written and
     the board rebooted/unmounted before metadata copied. Treat it as flashed.

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
   - macOS: `until [ -d /Volumes/NICENANO ]; do sleep 1; done; sleep 2; for i in 1 2 3; do cp "<left_uf2>" /Volumes/NICENANO/ 2>/tmp/cperr && break; grep -q "Device not configured\|Input/output" /tmp/cperr && break; sleep 2; [ -d /Volumes/NICENANO ] || break; done; while [ -d /Volumes/NICENANO ]; do sleep 1; done` (a `Device not configured`/`Input/output` error means success — see macOS notes)
   - WSL: use the WSL flash method from step 8 with `<left_uf2>` path.
   - Tell the user the left half is flashed.
10. Flash right half:
   - Output this message in bold so the user sees it before approving:
     **ACTION REQUIRED: Double-click reset on the RIGHT half to enter bootloader mode, then approve the next command.**
   - In a SINGLE Bash call (so it only needs one approval), wait for the volume, copy, and wait for it to disappear.
   - macOS: `until [ -d /Volumes/NICENANO ]; do sleep 1; done; sleep 2; for i in 1 2 3; do cp "<right_uf2>" /Volumes/NICENANO/ 2>/tmp/cperr && break; grep -q "Device not configured\|Input/output" /tmp/cperr && break; sleep 2; [ -d /Volumes/NICENANO ] || break; done; while [ -d /Volumes/NICENANO ]; do sleep 1; done` (a `Device not configured`/`Input/output` error means success — see macOS notes)
   - WSL: use the WSL flash method from step 8 with `<right_uf2>` path.
   - Tell the user the right half is flashed.
11. Report success: both halves flashed. Remind the user that `settings_reset-nice_nano_v2-zmk.uf2` is available in ~/Downloads/zmk-flash/firmware/ if they need to wipe BLE pairing.

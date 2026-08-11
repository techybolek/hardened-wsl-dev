---
description: Pull the live wsl-dev firewall config into the Windows master copies (distro → master). Diff first, never overwrite blind.
---

# /sync-from-wsl

Direction: **live `Ubuntu-24.04` distro → Windows master copies in `C:\work\WSL`.**
This is the *pull*. It never writes to the distro. If you want the opposite, that is `/sync-to-wsl`.

## Files

| Live (source of truth for this command) | Master (destination)               |
| --------------------------------------- | ---------------------------------- |
| `/usr/local/sbin/allowed-domains.list`  | `C:\work\WSL\allowed-domains.list` |
| `/usr/local/sbin/allowed-ips.list`      | `C:\work\WSL\allowed-ips.list`     |
| `/usr/local/sbin/init-firewall.sh`      | `C:\work\WSL\init-firewall.sh`     |

`$ARGUMENTS` may name a subset of files; with no arguments, handle all three.

## Steps

1. **Check the master worktree is clean** for these three paths:
   ```powershell
   git -C C:\work\WSL status --short -- allowed-domains.list allowed-ips.list init-firewall.sh
   ```
   If any are modified, **say so and stop for confirmation** — pulling would discard uncommitted Windows-side edits, which is the same class of mistake in the opposite direction. Offer to commit or stash first.

2. **Pull the live copies to scratch** (never straight over the masters). PowerShell tool only — the Bash tool mangles `wsl.exe` args.
   ```powershell
   . C:\work\wsl\wsl-transfer.ps1
   Copy-FromDistro /usr/local/sbin/allowed-domains.list $env:TEMP\live-allowed-domains.list
   Copy-FromDistro /usr/local/sbin/allowed-ips.list     $env:TEMP\live-allowed-ips.list
   Copy-FromDistro /usr/local/sbin/init-firewall.sh     $env:TEMP\live-init-firewall.sh
   ```
   Do not use the `\\wsl.localhost\...` UNC share — it hangs unpredictably. If a call runs past ~15s, kill it (`TaskStop`) and retry.

3. **Diff each pair and show the user the full diff** before touching anything:
   ```powershell
   git -C C:\work\WSL diff --no-index -- .\allowed-domains.list $env:TEMP\live-allowed-domains.list
   ```
   Report per file: identical / live-ahead / divergent. **Do not summarize away the content** — the operator needs to see what their in-distro edits actually were.

4. **Stop and confirm** if anything is divergent (both sides changed independently). Present the conflict; let the user decide what wins. Never auto-merge.

5. **Copy the live files over the masters** (only after the diff is shown, and after confirmation if step 4 triggered). Preserve the repo's line-ending convention — write LF; git handles the rest.

6. **Report** what changed, then ask whether to commit. Suggest a conventional-commit message describing the entries/behavior pulled in, e.g. `chore(firewall): pull in-distro allow-list edits for <hosts>`.

## Don't

- Don't push anything to the distro. Zero `install`, zero `Copy-ToDistro`, zero `init-firewall.sh` reruns. This command is read-only against `Ubuntu-24.04`.
- Don't overwrite a modified master without showing the diff and getting a yes.
- Don't assume the master is authoritative. Firewall edits are frequently made live in-distro first; that is the normal workflow this command exists to capture.
- Don't treat a bare "sync" from the user as license to run this or its mirror. Ask which direction, or just diff and report.

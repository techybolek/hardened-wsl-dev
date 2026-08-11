---
description: Push the Windows master firewall config to the live wsl-dev distro and reload (master → distro). Confirm the live copy isn't ahead first.
---

# /sync-to-wsl

Direction: **Windows master copies in `C:\work\WSL` → live `Ubuntu-24.04` distro.**
This is the *push*, and it is **destructive to the live files** — it overwrites `/usr/local/sbin/*` and rebuilds the ipset. The mirror is `/sync-from-wsl`.

## Files

| Master (source of truth for this command) | Live (destination, overwritten)        |
| ----------------------------------------- | -------------------------------------- |
| `C:\work\WSL\allowed-domains.list`        | `/usr/local/sbin/allowed-domains.list` |
| `C:\work\WSL\allowed-ips.list`            | `/usr/local/sbin/allowed-ips.list`     |
| `C:\work\WSL\init-firewall.sh`            | `/usr/local/sbin/init-firewall.sh`     |

`$ARGUMENTS` may name a subset of files; with no arguments, handle all three.

## Steps

1. **Check whether the live copy is ahead — this step is mandatory and comes first.**
   Firewall edits are frequently made live in-distro. Pushing without checking silently destroys them, with no undo: the overwrite is byte-level, `/tmp` gets clobbered too, and `.vhdx` snapshots are usually weeks stale.
   ```powershell
   . C:\work\wsl\wsl-transfer.ps1
   Copy-FromDistro /usr/local/sbin/allowed-domains.list $env:TEMP\live-allowed-domains.list
   Copy-FromDistro /usr/local/sbin/allowed-ips.list     $env:TEMP\live-allowed-ips.list
   Copy-FromDistro /usr/local/sbin/init-firewall.sh     $env:TEMP\live-init-firewall.sh
   git -C C:\work\WSL diff --no-index -- .\allowed-domains.list $env:TEMP\live-allowed-domains.list
   ```
   - Identical → proceed.
   - **Live has content the master lacks → STOP.** Show the diff and ask. The likely correct action is `/sync-from-wsl` (or a merge), not this command.

2. **Push.** PowerShell tool only — the Bash tool mangles `wsl.exe` args. Never the `\\wsl.localhost` UNC share (hangs with no error; kill past ~15s with `TaskStop` and retry).
   ```powershell
   Copy-ToDistro C:\work\wsl\allowed-domains.list /tmp/allowed-domains.list
   wsl.exe -d Ubuntu-24.04 -u root -- install -m 644 -o root -g root /tmp/allowed-domains.list /usr/local/sbin/allowed-domains.list
   ```
   Use `-m 755` for `init-firewall.sh`. Verify with `md5sum` on the installed paths.

3. **Reload.** Rebuilds the ipset in place; no reboot.
   ```powershell
   wsl.exe -d Ubuntu-24.04 -u root -- /usr/local/sbin/init-firewall.sh
   ```
   Expect `[init-firewall] Firewall active (default OUTPUT=DROP, allow-list applied)` and note the entry count.

4. **Probe any newly allowed hosts** — see the `update-firewall` skill, step 5, for the base64 probe pattern. `000` at ~8s means still blocked (CDN-fronted → needs a `/24` in `allowed-ips.list`).

5. **Report** the entry-count delta and probe results.

## Don't

- Don't run step 2 before step 1 has been done and shown. That ordering is the whole point of this command.
- Don't treat a bare "sync" from the user as license to run this. Ask which direction, or just diff and report.
- Don't skip the probe — a clean reload with a still-unreachable host is the most common failure mode.

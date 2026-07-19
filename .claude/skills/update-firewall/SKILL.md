---
name: update-firewall
description: Add (or remove) hosts/CIDRs in the wsl-dev outbound allow-list and reload the in-distro firewall without rebooting. Use when the user asks to whitelist a domain, allow a host, open a CDN /24, unblock a URL, or update allowed-domains.list / allowed-ips.list.
---

# update-firewall

Edit the Windows-side master allow-list, push to the live `Ubuntu-24.04` distro, reload `init-firewall.sh`, and probe that the new hosts actually reach origin. No distro reboot required.

## Files (master copies on Windows, live copies in-distro)

| Master (this repo)                  | Live (inside distro)                      | When to use                                                                 |
| ----------------------------------- | ----------------------------------------- | --------------------------------------------------------------------------- |
| `C:\work\wsl\allowed-domains.list`  | `/usr/local/sbin/allowed-domains.list`    | Hostnames resolved at firewall-init time (one A-record → ipset entry).      |
| `C:\work\wsl\allowed-ips.list`      | `/usr/local/sbin/allowed-ips.list`        | Direct IPs / `/24` CIDRs for CDN-fronted hosts whose edge pool rotates.     |
| `C:\work\wsl\init-firewall.sh`      | `/usr/local/sbin/init-firewall.sh`        | Live-fetched ranges (GitHub meta, AWS CloudFront/GA, Google goog.json).     |

## Steps

1. **Decide which list each entry goes in.**
   - Stable A record (api.example.com, docs site) → `allowed-domains.list`.
   - CDN-fronted (symptom: ~135s curl timeout despite hostname listed) → `/24` in `allowed-ips.list`. See README "CDN-CIDR caveat".
   - Already covered by live-fetched ranges? GitHub web/API/git/packages/actions come from `api.github.com/meta`. Google-owned hosts (web.dev, angular.dev, *.googleusercontent, storage.googleapis.com) usually come from `goog.json`. Listing the hostname is still useful as documentation but is redundant for resolution.

2. **Edit the master file(s).** Group additions under a comment header explaining the use case ("Research / docs for X"). One entry per line; `#` for comments.

3. **Push to the distro.** PowerShell tool only — never the Bash tool (Git Bash MSYS mangles `wsl.exe` args and crashes).

   **Use `wsl-transfer.ps1` (preferred — do not use the `\\wsl.localhost` UNC share, see Gotchas):**
   ```powershell
   . C:\work\wsl\wsl-transfer.ps1
   Copy-ToDistro C:\work\wsl\allowed-domains.list /tmp/allowed-domains.list
   wsl.exe -d Ubuntu-24.04 -u root -- install -m 644 -o root -g root /tmp/allowed-domains.list /usr/local/sbin/allowed-domains.list
   ```
   `Copy-ToDistro` routes bytes through `wsl -- base64`, which works regardless of share/session state. The UNC path (`\\wsl.localhost\...`, or the older `Push-FileToDistro` helper) can hang indefinitely with no error — it is flaky per-session and not a reliable fallback. If a `wsl.exe`/UNC command runs past ~15s, kill it (`TaskStop`) rather than waiting, and redo the push with `Copy-ToDistro`.

4. **Reload the firewall.** Rebuilds the ipset in-place; no reboot.
   ```powershell
   wsl.exe -d Ubuntu-24.04 -u root -- /usr/local/sbin/init-firewall.sh
   ```
   Expect last line: `[init-firewall] Firewall active (default OUTPUT=DROP, allow-list applied)`. Note the entry count; should grow by roughly (added domains × A records per domain).

5. **Probe each new host** to catch CDN misses early. Do NOT inline via `wsl -- bash -c "for h in ...; do ... done"` (PowerShell eats the inner quotes) and do NOT write the script via the UNC share (flaky, see step 3). Base64-encode it inline instead:
   ```powershell
   $probe = @"
   for h in host1 host2 host3; do
     printf '%-25s ' "`$h"
     curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' --max-time 8 "https://`$h/" || true
   done
   "@
   $b64 = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($probe.Replace("`r`n","`n")))
   wsl.exe -d Ubuntu-24.04 -- bash -c "echo $b64 | base64 -d > /tmp/probe.sh && bash /tmp/probe.sh"
   ```
   - Any HTTP code (200/301/302/403) returned in <1s → origin reachable, firewall is fine.
   - `000` with ~8s total → blocked. The host is CDN-fronted; find its edge `/24` and add to `allowed-ips.list`, then redo steps 3–5.

6. **(Optional) Snapshot** after a meaningful firewall change:
   ```powershell
   wsl --export Ubuntu-24.04 C:\work\wsl\backups\ubuntu-24.04-clean-<date>.vhdx --vhd
   ```
   Per CLAUDE.md, the distro is the deliverable — meaningful allow-list changes should land in a fresh `.vhdx`.

## Gotchas (do not repeat)

- **The `\\wsl.localhost\<distro>\...` UNC share hangs unpredictably — do not use it, even as a "quick" one-off.** It works in one PowerShell session and hangs with no error in another (per-session/elevation/cache flakiness, not reproducible on demand). A hung `wsl.exe`/`Copy-Item` call against it can block a tool call past its timeout with zero output. Use `Copy-ToDistro`/`Copy-FromDistro` from `C:\work\wsl\wsl-transfer.ps1` instead — it routes bytes through `wsl -- base64`, which doesn't depend on the share. This supersedes `Push-FileToDistro` in `setup-dev-distro.ps1` (UNC-based, same hang risk).
- **Bash tool + `wsl.exe` = crash.** Git Bash / MSYS path-translates the `--` and `\\wsl.localhost\...` args and dies with `add_item "\??\C:\..." failed, errno 1`. Always shell out to `wsl.exe` from the **PowerShell** tool.
- **`Out-File -Encoding utf8` writes a BOM** in PS5.1. `wsl-transfer.ps1` already handles this — no manual BOM-stripping needed when using `Copy-ToDistro`.
- **Inline `wsl -- bash -c "..."` mangles quotes.** Base64-encode the script and decode it on the far side (see step 5), or use `Copy-ToDistro` + `wsl -- bash /tmp/foo.sh`.
- **Editing only the Windows master file changes nothing.** The live distro reads `/usr/local/sbin/allowed-*.list` — you must push + reload.
- **`init-firewall.sh` flushes and rebuilds the ipset** — there is no "add one rule" mode. Always rerun the whole script.
- **CDN /24 churn:** if a host worked yesterday and times out today, its edge IP rotated outside the pinned /24. Probe with `dig +short` from inside the distro, then widen the CIDR.

## Don't

- Don't add a domain to the list and stop — without step 4 (reload) it has zero effect.
- Don't push the *live* in-distro file back to the Windows master — flow is one-way (master → live).
- Don't pin individual A-records for CDN-fronted hosts (Fastly, Akamai, Cloudfront, Azure Front Door, Edgio). Always use the /24.
- Don't skip the probe. A reload that "succeeded" but the host still 135s-times-out is the most common failure mode and the easiest to catch immediately.
- Don't use the `\\wsl.localhost\...` UNC share, or wait out a `wsl.exe` call that's hanging — kill it (`TaskStop`) and retry with `Copy-ToDistro`.

## Related

- `memory/wsl-transfer-utility.md` — `wsl-transfer.ps1` (`Copy-ToDistro`/`Copy-FromDistro`), the preferred transfer method; supersedes the UNC pattern below.
- `memory/firewall-cdn-cidrs.md` — full CDN /24 playbook + symptom catalogue.
- `memory/powershell-wsl-file-transfer.md` — background on why raw `wsl -- bash -s`/`-c` pipes mangle scripts (CRLF/BOM/quoting); superseded by `wsl-transfer.ps1` for file transfer specifically.
- `memory/wsl-dev-firewall-reload.md` — short-form version of this skill (kept for cross-conversation recall).
- `README.md` § "Outbound firewall" — architecture and the live-fetched ranges (GitHub meta, AWS, Google).

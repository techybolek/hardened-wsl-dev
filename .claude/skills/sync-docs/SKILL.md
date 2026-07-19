---
name: sync-docs
description: Audit the live wsl-dev (Ubuntu-24.04) distro and update README.md + setup-dev-distro.ps1 to reflect its current state. Use when the user asks to sync docs, update the README/blueprint, or check for drift between docs and distro.
---

# sync-docs

Make `README.md` and `setup-dev-distro.ps1` match what the live `Ubuntu-24.04` distro is actually doing.

## Steps

1. **Inspect the distro.** One command, broad sweep:
   ```bash
   wsl -d Ubuntu-24.04 -- bash -lc '
   echo "## wsl.conf"; sudo cat /etc/wsl.conf
   echo "## installed (manual)"; apt-mark showmanual | sort
   echo "## profile.d"; ls /etc/profile.d/
   echo "## sudoers.d"; sudo ls /etc/sudoers.d/
   echo "## sshd"; sudo sshd -T 2>/dev/null | grep -E "^(port|listenaddress|passwordauth|pubkey|permitroot|allowusers)" | sort
   echo "## claude hooks"; ls ~/.claude/hooks/ 2>/dev/null; cat ~/.claude/settings.json 2>/dev/null
   echo "## versions"; node -v; ~/.bun/bin/bun -v; ~/.local/bin/claude --version 2>/dev/null; docker version --format "{{.Server.Version}}"
   echo "## firewall"; systemctl is-active init-firewall
   '
   ```

2. **Diff against docs.** Read `README.md`, `setup-dev-distro.ps1`, and `init-firewall.sh`. Compare the master `init-firewall.sh` to the live one at `\\wsl.localhost\Ubuntu-24.04\usr\local\sbin\init-firewall.sh`.

3. **List findings to the user.** Group by category (packages, config, hooks, firewall, versions). For each, say whether it looks intentional (update docs) or like drift (revert distro). Ask before applying anything ambiguous.

4. **Apply updates to both files.** A change to the blueprint without a matching README change means the operator manual lies; a change to README without blueprint means `-Mode Fresh` won't reproduce it. Always touch both when relevant.

5. **Syntax-check the blueprint** after editing:
   ```powershell
   $e=$null;[System.Management.Automation.Language.Parser]::ParseFile('C:\work\WSL\setup-dev-distro.ps1',[ref]$null,[ref]$e)|Out-Null;if($e){$e}else{"OK"}
   ```

## Don't

- Don't modify the live distro to match docs — docs follow the distro.
- Don't touch `memory/` — that's separate.
- Don't commit. User runs git themselves.

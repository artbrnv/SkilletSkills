---
name: autostart-checker
description: >
  Audit, list, and manage autostart / startup programs on any operating system
  (macOS, Windows, Linux). Use this skill whenever the user asks about startup
  programs, login items, boot time, slow startup, background processes, or
  anything that runs automatically when the computer starts. Trigger phrases
  include: "what's running at startup", "why is my computer slow to boot",
  "disable startup programs", "check my login items", "what launches on boot",
  "manage autostart", "startup apps", "scheduled tasks at boot", "services at
  startup", or "what's running in the background". Always use this skill — it
  produces OS-specific, runnable audit and management scripts.
---

# Autostart Checker

Audit and manage everything that runs automatically on boot/login across
**macOS**, **Windows**, and **Linux**. Produces a full inventory and safe
disable/enable scripts.

---

## Autostart Locations by OS

### macOS

| Location | What lives there |
|---|---|
| `~/Library/LaunchAgents/` | Per-user daemons (plist files) |
| `/Library/LaunchAgents/` | System-wide user-level daemons |
| `/Library/LaunchDaemons/` | System-wide root daemons |
| `/System/Library/LaunchAgents/` | Apple-owned agents (don't touch) |
| System Preferences → Login Items | GUI apps set to open at login |
| `/Library/StartupItems/` | Legacy (pre-launchd), rare |

### Windows

| Location | What lives there |
|---|---|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Per-user registry run keys |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` | Machine-wide registry run keys |
| `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup` | Per-user startup folder |
| `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp` | All-users startup folder |
| Task Scheduler | Scheduled tasks with boot/login triggers |
| Services (`services.msc`) | Windows services set to Automatic |

### Linux

| Location | What lives there |
|---|---|
| `~/.config/autostart/` | XDG autostart (GUI desktop apps) |
| `/etc/xdg/autostart/` | System-wide XDG autostart |
| `~/.bashrc`, `~/.zshrc`, `~/.profile` | Shell init (programs launched in terminals) |
| `systemctl --user list-units` | User systemd services |
| `systemctl list-units` | System systemd services |
| `/etc/init.d/` | SysV init scripts (older distros) |
| `crontab -l` + `/etc/cron.d/` | Cron jobs with `@reboot` |
| `/etc/rc.local` | Legacy boot script |

---

## Audit Scripts (Read-Only)

### macOS audit (Bash)

```bash
#!/usr/bin/env bash
# audit_autostart_mac.sh — read only

echo "=== macOS Autostart Audit ==="
echo ""

echo "--- Launch Agents (user) ---"
ls ~/Library/LaunchAgents/ 2>/dev/null || echo "(none)"

echo ""
echo "--- Launch Agents (system-wide) ---"
ls /Library/LaunchAgents/ 2>/dev/null || echo "(none)"

echo ""
echo "--- Launch Daemons (system-wide, runs as root) ---"
ls /Library/LaunchDaemons/ 2>/dev/null || echo "(none)"

echo ""
echo "--- Login Items (GUI apps) ---"
osascript -e 'tell application "System Events" to get the name of every login item'

echo ""
echo "--- Currently loaded user agents ---"
launchctl list | grep -v "^-" | sort
```

### Windows audit (PowerShell)

```powershell
# audit_autostart_win.ps1 — read only

Write-Host "=== Windows Autostart Audit ===" -ForegroundColor Cyan

Write-Host "`n--- Registry: HKCU Run ---"
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" |
  Select-Object * -ExcludeProperty PS* | Format-List

Write-Host "`n--- Registry: HKLM Run ---"
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run" |
  Select-Object * -ExcludeProperty PS* | Format-List

Write-Host "`n--- Startup Folder (user) ---"
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" -ErrorAction SilentlyContinue |
  Format-Table Name, LastWriteTime -AutoSize

Write-Host "`n--- Startup Folder (all users) ---"
Get-ChildItem "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp" -ErrorAction SilentlyContinue |
  Format-Table Name, LastWriteTime -AutoSize

Write-Host "`n--- Scheduled Tasks with boot/logon triggers ---"
Get-ScheduledTask | Where-Object {
  $_.Triggers | Where-Object { $_ -is [Microsoft.Management.Infrastructure.CimInstance] -and
    ($_.CimClass.CimClassName -match 'Boot|Logon') }
} | Format-Table TaskName, TaskPath, State -AutoSize

Write-Host "`n--- Services set to Automatic ---"
Get-Service | Where-Object { $_.StartType -eq 'Automatic' } |
  Sort-Object Status -Descending |
  Format-Table Name, DisplayName, Status -AutoSize
```

### Linux audit (Bash)

```bash
#!/usr/bin/env bash
# audit_autostart_linux.sh — read only

echo "=== Linux Autostart Audit ==="
echo ""

echo "--- XDG Autostart (user) ---"
ls ~/.config/autostart/ 2>/dev/null || echo "(none)"

echo ""
echo "--- XDG Autostart (system) ---"
ls /etc/xdg/autostart/ 2>/dev/null || echo "(none)"

echo ""
echo "--- Systemd user services (enabled) ---"
systemctl --user list-unit-files --state=enabled 2>/dev/null || echo "(systemd user not available)"

echo ""
echo "--- Systemd system services (enabled, non-static) ---"
systemctl list-unit-files --state=enabled --type=service 2>/dev/null | head -40

echo ""
echo "--- Cron @reboot jobs (current user) ---"
crontab -l 2>/dev/null | grep '@reboot' || echo "(none)"

echo ""
echo "--- /etc/rc.local ---"
cat /etc/rc.local 2>/dev/null || echo "(not present)"

echo ""
echo "--- Shell init files (commands that launch processes) ---"
for f in ~/.bashrc ~/.zshrc ~/.profile ~/.bash_profile; do
  [[ -f "$f" ]] && echo "  $f:" && grep -E '^\s*[^#].*&\s*$|nohup|exec' "$f" || true
done
```

---

## Disable / Enable Scripts

Only generate these after the user has reviewed the audit and named specific
entries to change. Always back up before disabling.

### macOS — disable a Launch Agent

```bash
# disable_agent_mac.sh
# Usage: bash disable_agent_mac.sh com.example.thing

LABEL="${1:?Provide the plist label, e.g. com.example.thing}"
PLIST=$(find ~/Library/LaunchAgents /Library/LaunchAgents -name "${LABEL}.plist" 2>/dev/null | head -1)

if [[ -z "$PLIST" ]]; then
  echo "Plist not found for: $LABEL"
  exit 1
fi

echo "Unloading: $PLIST"
launchctl unload -w "$PLIST"
echo "Done. To re-enable: launchctl load -w \"$PLIST\""
```

### Windows — disable a registry run entry

```powershell
# disable_runkey_win.ps1
# Usage: .\disable_runkey_win.ps1 -Name "Spotify"
param([string]$Name = $(throw "Provide -Name <entry name>"))

$key = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
$val = (Get-ItemProperty $key).$Name

if (-not $val) { Write-Host "Entry not found: $Name"; exit 1 }

# Back up before removing
$backup = "$env:USERPROFILE\runkey_backup_$(Get-Date -Format 'yyyyMMdd_HHmmss').reg"
reg export "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" $backup
Write-Host "Backup saved: $backup"

Remove-ItemProperty -Path $key -Name $Name
Write-Host "Disabled: $Name"
Write-Host "To restore: reg import `"$backup`""
```

### Linux — disable a systemd user service

```bash
# disable_service_linux.sh
# Usage: bash disable_service_linux.sh <service-name>

SERVICE="${1:?Provide service name, e.g. dropbox}"
systemctl --user disable "$SERVICE"
systemctl --user stop "$SERVICE"
echo "Disabled: $SERVICE"
echo "To re-enable: systemctl --user enable $SERVICE && systemctl --user start $SERVICE"
```

### Linux — disable an XDG autostart entry

```bash
# disable_xdg_linux.sh
# Usage: bash disable_xdg_linux.sh <app-name>.desktop

ENTRY="${1:?Provide .desktop filename}"
FILE=~/.config/autostart/"$ENTRY"

if [[ ! -f "$FILE" ]]; then
  # Copy from system dir so we can override it
  cp /etc/xdg/autostart/"$ENTRY" "$FILE" 2>/dev/null || { echo "Not found"; exit 1; }
fi

echo "Hidden=true" >> "$FILE"
echo "Disabled: $ENTRY (Hidden=true added)"
```

---

## Risk Levels

When presenting findings, label each entry with a risk level:

| Label | Meaning |
|---|---|
| ✅ Safe to disable | Third-party app helpers, updaters, chat apps |
| ⚠️ Disable with care | Drivers, cloud sync agents, security tools |
| 🚫 Do not touch | OS components, antivirus, system services |

**Never suggest disabling:**
- Windows Defender / Security Center
- macOS System agents under `/System/Library/`
- Systemd core services (`dbus`, `NetworkManager`, `ssh`)
- Anything the user cannot name/identify (flag as "unknown — investigate first")

---

## Delivery Checklist

- [ ] OS identified
- [ ] Audit script provided and labelled read-only
- [ ] Risk levels applied to findings
- [ ] Disable scripts only generated for entries user explicitly named
- [ ] Backup step included in every disable script
- [ ] Re-enable instructions included alongside every disable
- [ ] Scripts saved as files via `create_file` + `present_files`

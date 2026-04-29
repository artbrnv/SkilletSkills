---
name: home-directory-organiser
description: >
  Organise, audit, and clean up a user's home directory on any operating system
  (macOS, Windows, Linux). Use this skill whenever the user asks to tidy, sort,
  declutter, audit, or reorganise their home folder, Desktop, Downloads,
  Documents, or any common personal directory — even if they don't say "home
  directory" explicitly. Trigger phrases include: "my Downloads is a mess",
  "clean up my Desktop", "organise my files", "sort my home folder", "what's
  taking up space", "declutter my computer", "help me tidy up my files", or
  "move files into folders". Always use this skill over general file-management
  advice — it produces runnable, OS-specific scripts the user can execute safely.
---

# Home Directory Organiser

A skill for auditing and reorganising a user's home directory on **macOS**,
**Windows**, or **Linux**. It produces:

1. A **safe, non-destructive audit** (nothing deleted or moved without consent).
2. A **runnable script** in the correct shell for the detected OS.
3. An **undo script** so the user can revert everything with one command.

---

## Step 0 — Detect OS and Home Path

Ask the user which OS they are on if not already known:

- **macOS / Linux** → home is `~/` (`/Users/<name>` or `/home/<name>`)
- **Windows** → home is `%USERPROFILE%` (`C:\Users\<name>`)

If the user pastes `uname -a`, `ver`, or `echo $HOME` output, infer the OS from
that instead.

---

## Step 1 — Understand Scope and Goals

Before writing any script, clarify:

| Question | Why it matters |
|---|---|
| Which folders to target? (Desktop, Downloads, Documents, all?) | Avoids touching sensitive dirs unintentionally |
| Any folders/extensions to skip? | e.g. `.git` dirs, work projects |
| Organise by file type, date, or custom categories? | Drives the folder structure |
| Should duplicates be flagged? | Optional but popular |
| Destructive (move) or non-destructive (copy) first pass? | Safety preference |

Default safe assumptions if the user says "just do it":
- Target: `Desktop` and `Downloads` only
- Organise by **file type** using the standard category map below
- **Move** files (not copy) after showing a preview
- Generate an undo script

---

## Step 2 — Standard Category Map

Use this mapping unless the user requests something different:

```
Images      → .jpg .jpeg .png .gif .webp .svg .heic .raw .tiff .bmp
Videos      → .mp4 .mov .avi .mkv .wmv .flv .m4v .webm
Audio       → .mp3 .wav .flac .aac .ogg .m4a .wma
Documents   → .pdf .doc .docx .xls .xlsx .ppt .pptx .odt .pages .numbers .key
Text        → .txt .md .rtf .csv .json .xml .yaml .yml .log
Code        → .py .js .ts .html .css .sh .rb .go .rs .java .c .cpp .h .sql
Archives    → .zip .tar .gz .rar .7z .dmg .iso .pkg
Executables → .exe .msi .app .deb .rpm (flag, don't auto-move)
Fonts       → .ttf .otf .woff .woff2
Other       → anything not matched above
```

---

## Step 3 — Generate the Audit (Read-Only)

Always produce an **audit first** — a script that only reads and reports, never
moves anything. Show the user what *would* happen before committing.

### macOS / Linux audit (Bash)

```bash
#!/usr/bin/env bash
# audit_home.sh — READ ONLY, nothing is moved
TARGET="${1:-$HOME/Downloads}"
echo "=== Audit of: $TARGET ==="
echo ""
echo "--- File counts by extension ---"
find "$TARGET" -maxdepth 1 -type f | \
  sed 's/.*\.//' | sort | uniq -c | sort -rn

echo ""
echo "--- Largest files (top 20) ---"
find "$TARGET" -maxdepth 1 -type f -exec du -sh {} + 2>/dev/null | \
  sort -rh | head -20

echo ""
echo "--- Total size ---"
du -sh "$TARGET"
```

Run with:
```bash
bash audit_home.sh ~/Downloads
bash audit_home.sh ~/Desktop
```

### Windows audit (PowerShell)

```powershell
# audit_home.ps1 — READ ONLY, nothing is moved
param([string]$Target = "$env:USERPROFILE\Downloads")

Write-Host "=== Audit of: $Target ===" -ForegroundColor Cyan

Write-Host "`n--- File counts by extension ---"
Get-ChildItem -Path $Target -File |
  Group-Object Extension |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize

Write-Host "`n--- Largest files (top 20) ---"
Get-ChildItem -Path $Target -File |
  Sort-Object Length -Descending |
  Select-Object -First 20 |
  Format-Table Name, @{N='Size(MB)';E={[math]::Round($_.Length/1MB,2)}} -AutoSize

Write-Host "`n--- Total size ---"
"{0:N2} MB" -f ((Get-ChildItem $Target -File | Measure-Object Length -Sum).Sum / 1MB)
```

Run with:
```powershell
.\audit_home.ps1 -Target "$env:USERPROFILE\Downloads"
```

---

## Step 4 — Generate the Organise Script

Only after the user has reviewed the audit and confirmed they want to proceed,
produce the full organise script. Always include:

- A **dry-run mode** (`--dry-run` flag) that prints what would happen
- A **log file** recording every move (used by the undo script)
- The **undo script** generated alongside it

### macOS / Linux organise (Bash)

```bash
#!/usr/bin/env bash
# organise_home.sh
# Usage: bash organise_home.sh [--dry-run] [target_dir]

DRY_RUN=false
TARGET="$HOME/Downloads"
LOG="$HOME/.organise_log_$(date +%Y%m%d_%H%M%S).csv"

for arg in "$@"; do
  case $arg in
    --dry-run) DRY_RUN=true ;;
    *) TARGET="$arg" ;;
  esac
done

declare -A CATEGORIES=(
  [Images]="jpg jpeg png gif webp svg heic raw tiff bmp"
  [Videos]="mp4 mov avi mkv wmv flv m4v webm"
  [Audio]="mp3 wav flac aac ogg m4a wma"
  [Documents]="pdf doc docx xls xlsx ppt pptx odt pages numbers key"
  [Text]="txt md rtf csv json xml yaml yml log"
  [Code]="py js ts html css sh rb go rs java c cpp h sql"
  [Archives]="zip tar gz rar 7z dmg iso pkg"
  [Fonts]="ttf otf woff woff2"
)

get_category() {
  local ext="${1,,}"
  for cat in "${!CATEGORIES[@]}"; do
    for e in ${CATEGORIES[$cat]}; do
      [[ "$e" == "$ext" ]] && echo "$cat" && return
    done
  done
  echo "Other"
}

echo "source,destination" > "$LOG"
echo "Organising: $TARGET"
$DRY_RUN && echo "(DRY RUN — nothing will be moved)"

find "$TARGET" -maxdepth 1 -type f | while read -r file; do
  ext="${file##*.}"
  cat=$(get_category "$ext")
  dest_dir="$TARGET/$cat"
  dest_file="$dest_dir/$(basename "$file")"

  if $DRY_RUN; then
    echo "WOULD MOVE: $(basename "$file") → $cat/"
  else
    mkdir -p "$dest_dir"
    mv -- "$file" "$dest_file"
    echo "\"$file\",\"$dest_file\"" >> "$LOG"
    echo "Moved: $(basename "$file") → $cat/"
  fi
done

if ! $DRY_RUN; then
  echo ""
  echo "Done. Log saved to: $LOG"
  echo "To undo: bash undo_organise.sh $LOG"
fi
```

**Undo script (auto-generated):**

```bash
#!/usr/bin/env bash
# undo_organise.sh
# Usage: bash undo_organise.sh <log_file.csv>
LOG="${1:?Please provide the log CSV file}"
tail -n +2 "$LOG" | while IFS=, read -r src dst; do
  src="${src//\"/}"
  dst="${dst//\"/}"
  if [[ -f "$dst" ]]; then
    mv -- "$dst" "$src"
    echo "Restored: $(basename "$src")"
  else
    echo "Skipped (not found): $dst"
  fi
done
echo "Undo complete."
```

### Windows organise (PowerShell)

```powershell
# organise_home.ps1
# Usage: .\organise_home.ps1 [-DryRun] [-Target <path>]
param(
  [switch]$DryRun,
  [string]$Target = "$env:USERPROFILE\Downloads"
)

$LogFile = "$env:USERPROFILE\organise_log_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"

$Categories = @{
  Images    = @('jpg','jpeg','png','gif','webp','svg','heic','raw','tiff','bmp')
  Videos    = @('mp4','mov','avi','mkv','wmv','flv','m4v','webm')
  Audio     = @('mp3','wav','flac','aac','ogg','m4a','wma')
  Documents = @('pdf','doc','docx','xls','xlsx','ppt','pptx','odt','pages','numbers','key')
  Text      = @('txt','md','rtf','csv','json','xml','yaml','yml','log')
  Code      = @('py','js','ts','html','css','sh','rb','go','rs','java','c','cpp','h','sql')
  Archives  = @('zip','tar','gz','rar','7z','dmg','iso','pkg')
  Fonts     = @('ttf','otf','woff','woff2')
}

function Get-Category($ext) {
  $ext = $ext.TrimStart('.').ToLower()
  foreach ($cat in $Categories.Keys) {
    if ($Categories[$cat] -contains $ext) { return $cat }
  }
  return 'Other'
}

"source,destination" | Out-File $LogFile -Encoding utf8
Write-Host "Organising: $Target"
if ($DryRun) { Write-Host "(DRY RUN — nothing will be moved)" -ForegroundColor Yellow }

Get-ChildItem -Path $Target -File | ForEach-Object {
  $cat = Get-Category $_.Extension
  $destDir = Join-Path $Target $cat
  $destFile = Join-Path $destDir $_.Name

  if ($DryRun) {
    Write-Host "WOULD MOVE: $($_.Name) → $cat\"
  } else {
    New-Item -ItemType Directory -Path $destDir -Force | Out-Null
    Move-Item -Path $_.FullName -Destination $destFile
    "$($_.FullName),$destFile" | Out-File $LogFile -Append -Encoding utf8
    Write-Host "Moved: $($_.Name) → $cat\"
  }
}

if (-not $DryRun) {
  Write-Host "`nDone. Log: $LogFile"
  Write-Host "To undo: .\undo_organise.ps1 -Log `"$LogFile`""
}
```

**Undo script:**

```powershell
# undo_organise.ps1
param([string]$Log = $(throw "Provide -Log <path>"))
Import-Csv $Log | ForEach-Object {
  if (Test-Path $_.destination) {
    Move-Item -Path $_.destination -Destination $_.source
    Write-Host "Restored: $(Split-Path $_.source -Leaf)"
  } else {
    Write-Host "Skipped (not found): $($_.destination)"
  }
}
Write-Host "Undo complete."
```

---

## Step 5 — Optional Extras

Offer these after the main script is delivered:

### Duplicate Finder

- **macOS/Linux**: `fdupes -r ~/Downloads` (install via `brew install fdupes` or `apt install fdupes`)
- **Windows**: Recommend [dupeGuru](https://dupeguru.voltaicideas.net/) or the PowerShell snippet below

```powershell
# Find duplicate files by hash in Downloads
Get-ChildItem "$env:USERPROFILE\Downloads" -File |
  Group-Object { (Get-FileHash $_.FullName).Hash } |
  Where-Object { $_.Count -gt 1 } |
  ForEach-Object { $_.Group | Format-Table Name, Length, LastWriteTime }
```

### Old File Archiver

Moves files not accessed in 90+ days to an `_Archive` subfolder:

```bash
# macOS/Linux
find ~/Downloads -maxdepth 1 -type f -atime +90 \
  -exec mv {} ~/Downloads/_Archive/ \;
```

```powershell
# Windows
$cutoff = (Get-Date).AddDays(-90)
$archive = "$env:USERPROFILE\Downloads\_Archive"
New-Item -ItemType Directory $archive -Force | Out-Null
Get-ChildItem "$env:USERPROFILE\Downloads" -File |
  Where-Object { $_.LastAccessTime -lt $cutoff } |
  Move-Item -Destination $archive
```

---

## Delivery Checklist

Before presenting the final output, confirm:

- [ ] OS correctly identified
- [ ] Target directories confirmed with user
- [ ] Audit script included and clearly labelled
- [ ] Organise script includes `--dry-run` / `-DryRun` flag
- [ ] Undo script included alongside organise script
- [ ] Scripts saved as files the user can download (use `create_file` + `present_files`)
- [ ] Run instructions are clear and copy-pasteable

---

## Safety Rules

> **Never** generate a script that:
> - Deletes files permanently (use a `_Trash` staging folder instead)
> - Operates on system directories (`/etc`, `/usr`, `C:\Windows`, etc.)
> - Runs recursively more than 2 levels deep without an explicit user request
> - Moves `.git` directories, `node_modules`, or hidden dotfiles (`.bashrc`, `.zshrc`, etc.)

If the user asks for something that would violate these rules, flag the risk and
suggest a safer alternative.

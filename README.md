# Renaming a Windows User Profile Folder — A Practical Guide

A field-tested, step-by-step guide to renaming an existing Windows user profile
folder (`C:\Users\<name>`) — including every problem we actually ran into, why it
happened, and exactly how it was fixed. Written up after doing this twice on the
same Windows 11 Pro machine.

> ⚠️ **Disclaimer:** Renaming a user profile folder is **not officially supported
> by Microsoft**. This guide worked reliably on a fresh install with a light
> software footprint. Back up (or take a System Restore Point) before you start,
> read the whole guide once before running anything, and understand that on an
> older, heavily-used machine your mileage may vary (see [Why this is harder on
> an old machine](#why-this-is-harder-on-an-old-machine) at the end).
>
> **Recommendation: do this as early as possible — ideally immediately after
> Windows setup, before installing any software.** The number of places that
> cache the old profile path is small and fully known on a clean install; every
> app you install afterward is a potential extra place to find and fix.

---

## Situation & Goal

**Situation:** A Windows local account was originally created with a placeholder
profile folder name (e.g. `C:\Users\jdoe`), and we wanted it renamed to something
meaningful (e.g. `C:\Users\jane.doe`) — without recreating the account, losing
data, or reinstalling software.

**Goal:** `C:\Users\jdoe` → `C:\Users\jane.doe`, with the account fully functional
afterward: correct Desktop/Documents/Downloads locations, no broken system
dialogs, no broken shortcuts, and no need to touch installed applications beyond
fixing a couple of stale path references.

The examples below use `jdoe` (old name) and `jane.doe` (new name) — substitute
your own values throughout.

---

## Why this isn't "just a folder rename"

Renaming the folder itself takes one command. The actual work is that Windows and
many applications **cache the absolute profile path** in several places, and none
of those get updated automatically when you rename the folder:

1. `HKLM\...\ProfileList\<SID>\ProfileImagePath` — tells Windows which folder
   belongs to which account (system-wide, admin-only).
2. `HKCU\...\Explorer\Shell Folders` and `User Shell Folders` — where Desktop,
   Documents, Downloads etc. actually live (per-user).
3. A set of legacy NTFS **junctions** inside the profile folder, kept for
   backward compatibility with old software (per-user, on disk).
4. `HKCU:\Environment` — user-level environment variables like `PATH` or
   app-specific variables that happened to contain the old absolute path.
5. Any **third-party application's own registry entries or config files** that
   independently cached the old path (uninstaller entries, install-path keys,
   shell extensions, autostart entries, etc.).

The rest of this guide fixes each of these, in order, with the exact commands and
the exact symptoms that show up when a step is skipped.

---

## Part 1 — Rename the folder + fix `ProfileImagePath`

Must be done **from a different account** (the target account has to be fully
logged out — not just switched via Fast User Switching, or the folder stays
locked) and requires **administrator rights**.

```powershell
# 1. Confirm the folder exists and isn't locked
Test-Path "C:\Users\jdoe"

# 2. Rename it
Rename-Item -Path "C:\Users\jdoe" -NewName "jane.doe"

# 3. Find the account's SID
Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" | ForEach-Object {
    [PSCustomObject]@{
        SID  = $_.PSChildName
        Path = (Get-ItemProperty $_.PSPath -Name ProfileImagePath -ErrorAction SilentlyContinue).ProfileImagePath
    }
}

# 4. Point ProfileImagePath at the new folder
$sid = "S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-1001"   # from step 3
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$sid" `
    -Name "ProfileImagePath" -Value "C:\Users\jane.doe"

# 5. Verify
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$sid" |
    Select-Object ProfileImagePath
```

### Problem we hit: orphaned profile with a random suffix (`name_xxxxxxx`)

On one of our two accounts, step 2 was done but step 4 wasn't completed before the
user logged back in. Result: Windows didn't recognize the renamed folder as
belonging to that SID, and instead created a **brand-new profile** with a random
suffix appended — `jdoe_a1b2c3d4` — leaving the intended rename half-finished.

**How to recognize it:** any folder under `C:\Users` with a random alphanumeric
suffix is a strong signal that Windows created a fallback profile because
`ProfileImagePath` didn't match an existing, claimed folder.

**How to fix it:** before touching anything, check whether real user data (recent
timestamps, populated `Desktop`/`Documents`/`AppData`) lives in the suffixed
folder — that's usually where the account actually ended up living day-to-day.
Then simply repeat steps 1–5 above, targeting the suffixed folder as the "old"
name and your intended clean name as the "new" one.

```powershell
Rename-Item -Path "C:\Users\jdoe_a1b2c3d4" -NewName "jane.doe"
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$sid" `
    -Name "ProfileImagePath" -Value "C:\Users\jane.doe"
```

---

## Part 2 — Re-link every reference (run *inside* the renamed account)

Everything from here on is `HKCU`, so it only affects the currently logged-in
user — do this logged into the renamed account itself.

### 2.1 Shell Folders (Desktop, Documents, Downloads, ...)

There are **two** parallel registry keys:

- `HKCU:\...\Explorer\User Shell Folders` — the key Windows actually resolves
  paths from.
- `HKCU:\...\Explorer\Shell Folders` — a **cached copy** of the same paths that
  does **not** get refreshed automatically when the profile is renamed.

**Symptom when this is stale:** components that read the cached copy directly
break. In our case, the Windows **Settings app failed to launch** until this was
fixed.

```powershell
$old = "C:\Users\jdoe"
$new = "C:\Users\jane.doe"

foreach ($keyPath in @(
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders"
)) {
    (Get-ItemProperty $keyPath).PSObject.Properties |
      Where-Object { $_.Value -is [string] -and $_.Value -match [regex]::Escape($old) } |
      ForEach-Object {
        $newValue = $_.Value -replace [regex]::Escape($old), $new
        Set-ItemProperty -Path $keyPath -Name $_.Name -Value $newValue
        Write-Output "Fixed [$($_.Name)]: $($_.Value) -> $newValue"
      }
}
```

On our system this touched ~28 values: Desktop, Documents, Downloads, Music,
Pictures, Videos, Favorites, AppData, Local AppData, Cache, Cookies, History,
Start Menu, Programs, Startup, Templates, NetHood, PrintHood, Contacts, Links,
Saved Games, Searches, RoamingTiles, Libraries, CD Burning, LocalLow,
Administrative Tools.

### 2.2 Legacy compatibility junctions

Windows keeps a set of NTFS **junctions** in the profile root for old software
that expects legacy folder names (localized on non-English systems, e.g. German
`Anwendungsdaten` for `AppData\Roaming`). Their reparse-point target is stored as
a **literal absolute path** and is not rewritten by a plain folder rename.

**Detect** (don't rely on `Get-Item.Target` — it's unreliable for this; use
`fsutil` to see the real stored value):

```powershell
fsutil reparsepoint query "C:\Users\jane.doe\Application Data"
# Substitute Name may still show: \??\C:\Users\jdoe\AppData\Roaming
```

**Fix** — delete and recreate each junction with the correct target. This does
**not** require admin rights (unlike symlinks), only write access to the profile
folder, and `Remove-Item` on a junction removes only the link, never the real
data behind it:

```powershell
$base = "C:\Users\jane.doe"
$junctions = @{
  "Application Data"  = "AppData\Roaming"
  "Cookies"            = "AppData\Local\Microsoft\Windows\INetCookies"
  "PrintHood"          = "AppData\Roaming\Microsoft\Windows\Printer Shortcuts"
  "My Documents"       = "Documents"
  "Local Settings"     = "AppData\Local"
  "NetHood"            = "AppData\Roaming\Microsoft\Windows\Network Shortcuts"
  "Recent"             = "AppData\Roaming\Microsoft\Windows\Recent"
  "SendTo"             = "AppData\Roaming\Microsoft\Windows\SendTo"
  "Start Menu"         = "AppData\Roaming\Microsoft\Windows\Start Menu"
  "Templates"          = "AppData\Roaming\Microsoft\Windows\Templates"
}
foreach ($kv in $junctions.GetEnumerator()) {
    $linkPath   = Join-Path $base $kv.Key
    $targetPath = Join-Path $base $kv.Value
    if (Test-Path $targetPath) {
        Remove-Item $linkPath -Force
        New-Item -ItemType Junction -Path $linkPath -Target $targetPath | Out-Null
        Write-Output "Fixed: $($kv.Key) -> $targetPath"
    } else {
        Write-Output "SKIP (target missing): $($kv.Key) -> $targetPath"
    }
}
```

(Names above are the English defaults; on a localized Windows install use the
actual folder names present on disk — check with `Get-ChildItem $base -Force`.)

### 2.3 User-level environment variables

```powershell
$envKey = "HKCU:\Environment"
$old = "C:\Users\jdoe"
$new = "C:\Users\jane.doe"
(Get-ItemProperty $envKey).PSObject.Properties |
  Where-Object { $_.Value -is [string] -and $_.Value -match [regex]::Escape($old) } |
  ForEach-Object {
    $newValue = $_.Value -replace [regex]::Escape($old), $new
    Set-ItemProperty -Path $envKey -Name $_.Name -Value $newValue
    Write-Output "Fixed [$($_.Name)]: $($_.Value) -> $newValue"
  }
```

In our case this caught two things: a stale entry in `Path` (added by a
previously-installed app) and a stale `OneDrive` variable.

### 2.4 Final sweep

```powershell
$leftover = @()
$leftover += (Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders").PSObject.Properties |
  Where-Object { $_.Value -is [string] -and $_.Value -match "jdoe" }
$leftover += (Get-ItemProperty "HKCU:\Environment").PSObject.Properties |
  Where-Object { $_.Value -is [string] -and $_.Value -match "jdoe" }
if ($leftover.Count -eq 0) { "Clean." } else { $leftover }
```

### 2.5 Sanity check

```powershell
Start-Process "ms-settings:"
Get-Process | Where-Object { $_.ProcessName -match "SystemSettings" }
```

If the Settings app opens and stays running, the core Shell Folder references are
correctly wired.

---

## Case study: fixing a third-party app with its own cached path (OneDrive)

The same pattern — *find every place the old path is cached, then fix it* —
applies to individual installed applications too, not just Windows itself. We
hit this trying to uninstall OneDrive after the rename: **uninstalling failed**
because the uninstaller's own registry entry pointed at a file that no longer
existed.

**Where OneDrive had cached the old path (all `HKCU`):**

| Key | Value(s) | Effect if stale |
|---|---|---|
| `HKCU:\Software\Microsoft\OneDrive\<version>` | `InstallPath`, `InstallPaths` | Cosmetic / internal bookkeeping |
| `HKCU:\Software\Classes\CLSID\{<OneDrive-shell-extension-GUID>}` | `(default)` | Broken icon in Explorer sidebar |
| `HKCU:\...\Uninstall\OneDriveSetup.exe` | `UninstallString`, `DisplayIcon` | **Uninstall fails outright** — this was the actual root cause |
| `HKCU:\...\Run` | `OneDrive` | App silently fails to autostart at login |

**General approach for any app in this situation:**

1. Search the app's known registry tree recursively for the old path string.
2. Specifically check `HKCU:\...\CurrentVersion\Uninstall\<AppName>` —
   this is the entry the uninstaller/Settings-app "Uninstall" button actually
   reads, and a stale `UninstallString` here is the classic cause of "won't
   uninstall" complaints.
3. Check the `Run` key for a stale autostart path.
4. Fix each value, verify the target file the corrected path now points at
   actually exists, then run the (now-correct) uninstaller.

```powershell
$new = "C:\Users\jane.doe"

Set-ItemProperty "HKCU:\Software\Microsoft\OneDrive\<version>" -Name "InstallPath" `
    -Value "$new\AppData\Local\Microsoft\OneDrive\<version>"

$unKey = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\OneDriveSetup.exe"
Set-ItemProperty $unKey -Name "UninstallString" `
    -Value "`"$new\AppData\Local\Microsoft\OneDrive\<version>\OneDriveSetup.exe`"  /uninstall"

$uninstaller = "$new\AppData\Local\Microsoft\OneDrive\<version>\OneDriveSetup.exe"
Start-Process -FilePath $uninstaller -ArgumentList "/uninstall" -Wait
```

### Extra snag: leftover process / locked install folder

After uninstalling, the process was still running and the install folder
couldn't be removed immediately. Windows allows "unlinking" files that are still
open (delete-on-close), so the folder can disappear from `Test-Path` before the
process has actually exited — give it a moment, then confirm:

```powershell
Stop-Process -Name OneDrive -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 2
Remove-Item "$new\AppData\Local\Microsoft\OneDrive" -Recurse -Force -ErrorAction SilentlyContinue
```

**If `Stop-Process -Force` still throws "Access denied":** check whether the
process actually belongs to a *different* logged-in account (e.g. another user
still active in the background via Fast User Switching). You have no rights to
kill another account's process, and that's expected, not a bug:

```powershell
$p = Get-CimInstance Win32_Process -Filter "ProcessId = <PID>"
Invoke-CimMethod -InputObject $p -MethodName GetOwner
```

---

## Checklist

- [ ] Target account fully logged out (not just switched)
- [ ] Folder renamed (from an admin session)
- [ ] `ProfileImagePath` (HKLM) updated
- [ ] No random-suffix orphan folder appeared on next login
- [ ] `Shell Folders` **and** `User Shell Folders` (HKCU) fixed
- [ ] All legacy junctions fixed (verify with `fsutil reparsepoint query`)
- [ ] `HKCU:\Environment` fixed (`Path` and any app-specific variables)
- [ ] Settings app / core Windows dialogs open without error
- [ ] Final sweep over Shell Folders + Environment returns no matches
- [ ] Individually check any third-party app that manages its own sync/cache
      folder (cloud storage clients, IDEs with "recent projects", backup tools)

---

## Why this is harder on an old machine

On a freshly installed system, the list of places caching the old profile path is
**small and fully enumerable** — exactly what's documented above. On a machine
that's been in use for a while, you get an open-ended long tail on top of that:
IDE recent-project lists, browser settings, other cloud-sync clients, shortcuts
with hardcoded absolute targets, the search index, scheduled tasks, and any
other app that happened to cache an absolute path somewhere. The *technique*
doesn't change — it's still "find the cached reference, rewrite it" — but
proving you found *all* of them stops being realistic. That's the actual reason
Microsoft doesn't support this and most guides recommend creating a new account
and migrating data instead of renaming in place.

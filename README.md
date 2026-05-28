# Bootload Modifier (BLM) — v0.1.0

> **Cross-platform bootloader management for advanced users and IT professionals.**
> View, modify, reorder and clean up boot entries through a clean browser-based GUI —
> without touching config files manually.

![Status](https://img.shields.io/badge/status-prototype-orange)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Windows%20%7C%20macOS-blue)
![Python](https://img.shields.io/badge/python-3.8%2B-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## ⚠️ Disclaimer 

This tool is intended for **experienced enthusiasts and IT professionals** who fully
understand what a bootloader is and the consequences of modifying it.

> Incorrect changes can **prevent your system from starting**, cause data loss, or leave
> your OS in an unrecoverable state. Always have a **live USB or recovery media** ready
> before making any changes. **You use this software entirely at your own risk.**
> The author accepts no responsibility whatsoever for any damage or data loss arising
> from the use or misuse of this program.

---

## 📋 Table of Contents

- [Description](#-description)
- [Features](#-features)
- [Supported Platforms](#-supported-platforms)
- [Architecture](#️-architecture)
- [File Structure](#-file-structure)
- [Installation & Usage](#-installation--usage)
- [Building an Executable](#-building-an-executable)
- [GUI Overview](#️-gui-overview)
- [Safety & Stability](#-safety--stability)
- [Full Changelog & Bug Fixes](#-full-changelog--bug-fixes)
- [C++ Backend (In Progress)](#-c-backend-in-progress)
- [Roadmap](#-roadmap)
- [Contributing](#-contributing)

---

## 🚀 Description

Bootload Modifier (BLM) is a system utility that simplifies bootloader management through
a modern graphical interface. It provides fine-grained control over boot entries across
multiple operating systems while maintaining safety through automatic backups, input
validation, and a full undo system.

Built for advanced users who work with multi-boot environments — distro hoppers, sysadmins,
and IT professionals who need reliable control over boot configuration systems such as GRUB2,
systemd-boot, and Windows Boot Manager — without having to memorise command-line syntax or
risk a typo in a config file.

The app runs as a local web server and opens in your browser. This means the GUI works
identically on Linux, Windows, and macOS with no platform-specific UI framework required.

---

## ✨ Features

- **Auto-detection** of bootloader on startup (GRUB2, systemd-boot, Windows BCD, macOS)
- **View all boot entries** with color-coded labels showing type and status
- **Change default boot entry** with confirmation dialog
- **Rename entries** for clarity — with full undo
- **Delete old entries** — with guards and undo support
- **Reorder entries** in the boot menu — with undo
- **↩ Undo last change** — every write operation records a precise undo action
- **Settings persistence** — all preferences saved to disk, restored on every launch
- **Demo mode** — full GUI with realistic mock data, no root required, nothing written to disk
- **Automatic backup** before every write (`~/.config/bootload-modifier/backups/`)
- **Windows Update warning** — detects a pending reboot and warns you before making changes
- **Operation log** at `~/.config/bootload-modifier/blm.log`
- **Disclaimer toggle** — power users can disable the startup disclaimer in Settings
- **Dark mode** — follows your OS preference automatically
- **Auto-loads on startup** — no manual rescan needed when opening the app
- **Port conflict handling** — tries ports 7373–7382 if default is in use
- **Clean shutdown** — proper Ctrl+C handling on Windows and Linux

---

## 🖥️ Supported Platforms

| OS | Bootloader | Support | Notes |
|---|---|---|---|
| Linux | GRUB2 | ✅ Full | Parses `grub.cfg`, writes `GRUB_DEFAULT`, runs `grub-mkconfig` |
| Linux | systemd-boot | ✅ Full | Reads/writes `.conf` files and `loader.conf` directly |
| Windows | BCD store | ✅ Full | Wraps `bcdedit.exe` — requires Administrator |
| macOS (Intel) | bless | ⚠️ Partial | `bless --setBoot` — limited control |
| macOS (Apple Silicon) | recoveryOS | 📖 Read-only | Sealed system volume — no direct EFI access |

---

## 🏗️ Architecture

The project is split into three layers:

```
+---------------------------------------+
|           Browser GUI                 |  gui.html
|   HTML / CSS / JavaScript             |  http://localhost:7373
|   No external framework               |
+------------------+--------------------+
                   | JSON over HTTP
+------------------v--------------------+
|         HTTP API Server               |  server.py
|   /api/entries    /api/set-default    |  Python built-in http.server
|   /api/delete     /api/rename         |  Binds to 127.0.0.1 only
|   /api/move       /api/undo           |
|   /api/rescan     /api/undo-info      |
|   /api/info       /api/log            |
|   /api/settings   /api/ready          |
+------------------+--------------------+
                   | Python method calls
+------------------v--------------------+
|       IBootBackend (abstract)         |  backend.py
|                                       |
|   Grub2Backend                        |
|   SystemdBootBackend                  |
|   WindowsBcdBackend                   |
|   MacOSBackend                        |
|   MockBackend  (demo mode)            |
|                                       |
|   PlatformDetector  Backup  Logger    |
|   Undo stack (all operations)         |
+---------------------------------------+
```

### Key design principles

**Single abstract interface.** Every bootloader backend implements `IBootBackend` with the
same methods: `list_entries`, `set_default`, `delete_entry`, `rename_entry`, `move_entry`,
`rescan`. The GUI and server never know which backend is running.

**Result tuples instead of exceptions.** Every operation returns `(True, value)` on success
or `(False, error_string)` on failure. Errors are always surfaced to the user as readable
messages — never raw tracebacks.

**Undo stack.** Before every write, a lambda is pushed onto an undo stack that precisely
reverses the change. Holds up to 10 entries. GUI reflects undo availability in real time.

**Backup before every write.** `Backup.save()` is called before touching any file or running
any system command. Backups are timestamped copies in `~/.config/bootload-modifier/backups/`.

**Settings persistence.** User preferences are saved to
`~/.config/bootload-modifier/settings.json` on every change and restored on every launch.
Corrupted or missing settings files fall back to safe defaults automatically.

**Ready polling.** The GUI polls `/api/ready` on startup and waits for the backend to finish
its initial scan before loading entries. This ensures the entry list is always populated
correctly without requiring a manual rescan.

**Demo mode.** `MockBackend` implements the full interface with realistic in-memory data.
All GUI features including undo work in demo mode — nothing is written to disk.

---

## 📁 File Structure

```
bootload-modifier/
│
├── blm.py              Entry point — run this to start the app
├── backend.py          All bootloader backends + undo system (1029 lines)
├── server.py           HTTP server + JSON API layer (276 lines)
├── gui.html            Complete browser GUI (1192 lines)
│
├── build.bat           Windows build script → dist\BootloadModifier.exe
├── build.sh            Linux/macOS build script → dist/BootloadModifier
├── blm.spec            PyInstaller spec for single-file packaging
│
├── README.md           This file
│
└── cpp-backend/        C++ backend (in progress — see section below)
    ├── CMakeLists.txt
    ├── include/
    │   ├── BootEntry.hpp           Shared data structures + Result<T> type
    │   ├── IBootBackend.hpp        Abstract interface
    │   ├── BackendBridge.hpp       JSON IPC layer (stdin/stdout)
    │   ├── Backup.hpp
    │   ├── Logger.hpp
    │   ├── PlatformDetector.hpp
    │   └── backends/
    │       ├── Grub2Backend.hpp
    │       ├── SystemdBootBackend.hpp
    │       ├── WindowsBcdBackend.hpp
    │       └── MacOSBackend.hpp
    └── src/
        └── backends/
            └── Grub2Backend.cpp    Fully implemented GRUB2 backend in C++
```

---

## 📦 Installation & Usage

### Requirements
- Python 3.8 or newer (zero external packages needed)
- A modern browser (Chrome, Firefox, Edge, Safari)

---

### 🪟 Windows

#### Demo mode — safe, no admin needed
```cmd
python blm.py --demo
```

#### Real mode — run terminal as Administrator first
```cmd
python blm.py
```

> ⚠️ Full BCD read/write access requires running as **Administrator**.
> Right-click your terminal → "Run as administrator" before launching.

---

### 🐧 Linux

#### Demo mode
```bash
python3 blm.py --demo
```

#### Real mode
```bash
sudo python3 blm.py
```

---

### 🍎 macOS

#### Demo mode
```bash
python3 blm.py --demo
```

#### Real mode (Intel only — Apple Silicon is read-only)
```bash
sudo python3 blm.py
```

---

The app opens automatically in your default browser at `http://localhost:7373`.
Press **Ctrl+C** in the terminal to quit.

---

## 🔧 Building an Executable

Uses [PyInstaller](https://pyinstaller.org) to bundle everything into a single file
with no Python installation required on the target machine.

### Windows → `dist\BootloadModifier.exe`

Double-click `build.bat`, or manually:
```cmd
pip install pyinstaller
pyinstaller blm.spec --clean --noconfirm
```

### Linux / macOS → `dist/BootloadModifier`

```bash
chmod +x build.sh && ./build.sh
```

Right-click the `.exe` and choose "Run as administrator" for full write access on Windows.

---

## 🖥️ GUI Overview

### Boot Entries tab

The main view. All boot entries in a list with color-coded tags:

| Tag | Meaning |
|---|---|
| `default` | Current default boot target — green left border |
| `fallback` | Fallback initramfs entry |
| `old distro` | Different kernel version than the current default |
| `read-only` | Cannot be modified (UEFI firmware entries, Apple Silicon) |

Click an entry to select it and see full details (kernel, partition, UUID, config path).
Click a selected entry again to deselect it.

**Actions panel:**

| Action | Description |
|---|---|
| Set as default | Makes selected entry the default — confirmation dialog shown |
| Rename entry | Opens dialog — Enter to confirm, Escape to cancel |
| Move up / Move down | Reorders in the boot menu |
| Delete entry | Removes entry — blocked if it is the current default |
| ↩ Revert last change | Undoes the most recent write — label shows exactly what |

### Platform Support tab

Bootloader support level per OS, plus info about the currently active backend and
whether it has write access.

### Settings tab

| Setting | Default | Description |
|---|---|---|
| Backup before changes | On | Saves config to `backups/` before every write |
| Show technical metadata | On | UUID, config paths in entry details |
| Confirm before changes | On | Confirmation dialog before set-default and delete |
| Demo mode | Off | Mock data — no real writes to disk |
| Show disclaimer on launch | On | Risk disclaimer on first launch. Disable for frequent use. |

All settings are saved automatically and restored on every launch.

### Log tab

Last 200 lines of `blm.log` — color coded INFO / WARNING / ERROR.
Includes a refresh button.

---

## 🧪 Safety & Stability

- **Automatic backups** — timestamped copies of every config file before any write
- **Full undo stack** — set-default, rename, delete, and move all support precise undo
- **Irreversible operation handling** — BCD delete cannot be undone; the undo button
  explains this clearly and tells the user what manual steps would be needed to recover
- **Windows Update warning** — detects pending reboot via registry and warns before changes
- **Confirmation dialogs** — both set-default and delete require confirmation (toggleable)
- **Guards against dangerous operations** — cannot delete the current default, cannot
  modify read-only entries, cannot set UEFI firmware entries as default
- **Demo mode** — full GUI with no system access, safe for testing and exploration
- **Input validation** — empty names rejected, boundary checks on reorder
- **Graceful failure** — all errors shown as readable messages in the GUI
- **Settings resilience** — corrupted or missing settings file silently falls back to defaults
- **Port conflict recovery** — finds an available port automatically if 7373 is taken
- **Localhost only** — the API server binds to `127.0.0.1`, never network-accessible
- **Startup ready check** — GUI waits for backend scan to complete before displaying entries

---

## 📝 Full Changelog & Bug Fixes

### v0.1.0 — Initial release

#### Core implementation

- Three-layer architecture: browser GUI → HTTP API server → bootloader backend
- `IBootBackend` abstract interface with `(ok, value)` result tuple pattern
- `PlatformDetector` — detects OS and active bootloader at startup, auto-selects backend
- `Backup` utility — timestamped config copies saved before every write operation
- `Logger` utility — append-only timestamped operation log (`blm.log`)
- `MockBackend` — full in-memory demo backend with 7 realistic boot entries
- `Undo stack` — every write operation pushes a precise reversal lambda (up to 10 deep)
- Settings persistence — preferences saved to `settings.json`, restored on every launch
- PyInstaller build pipeline (`build.bat`, `build.sh`, `blm.spec`)

#### GRUB2 backend
- Parses `/boot/grub/grub.cfg` and `/boot/grub2/grub.cfg` (Fedora/RHEL)
- Detects entry types: Linux, Windows, EFI, fallback
- Reads `GRUB_DEFAULT` from `/etc/default/grub` to identify active default
- `set_default` — writes `GRUB_DEFAULT`, runs `grub-mkconfig` or `grub2-mkconfig`
- `delete_entry` — removes BLM-managed snippets from `/etc/grub.d/`; explains why
  auto-generated entries cannot be directly deleted
- Guards against deletion of the current default entry
- Auto-detects `grub-mkconfig` vs `grub2-mkconfig`

#### systemd-boot backend
- Scans `/boot/loader/entries/*.conf`
- Reads/writes `/boot/loader/loader.conf` for current default
- `set_default` — rewrites `default=` line with backup
- `delete_entry` — removes `.conf` file with backup
- `rename_entry` — edits `title=` line in place with backup and undo

#### Windows BCD backend
- `list_entries` — parses `bcdedit /enum ALL /v` output into structured entries
- Reads display order from Boot Manager block for correct entry ordering
- Handles `Windows Boot Loader`, `Windows Boot Sector`, `Resume from Hibernate`
- `set_default` — `bcdedit /default {guid}` with undo
- `delete_entry` — `bcdedit /delete {guid}` with honest irreversible-undo warning
- `rename_entry` — `bcdedit /set {guid} description "name"` with undo
- `move_entry` — `bcdedit /displayorder` with full reordered GUID list and undo
- UAC elevation detection with clear human-readable error

#### macOS backend
- Apple Silicon detection at init
- Intel: `bless --setBoot` (partial implementation)
- Apple Silicon: read-only with explanation of sealed system volume restriction

#### HTTP API server
- All endpoints bound to `127.0.0.1` — never network-accessible
- Serves `gui.html` at `/`
- `GET /api/info` — backend name, kind, write access, OS, Windows Update status
- `GET /api/entries` — full entry list
- `GET /api/log` — last 200 log lines
- `GET /api/undo-info` — current undo description and availability
- `GET /api/settings` — load persisted user settings
- `GET /api/ready` — backend startup ready flag
- `POST /api/set-default` — change default boot entry
- `POST /api/delete` — delete an entry
- `POST /api/rename` — rename an entry
- `POST /api/move` — reorder an entry
- `POST /api/rescan` — re-read config from disk
- `POST /api/undo` — execute undo for last operation
- `POST /api/settings` — save user settings to disk
- `POST /api/quit` — clean shutdown
- `winUpdatePending` detection via Windows registry `RebootRequired` key
- Backend pre-scan on startup so entries are ready when browser opens
- Port conflict handling — tries 7373–7382, prints active port
- `SIGINT` / `SIGBREAK` signal handling for clean Ctrl+C on Windows and Linux
- PyInstaller `_MEIPASS` resource path support for bundled executable

#### GUI
- Four tabs: Boot Entries, Platform Support, Settings, Log
- Color-coded entry tags (default, fallback, old distro, read-only)
- Green left border on default entry
- Full entry detail panel (kernel, partition, type, UUID, config path)
- Dark mode via `prefers-color-scheme: dark` CSS media query
- XSS protection — all dynamic content passed through `esc()` before rendering
- Toast notifications with 3-second auto-dismiss
- All errors shown as human-readable messages — never raw Python tracebacks

---

#### Bug fixes and improvements (in development order)

| # | Problem | Fix |
|---|---|---|
| 1 | Checkbox could not be unchecked after first click | Added toggle-deselect: clicking a selected entry deselects it; `stopPropagation` on checkbox prevents double-fire |
| 2 | Windows BCD returned empty list silently when not running as Administrator | Returns a clear error message with step-by-step instructions, shown directly in the GUI |
| 3 | Disclaimer shown once ever and then permanently suppressed | Kept `localStorage` (once-ever) but added a **Settings toggle** so users can re-enable or permanently disable it |
| 4 | Boot entry list not refreshed on app launch — manual rescan required every time | Backend now pre-scans on server startup; GUI polls `/api/ready` before loading entries, waits for backend to be ready |
| 5 | Ctrl+C did not reliably stop the server on Windows | Replaced `KeyboardInterrupt` with proper `signal.SIGINT` + `SIGBREAK` handlers and a `finally` block with a clean exit message |
| 6 | BCD `move_entry` returned "not yet implemented" error | Implemented using `bcdedit /displayorder` with the full reordered GUID list in a single command |
| 7 | BCD parser missed display order and some entry types | Parser now reads `displayorder` from Boot Manager block for correct sorting; also handles `Windows Boot Sector` and `Resume from Hibernate` entries |
| 8 | No way to undo a change after making it | Full undo stack implemented: `set_default`, `rename`, `delete`, and `move` all push precise undo lambdas; GUI revert button label shows exactly what will be reverted |
| 9 | No warning when a Windows Update reboot was pending — risky combination with boot changes | Registry check for `HKLM\...\RebootRequired`; amber warning banner shown at top of the app when a pending reboot is detected |
| 10 | Browser tab title always showed "Bootload Modifier" regardless of selection | Title updates to show the selected entry name (e.g. `Arch Linux — Bootload Modifier`) and resets when deselected |
| 11 | After rescan, stale detail panel shown when the previously selected entry had been removed | Rescan now checks if the selected entry still exists; deselects cleanly with a cleared detail panel if not |
| 12 | Status line showed stale pending count after hitting Apply | Status is now driven by the real undo stack state, not a click counter; Apply resets status to `ready` |
| 13 | Hard crash if port 7373 was already in use | Server now tries ports 7373–7382 and prints which port it successfully bound to |
| 14 | No confirmation dialog before changing the default boot entry | Set-default now shows a confirmation dialog (same toggle as delete confirmation) showing both old and new default |
| 15 | Settings (backup, metadata, confirm, demo, disclaimer) reset to defaults on every app restart | All settings persisted to `~/.config/bootload-modifier/settings.json`; loaded and applied to toggles on every launch; graceful fallback if file is missing or corrupted |
| 16 | BCD `delete_entry` and `rename_entry` had no undo support | `rename_entry` pushes a proper undo that calls `bcdedit /set {guid} description "{original}"` to restore; `delete_entry` pushes an honest explanation that BCD deletion is irreversible with guidance on manual recovery |
| 17 | Mock backend `delete_entry` and `rename_entry` had no undo | Both now push precise undo lambdas so all operations are reversible in demo mode |
| 18 | Pending change count was a cosmetic click counter, not tied to real backend state | Replaced with `_hasChanges` flag driven by the actual undo stack via `/api/undo-info` polling |

---

## ⚙️ C++ Backend (In Progress)

The `cpp-backend/` folder contains the foundation of a compiled C++ backend that will
eventually replace the Python backend — removing the Python dependency and improving
startup time and system-level integration.

### What is implemented

**`BootEntry.hpp`** — shared data structures:
- `BootEntry` struct with all fields (id, name, kernel, partition, uuid, configPath,
  type, isDefault, isReadOnly)
- `EntryType` and `BootloaderKind` enums
- `Result<T>` template for error handling without exceptions across the IPC boundary
- `Result<void>` specialisation for write operations

**`IBootBackend.hpp`** — pure abstract C++ interface matching the Python design exactly,
with the same six core operations plus identity methods.

**`Grub2Backend.hpp` + `Grub2Backend.cpp`** — fully implemented C++20 GRUB2 backend:
- Parses `grub.cfg` using `<regex>` to extract all `menuentry` blocks
- Reads `GRUB_DEFAULT` from `/etc/default/grub` to identify active default
- `setDefault` — writes `GRUB_DEFAULT` and runs `grub-mkconfig`
- `deleteEntry` — removes BLM-managed snippets from `/etc/grub.d/`
- `renameEntry` — edits snippet title in place
- Root privilege check via `geteuid()`
- Resolves `grub-mkconfig` vs `grub2-mkconfig` automatically

**Header stubs ready for implementation:**
`SystemdBootBackend.hpp`, `WindowsBcdBackend.hpp`, `MacOSBackend.hpp`,
`BackendBridge.hpp`, `Backup.hpp`, `Logger.hpp`, `PlatformDetector.hpp`

**`CMakeLists.txt`** — cross-platform CMake build with CPack packaging:
- Linux: `.deb`, `.rpm`, `.tar.gz`
- Windows: NSIS installer + `.zip`
- macOS: `.dmg` + `.tar.gz`

### IPC protocol (BackendBridge)

When the C++ backend is complete, the GUI communicates with the binary over
newline-delimited JSON on stdin/stdout:

```json
{ "id": 1, "action": "listEntries" }
{ "id": 2, "action": "setDefault",  "entryId": "grub-0" }
{ "id": 3, "action": "deleteEntry", "entryId": "grub-2" }
{ "id": 4, "action": "renameEntry", "entryId": "grub-1", "name": "Arch Linux" }
{ "id": 5, "action": "moveEntry",   "entryId": "grub-1", "delta": -1 }
{ "id": 6, "action": "rescan" }
{ "id": 7, "action": "quit" }

{ "id": 1, "ok": true,  "data": { ... } }
{ "id": 1, "ok": false, "error": "human-readable message" }
```

### Implementation status

| File | Status |
|---|---|
| `include/BootEntry.hpp` | ✅ Done |
| `include/IBootBackend.hpp` | ✅ Done |
| `include/BackendBridge.hpp` | ✅ Done |
| `include/Backup.hpp` | ✅ Done |
| `include/Logger.hpp` | ✅ Done |
| `include/PlatformDetector.hpp` | ✅ Done |
| `include/backends/Grub2Backend.hpp` | ✅ Done |
| `include/backends/SystemdBootBackend.hpp` | ✅ Done |
| `include/backends/WindowsBcdBackend.hpp` | ✅ Done |
| `include/backends/MacOSBackend.hpp` | ✅ Done |
| `CMakeLists.txt` | ✅ Done |
| `src/backends/Grub2Backend.cpp` | ✅ Done |
| `src/utils/Backup.cpp` | ❌ Pending |
| `src/utils/Logger.cpp` | ❌ Pending |
| `src/PlatformDetector.cpp` | ❌ Pending |
| `src/BackendBridge.cpp` | ❌ Pending |
| `src/backends/SystemdBootBackend.cpp` | ❌ Pending |
| `src/backends/WindowsBcdBackend.cpp` | ❌ Pending |
| `src/backends/MacOSBackend.cpp` | ❌ Pending |
| `src/main.cpp` | ❌ Pending |

---

## 🚧 Roadmap

### v0.2.0 — C++ backend
- [ ] `Backup.cpp` and `Logger.cpp`
- [ ] `PlatformDetector.cpp`
- [ ] `SystemdBootBackend.cpp`
- [ ] `WindowsBcdBackend.cpp`
- [ ] `BackendBridge.cpp` and `main.cpp`
- [ ] Qt6 or Tauri GUI connected to the C++ binary
- [ ] Single compiled binary — no Python required on the target machine

### v0.3.0 — Packaging and polish
- [ ] Windows NSIS installer with UAC manifest
- [ ] Linux AppImage, `.deb`, `.rpm`
- [ ] macOS `.dmg`
- [ ] GRUB2 entry reordering via `/etc/grub.d/` script prefix numbering
- [ ] systemd-boot entry reordering via filename rename
- [ ] macOS Intel — full `bless` implementation
- [ ] UEFI variable editing via `efibootmgr`
- [ ] Multiple undo steps shown in a history panel

### Future ideas
- [ ] Config diff viewer — show exactly what will change before applying
- [ ] Automatic detection of orphaned entries (partition no longer exists)
- [ ] Export/import boot configuration as JSON
- [ ] Flatpak / Homebrew packaging

---

## 🤝 Contributing

Contributions, issues, and pull requests are welcome.

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Test with `python3 blm.py --demo` before testing with real boot data
4. Open a pull request with a clear description of what you changed and why

### Areas where help is most welcome

- C++ backend implementations (`SystemdBootBackend`, `WindowsBcdBackend`, `BackendBridge`, `main.cpp`)
- macOS testing — especially on Intel Macs
- Packaging (AppImage, Flatpak, Homebrew, NSIS)
- Automated test suite for backend logic

### Ground rules

- Never write to real boot config without an explicit user action
- Always call `Backup.save()` before any write operation
- All user-facing errors must be human-readable — no raw exceptions
- Test in demo mode first, real mode second
- Document what your code does and why, not just what it does

---

## 🔥 Project Status

**Prototype stage** — fully functional Python core (12/12 backend tests passing,
23/23 GUI checks passing, 18 bug fixes applied). Active development with ongoing
migration toward a full C++ production backend.

---

## 📄 License

MIT — LICENSE file to be added.

---

*Built with Python 3.8+, zero external dependencies for the runnable version.*
*C++ backend targets C++20 with CMake 3.20+.*

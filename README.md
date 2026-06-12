# Bootload Modifier (BLM)

Cross-platform bootloader management for advanced users and IT professionals.
View, rename, reorder, and change the default boot entry across GRUB2, systemd-boot,
Windows BCD, and macOS — without touching config files manually.

> ⚠️ **Disclaimer**
> This tool is intended for experienced enthusiasts and IT professionals who fully
> understand what a bootloader is and the consequences of modifying it.
> Incorrect changes can prevent your system from starting, cause data loss, or leave
> your OS in an unrecoverable state. Always have a live USB or recovery media ready
> before making any changes. You use this software entirely at your own risk.

---

## Table of Contents

- [Description](#description)
- [What Works Where](#what-works-where)
- [Supported Platforms](#supported-platforms)
- [Architecture](#architecture)
- [File Structure](#file-structure)
- [Quick Start](#quick-start)
- [Python App](#python-app)
- [C++ Stack](#c-stack)
- [Running the Stress Tests](#running-the-stress-tests)
- [Safety & Stability](#safety--stability)
- [Known Limitations](#known-limitations)
- [Changelog](#changelog)
- [Roadmap](#roadmap)
- [Contributing](#contributing)

---

## Description

Bootload Modifier (BLM) is a system utility that gives you fine-grained control
over boot entries across multiple operating systems, without memorising command-line
syntax or risking a typo in a config file.

The project ships in two layers:

**Python app** — fully functional today, runs on any machine with Python 3.8+.
Offers two GUI modes: a native FLTK window (default on Windows and macOS) and
a browser-based GUI (`--browser`, no pyFLTK needed). The browser GUI is the
fallback on Linux where no prebuilt pyFLTK wheel is available.

**C++ stack** — a compiled C++20 backend (`blm-backend`) that communicates with two
native GUI frontends (Dear ImGui and Nuklear+SDL2) over a JSON IPC protocol.
The C++ stack is fully compiled and tested on Linux and Windows.

Both layers share the same design: a single abstract `IBootBackend` interface,
`(ok, value)` result tuples rather than exceptions, automatic backups before
every write, and a full undo stack.

---

## What Works Where

| Feature | Python (FLTK) | Python (browser) | C++ ImGui | C++ Nuklear |
|---|---|---|---|---|
| List entries | ✅ | ✅ | ✅ | ✅ |
| Set default | ✅ | ✅ | ✅ | ✅ |
| Rename | ✅ | ✅ | ✅ | ✅ |
| Move up / down | ✅ | ✅ | ✅ | ✅ |
| Delete | ✅ | ✅ | ✅ | ✅ |
| Undo | ✅ | ✅ | ✅ | ✅ |
| Demo mode | ✅ | ✅ | ✅ | ✅ |
| No GPU required | ✅ | ✅ | ❌ | ✅ |
| No browser required | ✅ | ❌ | ✅ | ✅ |

---

## Supported Platforms

| OS | Bootloader | Support | Notes |
|---|---|---|---|
| Linux | GRUB2 | ✅ Full | Parses grub.cfg, writes GRUB_DEFAULT, runs grub-mkconfig |
| Linux | systemd-boot | ✅ Full | Reads/writes .conf files and loader.conf directly |
| Windows | BCD store | ✅ Full | Wraps bcdedit.exe — requires Administrator |
| macOS (Intel) | bless | ⚠️ Partial | bless --setBoot — limited control |
| macOS (Apple Silicon) | recoveryOS | 📖 Read-only | Sealed system volume — no direct EFI access |

---

## Architecture

```
+---------------------------------------+
|   FLTK native window                  |  Python (default — Windows / macOS)
|   OR Nuklear + SDL2 window            |  C++
|   OR Dear ImGui + OpenGL window       |  C++
|   OR Browser GUI  (gui.html)          |  http://localhost:7373  (--browser)
+------------------+--------------------+
                   | In-process calls (Python)
                   | JSON IPC on stdin/stdout (C++)
+------------------v--------------------+
|         IBootBackend (abstract)       |
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

**Key design principles**

- **Single abstract interface.** Every bootloader backend implements `IBootBackend`
  with the same six methods: `list_entries`, `set_default`, `delete_entry`,
  `rename_entry`, `move_entry`, `rescan`. The GUI never knows which backend is running.

- **Result tuples instead of exceptions.** Every operation returns `(True, value)` on
  success or `(False, error_string)` on failure. Errors are always surfaced to the
  user as readable messages — never raw tracebacks.

- **Undo stack.** Before every write, a precise reversal is pushed onto an undo stack
  (up to 10 entries). The GUI revert button label shows exactly what will be undone.

- **Backup before every write.** `Backup.save()` is called before touching any file
  or running any system command. Backups are timestamped copies in
  `~/.config/bootload-modifier/backups/`.

- **Demo mode.** `MockBackend` implements the full interface with realistic in-memory
  data. All GUI features including undo work in demo mode — nothing is written to disk.

- **JSON IPC (C++ backend).** The compiled binary communicates over newline-delimited
  JSON on stdin/stdout, which lets it be driven by any frontend or tested headlessly.

---

## File Structure

```
bootload-modifier/
│
├── python/                   Python app — runs today, pip install pyFLTK
│   ├── blm.py                Entry point
│   ├── backend.py            All bootloader backends + undo system (1203 lines)
│   ├── server.py             HTTP API server (browser GUI, --browser flag)
│   ├── gui.html              Browser-based GUI (1269 lines)
│   ├── gui_fltk.py           FLTK native GUI (default on Windows / macOS)
│   ├── run.bat / run.sh      Launch — auto-installs pyFLTK if missing
│   └── build.bat / build.sh  Build standalone executable via PyInstaller
│
├── cpp/                      C++ IPC backend (blm-backend)
│   ├── include/              Headers for all backends + utilities
│   ├── src/                  Implementations
│   │   ├── backends/         Grub2, SystemdBoot, WindowsBcd, macOS
│   │   └── utils/            Logger, Backup
│   ├── CMakeLists.txt
│   └── build.bat / build.sh
│
├── gui/                      Dear ImGui + OpenGL 3.3 / 2.1 GUI
│   ├── src/main.cpp
│   ├── src/IpcClient.hpp
│   ├── third_party/imgui/
│   ├── CMakeLists.txt
│   └── build.bat / build.sh  (pass --classic for OpenGL 2.1 fallback)
│
├── gui-nuklear/              Nuklear + SDL2 GUI (no dedicated GPU required)
│   ├── src/main.cpp
│   ├── src/IpcClient.hpp
│   ├── third_party/nuklear/
│   ├── CMakeLists.txt
│   ├── build_linux.sh        Linux one-click build (auto-installs SDL2)
│   ├── build.sh              Linux / macOS manual build
│   ├── build.bat             Windows (vcpkg SDL2)
│   └── build_novcpkg.bat     Windows — downloads SDL2 zip, no admin needed
│
├── stress/                   Headless test suites (123 tests, zero mocking)
│   ├── stress_test.cpp       Round 1 — 65 tests
│   ├── stress_test2.cpp      Round 2 — 58 tests
│   └── run.sh                Build and run both suites against real binary
│
├── setup_windows.bat         Full Windows C++ toolchain setup (one-click)
├── diagnose_windows.bat      System diagnostic for Windows build issues
├── CHANGES.md                Complete implementation history and all bug fixes
├── EXPLANATION.md            Plain-language guide to the project decisions
└── README.md                 This file
```

**Total codebase: ~5,000 lines** (excluding `third_party/`)

---

## Quick Start

### Python — easiest, runs today, zero build step

**Windows / macOS**
```bat
pip install pyFLTK
python\run.bat          # Windows
./python/run.sh         # macOS
```
On Windows, right-click → *Run as administrator* for write access to boot entries.
Pass `--demo` for safe testing without admin rights.

**Linux**
```bash
# No prebuilt pyFLTK wheel on Linux — use the browser GUI instead
chmod +x python/run.sh
./python/run.sh --demo      # safe, no root needed
sudo ./python/run.sh        # real mode
# (falls back automatically to --browser if pyFLTK is not installed)
```

---

## Python App

### Run from source

```bash
# Install the one dependency (prebuilt wheel — no compiler needed on Windows/macOS)
pip install pyFLTK

# Launch
python blm.py               # FLTK GUI   (native window, default)
python blm.py --browser     # browser GUI (opens in your browser, no pyFLTK)
python blm.py --demo        # demo mode   (no root, no writes to disk)
python blm.py --help        # all options
```

### Build a standalone executable

```bat
REM Windows
python\build.bat
REM Output: python\dist\BootloadModifier.exe
```

```bash
# macOS
cd python && ./build.sh
# Output: python/dist/BootloadModifier
```

### Features

- Auto-detection of bootloader on startup (GRUB2, systemd-boot, Windows BCD, macOS)
- View all boot entries with `[default]` and `[ro]` tags
- Change default boot entry with confirmation dialog
- Rename entries — with full undo
- Delete old entries — with guards and undo support
- Reorder entries — with undo
- Full undo stack — every write operation records a precise reversal
- Settings persistence — saved to disk, restored on every launch
- Demo mode — full GUI, realistic mock data, nothing written to disk
- Automatic backup before every write (`~/.config/bootload-modifier/backups/`)
- Windows Update warning — detects a pending reboot and warns before changes
- Operation log at `~/.config/bootload-modifier/blm.log`
- Dark theme
- Port conflict handling — tries ports 7373–7382 if default is in use

### FLTK GUI overview

The FLTK GUI has 3 tabs: **Boot Entries**, **Settings**, **Log**.

The **Boot Entries** tab is the main view. Entry tags use plain ASCII:

| Tag | Meaning |
|---|---|
| `[default]` | Current default boot target |
| `[ro]` | Read-only — cannot be modified |

Button order (all equal-width, filling the full window width):
`Rescan | Set Default | Rename | Up | Down | Undo | Delete`

The **Settings** tab has four sections:

| Section | Contents |
|---|---|
| General Settings | Confirm dialogs • Create backups • Show technical metadata |
| Mode & Status | Live vs demo indicator; privilege status (write access / run as admin) |
| File Paths | Log file path; backups path (platform-aware) |
| Backend & Platform | Backend name and kind; full bootloader support matrix |

All settings are saved automatically and restored on every launch.
A **Reset to defaults** button at the bottom of the Settings tab restores all three
checkboxes to their default state.

---

## C++ Stack

Three binaries. Place all three in the same directory to run.

| Binary | Source | Purpose |
|---|---|---|
| `blm-backend` | `cpp/` | IPC backend — spawned by both GUIs |
| `blm-gui` | `gui/` | Dear ImGui + OpenGL 3.3 GUI |
| `blm-gui-nuklear` | `gui-nuklear/` | Nuklear + SDL2 GUI (no GPU required) |

The C++ Nuklear GUI matches the FLTK GUI feature-for-feature: 3 tabs
(Boot Entries, Settings, Log), 3 settings checkboxes, Reset button,
detail panel respects the "Show technical metadata" toggle.

### Linux / macOS

**One-click build (recommended):**
```bash
chmod +x gui-nuklear/build_linux.sh
./gui-nuklear/build_linux.sh
# Auto-installs SDL2 via apt/dnf/pacman/brew, builds both binaries, puts them in run/
./run/blm-gui-nuklear --demo    # no root
sudo ./run/blm-gui-nuklear      # real mode
```

**Manual build:**
```bash
# 1. Build the backend (no external deps)
cd cpp && ./build.sh

# 2a. Nuklear GUI
#     Ubuntu/Debian: sudo apt install libsdl2-dev
#     Fedora:        sudo dnf install SDL2-devel
#     macOS:         brew install sdl2
cd ../gui-nuklear && ./build.sh

# 2b. ImGui GUI (alternative — needs OpenGL)
#     Ubuntu/Debian: sudo apt install libglfw3-dev libgl-dev
#     macOS:         brew install glfw
cd ../gui && ./build.sh             # OpenGL 3.3 core (modern)
cd ../gui && ./build.sh --classic   # OpenGL 2.1 (old GPUs, VMs)
```

### Windows

**Easiest — no vcpkg, no admin rights:**

```
1. Run setup_windows.bat
   — installs C++ compiler and cmake if missing
2. Double-click gui-nuklear\build_novcpkg.bat
   — downloads SDL2 automatically, builds both binaries, puts them in run\
3. run\blm-gui-nuklear.exe --demo
```

**With vcpkg:**
```
gui-nuklear\build.bat    (installs SDL2 via vcpkg then builds)
```

### C++ binary options

```
blm-backend --demo --list     list mock entries, no root needed
blm-backend --ipc             JSON IPC mode (used by GUIs)
blm-backend --info            print detected platform info
blm-backend --version         print version
blm-gui-nuklear --demo        Nuklear GUI, demo mode
blm-gui --demo                ImGui GUI, demo mode
```

### C++ IPC protocol

The backend communicates over newline-delimited JSON on stdin/stdout:

```json
{ "id": 1, "action": "listEntries" }
{ "id": 2, "action": "setDefault",  "entryId": "grub-0" }
{ "id": 3, "action": "deleteEntry", "entryId": "grub-2" }
{ "id": 4, "action": "renameEntry", "entryId": "grub-1", "name": "Arch Linux" }
{ "id": 5, "action": "moveEntry",   "entryId": "grub-1", "delta": -1 }
{ "id": 6, "action": "rescan" }
{ "id": 7, "action": "backendInfo" }
{ "id": 8, "action": "quit" }
```

---

## Running the Stress Tests

```bash
cd stress
./run.sh                            # builds and runs both suites
./run.sh ../cpp/build/blm-backend   # explicit backend path

# Expected: 65/65 + 58/58 = 123/123 passed — ALL PASSED ✓
```

The tests spawn real `blm-backend` processes and communicate over pipes using
the same JSON IPC protocol as the GUI — zero mocking. Coverage includes JSON
parser edge cases, all entry operations and their error paths, state consistency
after multi-step sequences, rapid-fire stress, concurrency, and latency benchmarks.

---

## Safety & Stability

- **Automatic backups** — timestamped copies of every config file before any write
- **Full undo stack** — set-default, rename, delete, and move are all reversible
- **Irreversible operation handling** — BCD delete cannot be undone; the undo button
  explains this clearly and provides manual recovery steps
- **Windows Update warning** — detects pending reboot via registry and warns before changes
- **Confirmation dialogs** — both set-default and delete require confirmation (toggleable)
- **Guards against dangerous operations** — cannot delete the current default, cannot
  modify read-only entries, cannot set UEFI firmware entries as default
- **Demo mode** — full GUI with no system access, safe for testing and exploration
- **Input validation** — empty names rejected, boundary checks on reorder, delta=0 rejected
- **Graceful failure** — all errors shown as readable messages in the GUI
- **Settings resilience** — corrupted or missing settings file silently falls back to defaults
- **Port conflict recovery** — finds an available port if 7373 is taken
- **Localhost only** — the HTTP API server binds to 127.0.0.1, never network-accessible

---

## Known Limitations

| Item | Status | Notes |
|---|---|---|
| GRUB2 `move_entry` | ⚠️ Returns actionable error | Auto-generated entries cannot be reordered without patching grub-mkconfig |
| macOS Intel `bless` | ⚠️ Stub | Needs Mac hardware for full implementation |
| macOS Apple Silicon | ⚠️ Read-only | Sealed by design |
| C++ Undo IPC action | ⚠️ Partial | Backend has full undo stack; GUI Undo triggers rescan instead of true undo |
| pyFLTK on Linux | ⚠️ No prebuilt wheel | Requires manual build with SWIG 4.1 + libfltk1.3-dev; `run.sh` falls back to `--browser` automatically |
| Packaging installers | ⚠️ Future | .deb, .rpm, AppImage, NSIS, .dmg |
| Config diff viewer | ⚠️ Future | Show exact changes before applying |

---

## Changelog

See [`CHANGES.md`](CHANGES.md) for the complete implementation history, including
all 67 bug fixes across 34 sessions, session-by-session feature additions, and
detailed root cause analyses for every significant issue.

**Major additions since initial release:**

- FLTK native GUI replaces pygame as the default Python GUI
- pygame fully removed from the codebase
- C++ Nuklear GUI brought to full parity with the FLTK GUI (3 tabs, 3 settings
  checkboxes, Reset button, metadata toggle in detail panel)
- Platform tab merged into Settings tab across all GUIs
- `build_linux.sh` — one-click Linux build script with auto-SDL2 install
- Bug 67 fixed: FLTK support matrix showed garbage text on Windows (dangling
  pointer from temporary Python strings passed to `Fl_Box`)
- 67 bugs fixed total (full index in CHANGES.md)

Also see [`EXPLANATION.md`](EXPLANATION.md) for a plain-language guide to the
project's design decisions, the hardest problems encountered, and key numbers.

---

## Roadmap

**v0.2.0 — Packaging and polish**
- Windows NSIS installer with UAC manifest
- Linux AppImage, .deb, .rpm
- macOS .dmg
- GRUB2 entry reordering via /etc/grub.d/ script prefix numbering
- systemd-boot entry reordering already implemented
- macOS Intel — full bless implementation
- UEFI variable editing via efibootmgr
- Multiple undo steps shown in a history panel

**Future ideas**
- Config diff viewer — show exactly what will change before applying
- Automatic detection of orphaned entries (partition no longer exists)
- Export / import boot configuration as JSON
- Flatpak / Homebrew packaging

---

## Contributing

Contributions, issues, and pull requests are welcome.

```bash
# Fork the repository
git checkout -b feature/my-feature
python3 blm.py --demo     # test in demo mode first
# open a pull request with a clear description of what changed and why
```

**Ground rules**

- Never write to real boot config without an explicit user action
- Always call `Backup.save()` before any write operation
- All user-facing errors must be human-readable — no raw exceptions
- Test in demo mode first, real mode second
- Document what your code does and why, not just what it does

**Areas where help is most welcome**

- macOS testing — especially on Intel Macs
- Packaging (AppImage, Flatpak, Homebrew, NSIS)
- Automated Python test suite for backend logic
- Full macOS Intel `bless` implementation
- pyFLTK Linux wheel (would remove the last platform gap in the Python GUI)

---

## Project Status

Active development. Fully functional Python core with FLTK native GUI (Windows/macOS)
and browser GUI (Linux fallback). C++ stack compiles and runs on Linux and Windows
with two native GUI frontends. **123/123 stress tests passing.**

Total codebase: ~5,000 lines (excluding `third_party/`)

---

## License

GPL-3.0 — see LICENSE file.

Built with Python 3.8+. C++ backend targets C++20 with CMake 3.20+.

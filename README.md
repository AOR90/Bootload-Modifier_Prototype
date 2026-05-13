# Bootload Modifier (BLM)

## 🚀 Description

Bootload Modifier (BLM) is a cross-platform bootloader management tool that lets advanced users safely view, modify, reorder, and manage boot entries through a clean and intuitive GUI. Designed for Linux distro hoppers and IT professionals working with multi-boot systems.

---

## 🧾 Overview

Bootload Modifier (BLM) is a system utility designed to simplify bootloader management through a modern graphical interface. It provides fine-grained control over boot entries across multiple operating systems, while maintaining safety through backups, validation, and a structured execution model.

The tool is built for advanced users who work with multi-boot environments and need reliable control over boot configuration systems such as GRUB, systemd-boot, and Windows Boot Manager.

---

## ⚠️ Disclaimer

This tool is intended for experienced users only. Incorrect usage may affect system bootability. Administrative/root privileges are required for many operations.

---

## ✨ Features

* View all boot entries across supported systems
* Change default boot entry
* Rename entries
* Delete entries (with safety checks)
* Reorder boot entries
* Cross-platform support (Linux, Windows, partial macOS)
* Demo mode for safe testing without system changes
* Automatic backup system before modifications
* Logging system for tracking all operations

---

## 🏗️ Architecture

### Backend

* Python prototype (HTTP-based local server + GUI)
* Planned C++ production backend (higher performance, system-level integration)

### Supported Boot Systems

* GRUB2 (Linux)
* systemd-boot (Linux)
* Windows Boot Manager (BCD)
* macOS (limited / read-only support)

---

## 🖥️ GUI

* Web-based interface (HTML + JavaScript)
* Two-panel layout (entries + details)
* Settings panel for configuration
* Live status and logging view
* Responsive and minimal design for clarity

---

## 🧪 Safety & Stability

* Automatic backups before any changes
* Input validation and error handling
* Demo mode to prevent accidental system modification
* Graceful failure handling for unsupported systems

---

## 🚧 Roadmap

* Complete C++ backend implementation
* Replace Python prototype with native binary
* Native GUI (Qt or Tauri)
* Improved EFI-level boot control
* Expanded Linux bootloader support
* Better cross-platform abstraction layer

---

## 📦 Installation / Usage

### Demo Mode (Safe)

```bash
python blm.py --demo
```

### Linux (Real Mode)

```bash
sudo python blm.py
```

---

## 🤝 Contributing

Contributions, issues, and pull requests are welcome. This project is under active development and open to improvements.

---

## 🔥 Project Status

Prototype stage — functional core system with active development and ongoing backend migration toward C++.

________________________________________
If needed, this can be further refined into a production-level GitHub README or expanded into full developer documentation.

# Wolf Optimizer — Future Plan

This document tracks planned features beyond v1, organized by theme. Items are not commitments — they are a directional roadmap to be prioritized against user feedback.

---

## v1.x — Polish & Stability (post-launch)

### Rule Improvements
- Expand built-in rule library based on real user profiles
- Per-rule impact tracking: did applying this rule actually help? (measure before/after)
- Rule confidence scores based on how much data backs the recommendation

### UX
- Mouse support in TUI (optional, keyboard remains primary)
- Notification system: desktop notification via `notify-send` when recommendations are ready
- Wolf status bar widget for tmux / shell prompt integration
- `wolf report` command: export weekly profile summary as plain text

### Packaging
- AUR package for Arch Linux
- Flatpak (limited functionality — no kernel param support due to sandbox)
- Snap (same caveat)

---

## v2 — Community Rules & Plugin System

### Plugin Architecture
- TOML rule files loadable from `~/.config/wolf-optimizer/rules/`
- Rule installer: `wolf rules install <path-or-url>`
- Rule sandbox: declarative TOML schema prevents arbitrary code execution
- Rules can only request whitelisted action types — daemon still enforces this regardless
- Rule signing: GPG-signed rule packages from a Wolf community registry
- `wolf rules list` — show installed rules, their author, risk levels, and last triggered date

### Community
- Official rule registry (GitHub-hosted, community-contributed)
- Rule submission process with safety review checklist
- Rule categories: developer, gamer, media, server, battery-saver

---

## v3 — macOS Support

### Platform Abstraction Layer
- Abstract all Linux-specific APIs (`/proc`, `sysctl`, systemd) behind a platform trait
- macOS implementation: `libproc` for process info, `launchd` instead of systemd, `IOKit` for power/CPU
- macOS-specific rules: memory pressure tuning, App Nap management, Spotlight index control
- `.pkg` installer for macOS

### Collector
- macOS: use `mach` APIs for process sampling instead of `/proc`
- Activity Monitor integration data

---

## v4 — Windows Support

### Platform
- Windows implementation of platform abstraction layer
- Win32 API / WMI for process and performance data
- Windows-specific rules: prefetch tuning, hibernation file management, temp file locations
- NSIS or WiX installer
- Windows service instead of systemd for collector and daemon

---

## v5 — Advanced Intelligence

### Enhanced Profiling
- Per-application resource forecasting: predict RAM spikes before they happen
- Workload pattern detection: "you typically start a heavy compile at 10am — pre-allocate resources"
- Anomaly detection: alert when a process is behaving unusually (memory leak, runaway CPU)

### Adaptive Optimization
- Time-aware rules: apply aggressive optimizations during known idle windows, revert for active periods
- Battery-aware mode: different optimization profiles for AC vs battery
- Thermal-aware: back off CPU optimizations when thermals are high

### ML (if warranted by data)
- Local, on-device model only — no data leaves the machine
- Train on user's own 90-day history to improve recommendation confidence
- Explicit opt-in, clearly explained to user

---

## Long-term / Exploratory

- **wolf-server**: optional local HTTP API for scripting and third-party integrations
- **wolf-web**: optional local web UI (accessible at localhost) as alternative to TUI
- **Multi-profile support**: switch between "work", "gaming", "battery-saver" profiles manually
- **Scheduled optimization windows**: "apply aggressive optimizations every night from 2-4am"
- **Container awareness**: detect Docker/Podman workloads and adjust recommendations accordingly
- **ZFS/btrfs integration**: snapshot before optimization using filesystem snapshots as rollback mechanism

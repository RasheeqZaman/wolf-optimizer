# Wolf Optimizer — v1 Requirements

## Scope
v1 targets Ubuntu 22.04+ (x86_64). The goal is a working, safe, installable tool that delivers genuine value from day one.

---

## Functional Requirements

### FR-01: Activity Collection
- [ ] `wolf-collector` runs as a user-space systemd service
- [ ] Samples every 30 seconds: PID, process name, CPU%, RAM (MB), disk read/write bytes
- [ ] Writes samples to SQLite at `~/.local/share/wolf-optimizer/activity.db`
- [ ] Weekly aggregation job builds profile table from raw samples
- [ ] Prunes raw data older than 14 days automatically
- [ ] Collector survives process crashes and restarts without data corruption

### FR-02: Profile Building
- [ ] Aggregates raw telemetry into per-process weekly profile: avg CPU, peak RAM, run frequency, last seen
- [ ] Auto-detects user category: developer, general (v1 supports these two)
- [ ] Developer detection: presence of rustc, gcc, node, python, git in process history
- [ ] Profile stored in `profiles` table in activity.db
- [ ] Bootstrap mode: on first run, performs a one-time scan of whitelisted paths to generate cold profile
- [ ] Bootstrap profile labeled as "estimated — improves with weekly data"

### FR-03: Recommendation Engine
- [ ] Rule-based engine reads profile and produces a ranked list of recommendations
- [ ] Each recommendation contains: action, evidence (which metric triggered it), expected impact, risk level
- [ ] Risk levels: `safe`, `moderate`, `requires_confirmation`
- [ ] Built-in rules for v1 (see FR-07)
- [ ] Rule engine is separate from execution — recommendations are always generated before any action is taken

### FR-04: Optimization Execution
- [ ] Dry-run mode: show all planned changes with before/after values, no execution
- [ ] User must explicitly confirm before any changes are applied
- [ ] Kernel parameter changes require a second confirmation step regardless of risk level
- [ ] Session snapshots all pre-change values to rollback file before first action
- [ ] On any failure: halt execution, rollback all changes in session, report to user
- [ ] If rollback fails: show manual restore commands, write `emergency-restore.sh`
- [ ] Audit log entry written for every action (success or failure) with timestamp

### FR-05: Storage Cleaning
- [ ] Scans only whitelisted paths (see Safety Requirements)
- [ ] Reports size and last-accessed date for each category
- [ ] Categories: package caches, thumbnail cache, user cache, trash, temp files, old logs
- [ ] Profile-aware: does NOT recommend cleaning cargo registry if user compiles Rust frequently
- [ ] User selects which categories to clean, previews exact files/dirs before deletion
- [ ] Deletion is permanent — user must confirm with explicit acknowledgment

### FR-06: TUI
- [ ] Built with `ratatui`
- [ ] Screens: Dashboard, Activity Explorer, Optimize, Clean, Settings
- [ ] Dashboard: live CPU/RAM/disk stats (2s refresh), profile summary, last optimization summary
- [ ] Activity Explorer: per-app weekly breakdown (avg CPU, peak RAM, run count)
- [ ] Optimize: dry-run preview → confirm → live progress → rollback option
- [ ] Clean: scan results by category → select → preview → confirm → execute
- [ ] Settings: notification toggle, schedule config, audit log viewer
- [ ] Keyboard navigation only (no mouse required)
- [ ] Color theme: works on both dark and light terminal backgrounds

### FR-07: Built-in Rules (v1 whitelist)
- [ ] `low-swappiness`: reduce vm.swappiness if avg RAM usage < 40% of total
- [ ] `keep-cargo-cache`: suppress cargo cache cleaning if rustc runs > 5x/week
- [ ] `clean-thumbnails`: recommend thumbnail cache cleanup if > 500MB and image viewer rarely used
- [ ] `clean-npm-cache`: recommend npm cache cleanup if npm not used in 30 days
- [ ] `clean-old-logs`: recommend /var/log cleanup for logs > 30 days old
- [ ] `nice-background-apps`: renice rarely-used background processes to +10
- [ ] `clean-trash`: recommend trash empty if trash > 1GB and items older than 7 days

### FR-08: Scheduling
- [ ] Collector runs continuously (user systemd service)
- [ ] Weekly analysis runs automatically: Sunday, midnight, only if system idle > 10 minutes
- [ ] Recommendations prepared and stored — not applied — until user confirms
- [ ] Optional: write a one-line summary to `/etc/motd.d/wolf` after analysis (opt-in in settings)

### FR-09: Installation
- [ ] Distributed as `.deb` package (built via `cargo-deb`)
- [ ] Package installs: both binaries, systemd units, man pages, wolf-optimizer group
- [ ] Post-install script enables collector service for installing user
- [ ] `sudo usermod -aG wolf-optimizer $USER` documented in README and shown post-install
- [ ] Clean uninstall: `apt remove wolf-optimizer` leaves user data intact; `apt purge` removes everything

---

## Non-Functional Requirements

### NFR-01: Safety
- Daemon binary executes only operations on a hardcoded whitelist
- No arbitrary sysctl writes under any code path
- TUI process never holds root privileges
- Daemon is socket-activated (not always running)
- Daemon rejects any request from callers not in `wolf-optimizer` group

### NFR-02: Performance
- Collector uses < 1% CPU on average (30s sampling interval)
- Collector uses < 20MB RAM
- TUI startup time < 1 second
- Full scan (bootstrap or clean scan) completes in < 60 seconds on a typical home directory

### NFR-03: Reliability
- Collector survives crashes and restarts without corrupting the SQLite database (WAL mode)
- All file operations are atomic where possible (write to temp, then rename)
- Wolf never deletes a file without the user having seen its path and size first

### NFR-04: Privacy
- No data leaves the machine. Ever. No telemetry, no analytics, no network calls.
- Wolf never reads file contents — only names, sizes, and access times
- Wolf never reads `~/.ssh/`, `~/.gnupg/`, or `~/.config/` contents

### NFR-05: Compatibility
- Ubuntu 22.04 LTS (Jammy) and 24.04 LTS (Noble)
- Rust 1.75+
- systemd 249+

---

## Out of Scope for v1
- macOS and Windows support
- Community/plugin rule system
- ML-based profile inference
- GUI (non-terminal) interface
- Network optimization
- Auto-apply of kernel params without confirmation
- Multi-user support

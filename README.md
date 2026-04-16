# Wolf Optimizer

A smart, profile-driven PC performance optimizer and storage cleaner for Linux — built in Rust.

Wolf watches how you use your machine for a week, builds a behavioral profile, then recommends targeted optimizations and cleaning actions tailored specifically to you. A developer gets different recommendations than a gamer. Your rarely-used apps get cleaned differently than your daily drivers.

---

## Features

- **Weekly activity profiling** — tracks process usage, CPU/RAM consumption, disk access patterns, and idle periods
- **Profile-driven recommendations** — rule-based engine explains every recommendation with evidence from your actual usage
- **Kernel-level optimization** — safely tunes `vm.swappiness`, CPU scheduler parameters, and more via a privileged daemon
- **Smart storage cleaning** — identifies cache bloat, old logs, and cold files based on your actual access patterns
- **Interactive TUI** — clean terminal UI with live system stats, heatmaps, dry-run previews, and one-key rollback
- **Bootstrap mode** — useful recommendations from the first run, improving with each week of data

## Safety First

Wolf is designed so that a bug cannot crash your OS:

- A separate, minimal privileged daemon executes only a hardcoded whitelist of safe kernel operations
- Every optimization session snapshots current values before making any changes
- Any failure triggers automatic rollback of all changes in that session
- Kernel parameter changes always require explicit user confirmation — no exceptions
- Full audit log of every action taken

## Installation

```bash
sudo apt install wolf-optimizer
sudo usermod -aG wolf-optimizer $USER
# Log out and back in, then:
wolf
```

## Architecture

```
wolf-collector  →  activity.db  ←  wolf-optimizer (TUI)
(user service)     (SQLite)              ↓ Unix socket
                                   wolf-daemon (privileged, socket-activated)
```

- **wolf-collector**: user-space systemd service, samples activity every 30 seconds
- **wolf-optimizer**: ratatui TUI, runs as your user, reads your profile, shows recommendations
- **wolf-daemon**: socket-activated privileged daemon, executes only whitelisted kernel operations

## Usage

```bash
wolf              # open TUI dashboard
wolf optimize     # jump to optimize screen
wolf clean        # jump to clean screen
wolf status       # print profile summary to stdout
wolf dry-run      # print recommendations without opening TUI
```

## Requirements

- Linux (Ubuntu 22.04+ recommended)
- systemd
- 10MB disk space for the application
- ~50MB disk space for activity database (auto-pruned after 2 weeks)

## Building from Source

```bash
git clone https://github.com/yourusername/wolf-optimizer
cd wolf-optimizer
cargo build --release
```

Requires Rust 1.75+.

## License

WolfTech

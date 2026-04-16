# Wolf Optimizer — Personal Study Guideline

**Format:** Sprint-gated + checklist-heavy
**Pace:** 10–20 hrs/week
**Starting level:** Rust beginner, Linux systems unfamiliar
**Structure:** 2–3 week Rust foundations phase, then per-sprint study gates

---

## How to Use This Guide

Each section has three parts:
1. **Study Gate** — what to learn *before* you start the sprint tasks
2. **Warmup Exercise** — a small throwaway project to verify you're ready
3. **Sprint Checklist** — the actual sprint tasks from the timeline

Do not start sprint tasks until the Study Gate is cleared. The Warmup Exercise is the proof.

---

## Phase 0 — Rust Foundations (Weeks 1–3)

Before any sprint work. Linux concepts are skipped here — you'll learn them in context.

### Read
- [ ] [The Rust Book](https://doc.rust-lang.org/book/) — Chapters 1–9
  - Ch 1–2: Installation, Hello World, Cargo
  - Ch 3: Variables, data types, functions, control flow
  - Ch 4: Ownership ← **most important chapter, re-read until solid**
  - Ch 5: Structs
  - Ch 6: Enums and `match`
  - Ch 7: Modules and `use`
  - Ch 8: Collections (`Vec`, `HashMap`)
  - Ch 9: Error handling with `Result`, `Option`, and `?`

### Build
- [ ] **Toy project: `rusty-notes`** — a CLI note-taking app
  - Takes a subcommand (`add`, `list`, `delete`)
  - Stores notes in a plain text file
  - Uses structs, enums, `match`, file I/O, and `Result` with `?`
  - No `unwrap()` allowed — handle every error explicitly
  - Goal: get comfortable with the ownership model and Cargo before touching the real codebase

### Gate Criteria
- [ ] You can explain ownership and borrowing without looking it up
- [ ] You can write a `struct` with `impl` methods
- [ ] You can use `?` to propagate errors through a function returning `Result<_, _>`
- [ ] `cargo build`, `cargo test`, `cargo clippy` all pass on your toy project

---

## Sprint 1 — Project Scaffold & Core Data Layer

### Study Gate

**Rust topics to cover before starting:**
- [ ] [The Rust Book Ch 14](https://doc.rust-lang.org/book/ch14-00-more-about-cargo.html) — Cargo workspaces (multiple crates in one repo)
- [ ] [The Rust Book Ch 10](https://doc.rust-lang.org/book/ch10-00-generics.html) — Traits and generics (needed for shared types in `wolf-core`)
- [ ] Skim the [`rusqlite` getting started guide](https://docs.rs/rusqlite/latest/rusqlite/) — understand `Connection`, `execute`, `query_map`
- [ ] Read what WAL mode is in SQLite: search "SQLite WAL mode" — one short article is enough

**Linux topics to cover before starting:**
- [ ] Read: [The /proc Filesystem (kernel docs)](https://docs.kernel.org/filesystems/proc.html) — focus on `/proc/[pid]/stat` and `/proc/[pid]/status`
- [ ] In your terminal, run: `cat /proc/self/status` and `cat /proc/1/stat` — understand what each field means
- [ ] Read: what is a systemd user service (vs system service)? Search "systemd user service tutorial" — one article

### Warmup Exercise
- [ ] **`proc-peek`**: Write a Rust binary (single file, no workspace) that reads `/proc/self/status`, parses the `VmRSS` (RAM) and `Name` fields, and prints them. Use `std::fs::read_to_string` and string splitting. No external crates.
- [ ] **`sqlite-toy`**: Write a Rust binary that creates a SQLite DB, inserts 3 rows into a `notes` table, and prints them back. Use `rusqlite`. Enable WAL mode with `PRAGMA journal_mode=WAL`.

### Sprint Checklist
- [ ] Initialize Cargo workspace with 4 crates: `wolf-core`, `wolf-collector`, `wolf-daemon`, `wolf-tui`
- [ ] Define shared types in `wolf-core`: `ProcessSample`, `AppProfile`, `Recommendation`, `RiskLevel`
- [ ] SQLite schema: `samples` table, `profiles` table, `audit_log` table
- [ ] Database initialization and migration logic in `wolf-core`
- [ ] `wolf-collector` binary: reads `/proc` every 30s, writes `ProcessSample` to DB
- [ ] WAL mode enabled on SQLite for crash safety
- [ ] Unit tests: schema creation, sample insert/read, duplicate handling
- [ ] CI: GitHub Actions pipeline compiles all crates on push

---

## Sprint 2 — Privileged Daemon (Safety-Critical)

### Study Gate

**Rust topics to cover before starting:**
- [ ] [Tokio tutorial — Hello Tokio](https://tokio.rs/tokio/tutorial/hello-tokio) — understand `async fn`, `.await`, `#[tokio::main]`
- [ ] [Tokio tutorial — Spawning](https://tokio.rs/tokio/tutorial/spawning) — `tokio::spawn` for concurrent tasks
- [ ] [`serde` + `serde_json` basics](https://serde.rs/) — serializing/deserializing structs to JSON; understand `#[derive(Serialize, Deserialize)]`
- [ ] Read: [Rust error handling in libraries](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) — understand when to use `panic!` vs `Result` (in the daemon: never `unwrap()`)

**Linux topics to cover before starting:**
- [ ] Read: "Unix domain sockets explained" — search this phrase, pick any clear article. Understand: server binds a socket file, client connects to it, they exchange bytes
- [ ] In your terminal: `ls -la /run/` — understand what `/run/` is (tmpfs, cleared on reboot)
- [ ] Read: "Linux groups and permissions" — understand `addgroup`, `usermod -aG`, and how file/socket permissions enforce group membership
- [ ] Read: "systemd socket activation" — search this phrase. Understand the `.socket` + `.service` unit pair pattern
- [ ] Run in your terminal: `sysctl vm.swappiness` and `sysctl -w vm.swappiness=60` (as root in a VM, not your main machine) — understand what sysctl does

### Warmup Exercise
- [ ] **`sock-ping`**: Write two Rust binaries — a server and a client — that communicate over a Unix domain socket at `/tmp/sock-ping.sock`. Server receives a JSON message `{"msg": "ping"}` and replies `{"msg": "pong"}`. Use `tokio::net::UnixListener` and `serde_json`. This is the exact pattern the daemon uses.
- [ ] **`group-check`**: Write a Rust snippet that reads the current process's groups (use `nix` crate or parse `/proc/self/status`) and prints whether the process is in a given group. This is how the daemon will enforce `wolf-optimizer` group membership.

### Sprint Checklist
- [ ] `wolf-daemon` binary skeleton with socket-activated systemd unit
- [ ] Unix domain socket setup at `/run/wolf-optimizer/daemon.sock`
- [ ] `wolf-optimizer` group enforcement: reject callers not in group
- [ ] Session token generation and validation
- [ ] Hardcoded kernel param whitelist (`vm.swappiness`, `vm.dirty_ratio`, `vm.dirty_background_ratio`, `kernel.sched_migration_cost_ns`)
- [ ] Request handler: parse JSON op, validate against whitelist, reject unknown ops
- [ ] Snapshot mechanism: read current sysctl value before any write
- [ ] Rollback mechanism: restore all snapshotted values on command or error
- [ ] Audit log writer: every op logged with timestamp, caller, before/after values
- [ ] Unit tests: whitelist enforcement, rollback logic
- [ ] Integration test (QEMU): daemon starts, accepts valid request, rejects invalid request, rollback restores values

---

## Sprint 3 — Profile Engine & Rule System

### Study Gate

**Rust topics to cover before starting:**
- [ ] [The Rust Book Ch 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html) — Iterators and closures (`map`, `filter`, `fold`, `collect`)
- [ ] [`toml` crate basics](https://docs.rs/toml/latest/toml/) — deserializing a TOML file into a Rust struct
- [ ] Read about Rust's `chrono` or `std::time` — understand how to compare timestamps and compute durations (needed for "last seen 14 days ago" logic)

**Linux topics to cover before starting:**
- [ ] No new Linux topics required for this sprint — focus is pure Rust logic

### Warmup Exercise
- [ ] **`toml-rules`**: Write a Rust binary that loads a TOML file with this shape:
  ```toml
  [[rule]]
  id = "low-swappiness"
  condition = "swappiness_above"
  threshold = 60
  ```
  Deserializes it into a `Vec<Rule>` struct, then prints each rule's `id`. This is the exact pattern the rule engine uses.
- [ ] **`profile-agg`**: Write a function (with unit tests) that takes a `Vec<ProcessSample>` and returns average CPU%, peak RAM, and how many times each process name appeared. Use iterators only — no manual `for` loops.

### Sprint Checklist
- [ ] Profile aggregation job: raw `samples` → `profiles`
- [ ] User category detection: developer (rustc/gcc/node/python/git present)
- [ ] Bootstrap mode: cold profile with "estimated" label
- [ ] Rule engine: loads built-in TOML rules, evaluates conditions, produces `Recommendation` list
- [ ] Implement all 7 built-in v1 rules
- [ ] Recommendation output: action, evidence string, expected impact, risk level
- [ ] Unit tests: each rule fires correctly given mock profile data, does NOT fire when conditions unmet
- [ ] Data pruning: raw samples older than 14 days deleted on weekly job run

---

## Sprint 4 — TUI Foundation

### Study Gate

**Rust topics to cover before starting:**
- [ ] [ratatui book — Getting Started](https://ratatui.rs/tutorials/hello-world/) — event loop, `Terminal`, `Frame`, `Block`, `Paragraph`
- [ ] [ratatui book — Layout](https://ratatui.rs/how-to/layout/split-a-space/) — `Layout::default().split()` for splitting the screen
- [ ] [Tokio tutorial — Channels](https://tokio.rs/tokio/tutorial/channels) — `mpsc` channels for sending data between async tasks (needed for the 2s refresh loop)
- [ ] Understand `crossterm` events: `KeyCode`, `KeyEvent`, `poll`, `read` — ratatui uses this for keyboard input

**Linux topics to cover before starting:**
- [ ] No new Linux topics required

### Warmup Exercise
- [ ] **`tui-hello`**: Build a ratatui app that:
  - Shows a `Block` with title "Wolf" and a `Paragraph` with "Hello, World"
  - Quits on `q` or `Ctrl+C`
  - Has a bottom bar showing `[q] Quit`
  - This is your first ratatui app — get comfortable with the event loop before building the real thing

- [ ] **`tui-counter`**: Extend `tui-hello` to show a number that increments every 2 seconds using `tokio::time::interval` and an `mpsc` channel. This is the exact pattern the dashboard refresh uses.

### Sprint Checklist
- [ ] `wolf-tui` binary with `ratatui` setup, event loop, clean shutdown on `q`/`Ctrl+C`
- [ ] Dashboard screen: live CPU%, RAM usage, disk I/O (2s refresh via `tokio` async)
- [ ] Profile summary widget: detected category, top 5 apps by usage
- [ ] Screen router: tab/arrow key navigation between Dashboard, Optimize, Clean, Settings placeholders
- [ ] Color theme: readable on dark and light backgrounds
- [ ] Help bar at bottom: always shows active keybindings
- [ ] TUI reads profile from SQLite (via `wolf-core`)

---

## Sprint 5 — Optimize Screen & Execution Pipeline

### Study Gate

**Rust topics to cover before starting:**
- [ ] Review `tokio::net::UnixStream` (client side) — you built the server in Sprint 2, now you're building the client
- [ ] [Tokio tutorial — Select](https://tokio.rs/tokio/tutorial/select) — `tokio::select!` for racing futures (useful for timeout handling during execution)
- [ ] Think through state machines: draw on paper the states for the optimize flow (`Idle → Confirming → Executing → Done/Failed/RolledBack`). Rust enums are perfect for this — no library needed.

**Linux topics to cover before starting:**
- [ ] No new Linux topics required

### Warmup Exercise
- [ ] **`tui-confirm`**: Build a ratatui screen with a list of 3 items, a `[Space]` to toggle selection, and a `[Enter]` confirm that shows a second "Are you sure? [y/N]" prompt before printing "Confirmed" or "Cancelled". This is the exact UX pattern for the optimize flow.

### Sprint Checklist
- [ ] Optimize screen: renders recommendation list with evidence and risk level
- [ ] Dry-run mode: shows before/after values, no execution
- [ ] Confirmation flow: select → confirm → second confirmation for kernel params
- [ ] Execution pipeline: TUI sends ops to daemon via Unix socket
- [ ] Live progress view: pending → success/failed per op
- [ ] Rollback button: one keypress reverts last session
- [ ] Failure handling: mid-session failure triggers automatic rollback
- [ ] Emergency restore script written on rollback failure
- [ ] Audit log viewer in Settings screen

---

## Sprint 6 — Clean Screen & Storage Analysis

### Study Gate

**Rust topics to cover before starting:**
- [ ] [`walkdir` crate](https://docs.rs/walkdir/latest/walkdir/) — recursive directory traversal; understand `WalkDir::new`, `DirEntry`, filtering
- [ ] `std::fs::metadata` — understand `len()` (file size), `accessed()` (last access time)
- [ ] Review Rust's `std::path::PathBuf` and `Path` — needed for safe path handling

**Linux topics to cover before starting:**
- [ ] Read: what is `atime` (access time) on Linux files? Search "Linux file atime mtime ctime explained" — one article
- [ ] In your terminal: `ls -lu ~/.cache/` — the `-u` flag shows access times. Understand what you're looking at.
- [ ] Read the whitelisted scan paths in `CLAUDE.md` — memorize what is and isn't allowed to be scanned

### Warmup Exercise
- [ ] **`disk-scout`**: Write a Rust binary that takes a directory path as a CLI arg, recursively walks it with `walkdir`, and prints the top 10 largest files with their sizes and last-accessed date. Add a `--min-age-days N` flag to filter files not accessed in N days. This is the core of the clean screen's scan logic.

### Sprint Checklist
- [ ] Clean screen: triggers scan of whitelisted paths
- [ ] Scan results grouped by category with size and last-accessed date
- [ ] Profile-aware suppression: cargo cache suppressed if user compiles Rust frequently
- [ ] File preview: show exact paths before any deletion
- [ ] Deletion flow: select → preview → explicit confirmation → execute
- [ ] Progress indicator during scan and deletion
- [ ] Nice values: renice feature for background processes
- [ ] Unit tests: path whitelist enforcement (assert no scan outside allowed dirs)

---

## Sprint 7 — Scheduling, Notifications & Polish

### Study Gate

**Rust topics to cover before starting:**
- [ ] No major new Rust topics — this sprint is mostly integration and polish
- [ ] Skim [`clap` derive API](https://docs.rs/clap/latest/clap/_derive/index.html) — for any remaining CLI subcommands

**Linux topics to cover before starting:**
- [ ] Read: "systemd timer units tutorial" — search this phrase. Understand `.timer` files, `OnCalendar=`, and `Persistent=true`
- [ ] Read: `/proc/stat` format — specifically the `cpu` line for idle time calculation. Run `cat /proc/stat` and read what each column means.
- [ ] Read: what is `/etc/motd.d/`? Run `ls /etc/motd.d/` on your system.
- [ ] Read: man page format basics — search "writing a man page with troff" or look at `scdoc` as a simpler alternative

### Warmup Exercise
- [ ] **`idle-check`**: Write a Rust binary that reads `/proc/stat`, computes the CPU idle percentage over a 5-second window (two reads, 5s apart), and prints "System idle: X%" . This is the idle detection logic for the weekly scheduler.
- [ ] **Write a `.timer` unit** (not Rust — just a systemd unit file) that triggers a script every Sunday at midnight. Test it with `systemd-analyze calendar 'Sun *-*-* 00:00:00'`. Understanding systemd timers in practice before wiring them to `wolf-optimizer`.

### Sprint Checklist
- [ ] Weekly analysis cron: systemd timer triggers analysis Sunday midnight if system idle > 10 min
- [ ] Idle detection: check `/proc/stat` and input device activity
- [ ] Optional motd integration: write summary to `/etc/motd.d/wolf` (opt-in)
- [ ] Settings screen: toggle notifications, configure schedule, view/clear audit log
- [ ] Bootstrap mode integrated into first-run TUI flow
- [ ] Error messages: all error states have user-facing explanations (no raw panics shown)
- [ ] Man page: `wolf(1)` covering all CLI subcommands
- [ ] README finalized with screenshots

---

## Sprint 8 — Packaging, Testing & Release

### Study Gate

**Rust topics to cover before starting:**
- [ ] [`cargo-deb` documentation](https://github.com/kornelski/cargo-deb) — understand `[package.metadata.deb]` in `Cargo.toml`, how assets are specified, and how `maintainer-scripts` work
- [ ] Read: Debian package structure basics — search "Debian package preinst postinst postrm scripts" — one article. Understand what `postinst` and `postrm` scripts do and when they run.

**Linux topics to cover before starting:**
- [ ] Read: "dpkg vs apt — what's the difference?" — one short article
- [ ] In a VM: practice `sudo dpkg -i package.deb`, `sudo apt install ./package.deb`, and `sudo dpkg --purge package-name` so you understand the install/remove lifecycle before testing Wolf's own package

### Warmup Exercise
- [ ] **Package `disk-scout`**: Take the `disk-scout` binary from Sprint 6's warmup and package it as a `.deb` using `cargo-deb`. Write a `postinst` script that prints "disk-scout installed". Install it in a VM with `sudo dpkg -i`. This is a low-stakes dry run of the full packaging workflow.

### Sprint Checklist
- [ ] `cargo-deb` configuration: both binaries, systemd units, man pages, post-install scripts
- [ ] Post-install: enable `wolf-collector` user service, create `wolf-optimizer` group
- [ ] Post-remove / purge scripts
- [ ] Full QEMU integration test suite green
- [ ] Manual test checklist completed on fresh Ubuntu 22.04 and 24.04 VMs
- [ ] `apt install wolf-optimizer` end-to-end tested
- [ ] Security review: audit daemon whitelist, socket auth, file permissions
- [ ] v1.0.0 tag, GitHub release with `.deb` artifact attached
- [ ] AUR PKGBUILD (stretch goal)

---

## Reference: Skills by Sprint

| Skill | First needed | Sprints that use it |
|-------|-------------|---------------------|
| Rust ownership & borrowing | Phase 0 | All |
| Cargo workspaces | Sprint 1 | All |
| `rusqlite` | Sprint 1 | 1, 3, 4 |
| `/proc` filesystem parsing | Sprint 1 | 1, 7 |
| `tokio` async/await | Sprint 2 | 2, 4, 5 |
| Unix domain sockets | Sprint 2 | 2, 5 |
| `serde_json` | Sprint 2 | 2, 5 |
| Linux groups & permissions | Sprint 2 | 2, 8 |
| `systemd` units | Sprint 2 | 2, 7, 8 |
| Iterators & closures | Sprint 3 | 3, 6 |
| TOML deserialization | Sprint 3 | 3 |
| `ratatui` | Sprint 4 | 4, 5, 6, 7 |
| `tokio` channels | Sprint 4 | 4, 5 |
| `walkdir` + `std::fs` | Sprint 6 | 6 |
| systemd timers | Sprint 7 | 7 |
| `cargo-deb` | Sprint 8 | 8 |

---

## Pacing Reality Check

At 10–20 hrs/week with a beginner starting point:

| Phase | Recommended Duration |
|-------|---------------------|
| Phase 0 (Rust Foundations) | 3 weeks |
| Sprint 1 | 3 weeks |
| Sprint 2 | 3–4 weeks (safety-critical, don't rush) |
| Sprints 3–6 | 3 weeks each |
| Sprints 7–8 | 2–3 weeks each |
| **Total** | **~28–32 weeks** |

The original 16-week timeline assumes a senior Rust developer. At your current level with the hybrid study approach, plan for roughly double. That's not a failure — it's an accurate map.

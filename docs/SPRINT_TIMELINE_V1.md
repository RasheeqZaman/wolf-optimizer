# Wolf Optimizer v1 — Sprint Timeline

2-week sprints. Each sprint has a clear deliverable that can be demoed or tested independently.

---

## Sprint 1 — Project Scaffold & Core Data Layer
**Goal:** Cargo workspace compiles, SQLite schema exists, collector writes data.

- [ ] Initialize Cargo workspace with 4 crates: `wolf-core`, `wolf-collector`, `wolf-daemon`, `wolf-tui`
- [ ] Define shared types in `wolf-core`: `ProcessSample`, `AppProfile`, `Recommendation`, `RiskLevel`
- [ ] SQLite schema: `samples` table, `profiles` table, `audit_log` table
- [ ] Database initialization and migration logic in `wolf-core`
- [ ] `wolf-collector` binary: reads `/proc` every 30s, writes `ProcessSample` to DB
- [ ] WAL mode enabled on SQLite for crash safety
- [ ] Unit tests: schema creation, sample insert/read, duplicate handling
- [ ] CI: GitHub Actions pipeline compiles all crates on push

**Exit criteria:** Running `wolf-collector` for 5 minutes produces valid rows in `activity.db`.

---

## Sprint 2 — Privileged Daemon (Safety-Critical)
**Goal:** Daemon runs, enforces whitelist, communicates over Unix socket.

- [ ] `wolf-daemon` binary skeleton with socket-activated systemd unit
- [ ] Unix domain socket setup at `/run/wolf-optimizer/daemon.sock`
- [ ] `wolf-optimizer` group enforcement: reject callers not in group
- [ ] Session token generation and validation
- [ ] Hardcoded kernel param whitelist (v1 list: `vm.swappiness`, `vm.dirty_ratio`, `vm.dirty_background_ratio`, `kernel.sched_migration_cost_ns`)
- [ ] Request handler: parse JSON op, validate against whitelist, reject unknown ops
- [ ] Snapshot mechanism: read current sysctl value before any write
- [ ] Rollback mechanism: restore all snapshotted values on command or error
- [ ] Audit log writer: every op logged with timestamp, caller, before/after values
- [ ] Unit tests: whitelist enforcement (assert rejection of non-whitelisted params), rollback logic
- [ ] Integration test (QEMU): daemon starts, accepts valid request, rejects invalid request, rollback restores values

**Exit criteria:** Daemon correctly rejects 100% of non-whitelisted params in integration tests. Rollback test passes.

---

## Sprint 3 — Profile Engine & Rule System
**Goal:** Weekly profile is generated, rules produce recommendations.

- [ ] Profile aggregation job: raw `samples` → `profiles` (avg CPU, peak RAM, run frequency, last seen)
- [ ] User category detection: developer (rustc/gcc/node/python/git present in profile)
- [ ] Bootstrap mode: one-time scan of whitelisted paths, generates cold profile with "estimated" label
- [ ] Rule engine: loads built-in TOML rules, evaluates conditions against profile, produces `Recommendation` list
- [ ] Implement all 7 built-in v1 rules (low-swappiness, keep-cargo-cache, clean-thumbnails, clean-npm-cache, clean-old-logs, nice-background-apps, clean-trash)
- [ ] Recommendation output: action, evidence string, expected impact, risk level
- [ ] Unit tests: each rule fires correctly given mock profile data, does NOT fire when conditions unmet
- [ ] Data pruning: raw samples older than 14 days deleted on weekly job run

**Exit criteria:** Given a synthetic 7-day sample dataset, rule engine produces correct recommendations with accurate evidence strings.

---

## Sprint 4 — TUI Foundation
**Goal:** TUI launches, dashboard shows live data, navigation works.

- [ ] `wolf-tui` binary with `ratatui` setup, event loop, clean shutdown on `q`/`Ctrl+C`
- [ ] Dashboard screen: live CPU%, RAM usage, disk I/O (2s refresh via `tokio` async)
- [ ] Profile summary widget: detected category, top 5 apps by usage
- [ ] Screen router: tab/arrow key navigation between Dashboard, Optimize, Clean, Settings placeholders
- [ ] Color theme: readable on dark and light backgrounds
- [ ] Help bar at bottom: always shows active keybindings
- [ ] TUI reads profile from SQLite (via `wolf-core`)

**Exit criteria:** TUI launches in < 1 second, dashboard updates live, keyboard navigation works across all screens.

---

## Sprint 5 — Optimize Screen & Execution Pipeline
**Goal:** User can view recommendations, dry-run, confirm, and execute kernel optimizations.

- [ ] Optimize screen: renders recommendation list with evidence and risk level per item
- [ ] Dry-run mode: shows before/after values for each planned change, no execution
- [ ] Confirmation flow: user selects recommendations, presses confirm, second confirmation for kernel params
- [ ] Execution pipeline: TUI sends ops to daemon via Unix socket, daemon executes with snapshot+rollback
- [ ] Live progress view: shows each op as it executes (pending → success/failed)
- [ ] Rollback button: one keypress reverts last session
- [ ] Failure handling: mid-session failure triggers automatic rollback, shows user what was reverted
- [ ] Emergency restore script written on rollback failure
- [ ] Audit log viewer in Settings screen

**Exit criteria:** Full optimize flow works end-to-end in QEMU VM. Intentional failure mid-session triggers clean rollback.

---

## Sprint 6 — Clean Screen & Storage Analysis
**Goal:** User can scan, preview, and safely delete storage waste.

- [ ] Clean screen: triggers scan of whitelisted paths
- [ ] Scan results grouped by category with size and last-accessed date
- [ ] Profile-aware suppression: cargo cache suppressed if user compiles Rust frequently
- [ ] File preview: show exact paths before any deletion
- [ ] Deletion flow: select categories → preview → explicit confirmation → execute
- [ ] Progress indicator during scan and deletion
- [ ] Nice values: `wolf-collector` renice feature for rarely-used background processes (user-space, no daemon needed)
- [ ] Unit tests: path whitelist enforcement (assert no scan outside allowed dirs)

**Exit criteria:** Clean screen correctly identifies cache bloat on a test Ubuntu VM and deletes only confirmed items.

---

## Sprint 7 — Scheduling, Notifications & Polish
**Goal:** Automated weekly analysis works, UX is polished, ready for packaging.

- [ ] Weekly analysis cron: systemd timer triggers analysis Sunday midnight if system idle > 10 min
- [ ] Idle detection: check `/proc/stat` and input device activity
- [ ] Optional motd integration: write summary to `/etc/motd.d/wolf` (opt-in in Settings)
- [ ] Settings screen: toggle notifications, configure schedule, view/clear audit log
- [ ] Bootstrap mode integrated into first-run TUI flow with clear "estimated" labeling
- [ ] Error messages: all error states have user-facing explanations (no raw Rust panics shown)
- [ ] Man page: `wolf(1)` covering all CLI subcommands
- [ ] README finalized with screenshots

**Exit criteria:** Fresh Ubuntu VM install → `wolf` first run → bootstrap profile → weekly timer fires and prepares recommendations.

---

## Sprint 8 — Packaging, Testing & Release
**Goal:** Installable `.deb`, CI green, v1.0.0 tagged.

- [ ] `cargo-deb` configuration: both binaries, systemd units, man pages, post-install scripts
- [ ] Post-install: enable `wolf-collector` user service, create `wolf-optimizer` group
- [ ] Post-remove / purge scripts
- [ ] Full QEMU integration test suite green: install from .deb, full optimize flow, rollback, clean flow, uninstall
- [ ] Manual test checklist completed on fresh Ubuntu 22.04 and 24.04 VMs
- [ ] `apt install wolf-optimizer` end-to-end tested
- [ ] Security review: audit daemon whitelist, socket auth, file permissions
- [ ] v1.0.0 tag, GitHub release with `.deb` artifact attached
- [ ] AUR PKGBUILD (stretch goal)

**Exit criteria:** `sudo apt install ./wolf-optimizer_1.0.0_amd64.deb && wolf` works on a fresh Ubuntu 24.04 VM with zero manual steps beyond the group membership command.

---

## Timeline Summary

| Sprint | Focus | Duration |
|--------|-------|----------|
| 1 | Scaffold + Data Layer | 2 weeks |
| 2 | Privileged Daemon | 2 weeks |
| 3 | Profile + Rule Engine | 2 weeks |
| 4 | TUI Foundation | 2 weeks |
| 5 | Optimize Screen + Execution | 2 weeks |
| 6 | Clean Screen | 2 weeks |
| 7 | Scheduling + Polish | 2 weeks |
| 8 | Packaging + Release | 2 weeks |
| **Total** | | **16 weeks** |

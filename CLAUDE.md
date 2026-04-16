# Wolf Optimizer — Claude Code Instructions

## Project Overview
Wolf Optimizer is a Rust-based terminal UI PC performance optimizer and storage cleaner for Linux. It analyzes weekly user activity, builds a behavioral profile, and recommends/applies optimizations. **OS safety is the #1 priority.**

## Architecture
Three separate binaries:
- `wolf-collector` — user-space systemd service, samples system activity every 30s, writes to SQLite
- `wolf-optimizer` — ratatui TUI, runs as regular user, reads profile, communicates with daemon via Unix socket
- `wolf-daemon` — privileged binary (socket-activated), executes whitelisted kernel-level operations only

## Safety Rules — Never Violate
- The daemon must ONLY execute operations on the hardcoded whitelist. No arbitrary sysctl writes, ever.
- Every optimization session must snapshot current values before making any changes.
- On any failure mid-session, rollback ALL changes made in that session before exiting.
- Kernel parameter changes ALWAYS require explicit user confirmation. Never auto-apply.
- If rollback fails, write an emergency restore script to `~/.local/share/wolf-optimizer/emergency-restore.sh` and show the user manual restore commands.
- The TUI process must never directly hold root privileges. All privileged ops go through the daemon.

## IPC Protocol
- Unix domain socket at `/run/wolf-optimizer/daemon.sock`
- Only users in the `wolf-optimizer` group may connect
- Session token auth issued at daemon start
- JSON messages: `{"op": "set_swappiness", "value": 60}` → `{"ok": true, "prev_value": 80}`

## File Layout
```
wolf-optimizer/
├── crates/
│   ├── wolf-collector/     # systemd user service binary
│   ├── wolf-daemon/        # privileged daemon binary
│   ├── wolf-tui/           # ratatui TUI binary
│   └── wolf-core/          # shared types, profile, rule engine, SQLite
├── rules/                  # built-in TOML rule files
├── packaging/              # .deb packaging config (cargo-deb)
├── tests/                  # integration tests (VM-based)
└── CLAUDE.md
```

## Key Dependencies
- `ratatui` — TUI framework
- `rusqlite` — SQLite profile storage
- `tokio` — async runtime
- `serde` / `serde_json` — serialization
- `clap` — CLI argument parsing
- `cargo-deb` — .deb packaging

## Data Locations
- Profile DB: `~/.local/share/wolf-optimizer/activity.db`
- Audit log: `~/.local/share/wolf-optimizer/audit.log`
- Daemon socket: `/run/wolf-optimizer/daemon.sock`
- Collector service: `~/.config/systemd/user/wolf-collector.service`

## Whitelisted Scan Paths
Only these paths may be scanned for cleaning recommendations:
`~/.cache/`, `~/.local/share/Trash/`, `~/.npm/`, `~/.cargo/registry/`, `~/.gradle/`, `~/.m2/`, `~/.pip/`, `~/.thumbnails/`, `~/Downloads/`, `/tmp/`, `/var/tmp/`, `/var/log/`, `/var/cache/`

Never touch: `~/Documents/`, `~/Pictures/`, `~/Videos/`, `~/.ssh/`, `~/.gnupg/`, `~/.config/` contents.

## Rule Format
Rules are TOML files in `rules/`. Each rule has: `id`, `name`, `author`, `version`, `risk` (safe/moderate/requires_confirmation), `[condition]`, `[action]`, `[reason]`. The daemon validates every action against the whitelist regardless of what the rule requests.

## Testing
- Unit tests: all rule logic, whitelist enforcement, rollback logic — 100% coverage on safety-critical paths
- Integration tests: QEMU Ubuntu VM via GitHub Actions
- Never skip integration tests for daemon changes

## Code Style
- No `unwrap()` in production paths — use `?` and proper error types
- Every public function in `wolf-daemon` must have a safety comment explaining why it's safe
- Prefer explicit error types over `anyhow` in the daemon crate

<!-- code-review-graph MCP tools -->
## MCP Tools: code-review-graph

**IMPORTANT: This project has a knowledge graph. ALWAYS use the
code-review-graph MCP tools BEFORE using Grep/Glob/Read to explore
the codebase.** The graph is faster, cheaper (fewer tokens), and gives
you structural context (callers, dependents, test coverage) that file
scanning cannot.

### When to use graph tools FIRST

- **Exploring code**: `semantic_search_nodes` or `query_graph` instead of Grep
- **Understanding impact**: `get_impact_radius` instead of manually tracing imports
- **Code review**: `detect_changes` + `get_review_context` instead of reading entire files
- **Finding relationships**: `query_graph` with callers_of/callees_of/imports_of/tests_for
- **Architecture questions**: `get_architecture_overview` + `list_communities`

Fall back to Grep/Glob/Read **only** when the graph doesn't cover what you need.

### Key Tools

| Tool | Use when |
|------|----------|
| `detect_changes` | Reviewing code changes — gives risk-scored analysis |
| `get_review_context` | Need source snippets for review — token-efficient |
| `get_impact_radius` | Understanding blast radius of a change |
| `get_affected_flows` | Finding which execution paths are impacted |
| `query_graph` | Tracing callers, callees, imports, tests, dependencies |
| `semantic_search_nodes` | Finding functions/classes by name or keyword |
| `get_architecture_overview` | Understanding high-level codebase structure |
| `refactor_tool` | Planning renames, finding dead code |

### Workflow

1. The graph auto-updates on file changes (via hooks).
2. Use `detect_changes` for code review.
3. Use `get_affected_flows` to understand impact.
4. Use `query_graph` pattern="tests_for" to check coverage.

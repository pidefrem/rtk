# Search & Navigation — RTK Codebase

Reference for navigating and searching RTK's Rust codebase. Read this when you need the module
map or a search recipe; it is intentionally *not* an always-loaded rule.

## Which search mechanism do you have?

Claude Code changes which tools are registered depending on model, build type, and server-side
feature flags. Two states seen in the wild:

- **Structured tools present** — a `Grep` and/or `Glob` tool appear in your tool list (typical of
  npm-installed builds, and older/other models). Prefer them: they're faster and cleaner than
  shelling out.
- **Bash-search build** — no `Grep`/`Glob` in the main loop; Claude Code replaced them with
  embedded `bfs`/`ugrep` reached **through Bash** (e.g. Opus 4.8 on native macOS/Linux, the
  `BashSearchTool` GrowthBook rollout). Here, Bash `grep`/`rg`/`ugrep` **is** the sanctioned
  primary, not a fallback, and the harness will tell you so ("search file contents with grep via
  the Bash tool").

**Decide by looking at your available tools, not from memory.** If a `Grep`/`Glob` tool is
registered, use it; otherwise search via Bash. The navigation *logic* below (what to look for, in
what order) is identical — only the mechanism differs. The dedicated `Grep`/`Glob` tools also
survive inside **Explore subagents** regardless of the main-loop state, so delegating a search
there is a reliable way to get structured search.

> **rtk hook caveat:** the rtk PreToolUse hook reshapes wrapped `grep` output. For exact-line
> fidelity (or to diff against the raw tool), bypass via an absolute path like `/usr/bin/grep`.

## Priority Order

Applies whichever mechanism is active — substitute "Grep tool" ↔ "Bash `grep`/`rg`" per above:

1. **Exact-pattern search** (fast) → for known symbols/strings. `Grep` tool if present, else Bash
   `rg` for recursive/`.gitignore`-aware sweeps or `grep -n` for a single known file.
2. **File discovery** → for finding modules by name. `Glob` tool if present, else Bash
   `rg --files -g '<glob>'` / `ls`.
3. **Read** (full file) → only after locating the right file.
4. **Explore subagent** (broad research, >3 queries, multi-location fan-out, or when you
   specifically want structured `Grep`/`Glob`) → subagents carry the dedicated tools regardless of
   the main-loop state.

## RTK Module Map

```
src/
├── main.rs                    ← Commands enum + routing (start here for any command)
├── core/                      ← Shared infrastructure
│   ├── config.rs              ← ~/.config/rtk/config.toml
│   ├── tracking.rs            ← SQLite token metrics
│   ├── tee.rs                 ← Raw output recovery on failure
│   ├── utils.rs               ← strip_ansi, truncate, execute_command
│   ├── filter.rs              ← Language-aware code filtering engine
│   ├── toml_filter.rs         ← TOML DSL filter engine
│   ├── display_helpers.rs     ← Terminal formatting helpers
│   └── telemetry.rs           ← Analytics ping
├── hooks/                     ← Hook system
│   ├── init.rs                ← rtk init command
│   ├── rewrite_cmd.rs         ← rtk rewrite command
│   ├── hook_cmd.rs            ← Gemini/Copilot hook processors
│   ├── hook_check.rs          ← Hook status detection
│   ├── verify_cmd.rs          ← rtk verify command
│   ├── trust.rs               ← Project trust/untrust
│   └── integrity.rs           ← SHA-256 hook verification
├── analytics/                 ← Token savings analytics
│   ├── gain.rs                ← rtk gain command
│   ├── cc_economics.rs        ← Claude Code economics
│   ├── ccusage.rs             ← ccusage data parsing
│   └── session_cmd.rs         ← Session adoption reporting
├── cmds/                      ← Command filter modules
│   ├── git/                   ← git, gh, gt, diff
│   ├── rust/                  ← cargo, runner (err/test)
│   ├── js/                    ← npm, pnpm, vitest, lint, tsc, next, prettier, playwright, prisma
│   ├── python/                ← ruff, pytest, mypy, pip
│   ├── go/                    ← go, golangci-lint
│   ├── dotnet/                ← dotnet, binlog, trx, format_report
│   ├── cloud/                 ← aws, container (docker/kubectl), curl, wget, psql
│   ├── system/                ← ls, tree, read, grep, find, wc, env, json, log, deps, summary, format, local_llm
│   └── ruby/                  ← rake, rspec, rubocop
├── discover/                  ← Claude Code history analysis
├── learn/                     ← CLI correction detection
├── parser/                    ← Parser infrastructure
└── filters/                   ← 60 TOML filter configs
```

## Common Search Patterns

Each pattern shows the **structured-tool** form and the **Bash** form. Use whichever matches your
session (see above); the intent is identical.

### "Where is command X handled?"

```
# Tool:  Grep pattern="Gh|Cargo|Git|Grep" path="src/main.rs" output_mode="content"
# Bash:  rg -n 'Gh|Cargo|Git|Grep' src/main.rs
# Then follow to module:  Read file_path="src/cmds/git/gh_cmd.rs"
```

### "Where is function X defined?"

```
# Tool:  Grep pattern="fn filter_git_log|fn run\b" type="rust"
# Bash:  rg -n 'fn filter_git_log|fn run\b' -g '*.rs'
```

### "All command modules"

```
# Tool:  Glob pattern="src/cmds/**/*_cmd.rs"
# Bash:  rg --files -g 'src/cmds/**/*_cmd.rs'
# Also: src/cmds/git/git.rs, src/cmds/rust/runner.rs, src/cmds/cloud/container.rs
```

### "Find all lazy_static regex definitions"

```
# Tool:  Grep pattern="lazy_static!" type="rust" output_mode="content"
# Bash:  rg -n 'lazy_static!' -g '*.rs'
```

### "Find unwrap() outside tests"

```
# Tool:  Grep pattern="\.unwrap()" type="rust" output_mode="content"
# Bash:  rg -n '\.unwrap\(\)' -g '*.rs'
# Then manually filter out #[cfg(test)] blocks
```

### "Which modules have tests?"

```
# Tool:  Grep pattern="#\[cfg\(test\)\]" type="rust" output_mode="files_with_matches"
# Bash:  rg -l '#\[cfg\(test\)\]' -g '*.rs'
```

### "Find token savings assertions"

```
# Tool:  Grep pattern="count_tokens|savings" type="rust" output_mode="content"
# Bash:  rg -n 'count_tokens|savings' -g '*.rs'
```

### "Find test fixtures"

```
# Tool:  Glob pattern="tests/fixtures/*.txt"
# Bash:  rg --files -g 'tests/fixtures/*.txt'
```

## RTK-Specific Navigation

### Adding a new filter

1. Check `src/main.rs` for Commands enum structure
2. Check existing modules in `src/cmds/<ecosystem>/` for patterns to follow (e.g., `src/cmds/git/gh_cmd.rs`)
3. Check `src/core/utils.rs` for shared helpers before reimplementing
4. Check `tests/fixtures/` for existing fixture patterns

### Debugging filter output

1. Start with `src/cmds/<ecosystem>/<cmd>_cmd.rs` → find `run()` function
2. Trace filter function (usually `filter_<cmd>()`)
3. Check `lazy_static!` regex patterns in same file
4. Check `src/core/utils.rs::strip_ansi()` if ANSI codes involved

### Tracking/metrics issues

1. `src/core/tracking.rs` → `track_command()` function
2. `src/core/config.rs` → `tracking.database_path` field
3. `RTK_DB_PATH` env var overrides config

### Configuration issues

1. `src/core/config.rs` → `RtkConfig` struct
2. `src/hooks/init.rs` → `rtk init` command
3. Config file: `~/.config/rtk/config.toml`
4. Filter files: `~/.config/rtk/filters/` (global) or `.rtk/filters/` (project)

## TOML Filter DSL Navigation

```
# Tool:  Glob pattern=".rtk/filters/*.toml"        (project-local filters)
# Bash:  rg --files -g '.rtk/filters/*.toml'
# Tool:  Glob pattern="src/core/toml_filter.rs"    (TOML filter engine)
# Tool:  Grep pattern="FilterRule|FilterConfig" type="rust"
# Bash:  rg -n 'FilterRule|FilterConfig' -g '*.rs'
```

## Dependency Check

```
# Check if a crate is already used (before adding)
# Tool:  Grep pattern="^regex|^anyhow|^rusqlite" glob="Cargo.toml" output_mode="content"
# Bash:  rg -n '^regex|^anyhow|^rusqlite' Cargo.toml

# Check if async is creeping in (forbidden)
# Tool:  Grep pattern="tokio|async-std|futures|async fn" type="rust"
# Bash:  rg -n 'tokio|async-std|futures|async fn' -g '*.rs'
```

## Anti-Patterns

- **Don't** read all `*_cmd.rs` files to find one function — search for it first
- **Don't** read `main.rs` entirely to find a module — search for the command name
- **Don't** ignore the tools you actually have — if a `Grep`/`Glob` tool is registered, prefer it
  over shelling out; if it isn't, don't refuse to search — use Bash `rg`/`grep`
- **Don't** hardcode one mechanism — the choice between structured tools and Bash search depends on
  the session (see the top), so branch on what's available

# 🏛️ ARCHITECTURE

> System structure, component relationships, and module boundaries for both the Python mirror and Rust operational surfaces.

---

## 1 · What This Repository Actually Is

This repository currently has **two implementation surfaces**:

| Surface | Location | Role |
|---------|----------|------|
| 🐍 Python mirror | `src/` | Snapshot-backed metadata model of command/tool/runtime concepts |
| 🦀 Rust runtime | `rust/` | Executable runtime logic with real CLI loop and policy enforcement |

This split is visible in:

- **Python entrypoint:** `src/main.py` → `main`, `build_parser`
- **Rust entrypoint:** `rust/crates/claw-cli/src/main.rs` → `main`, `run`

---

## 2 · Top-Level Folders And Purpose

| Folder | Purpose |
|--------|---------|
| `assets/` | Documentation/community images — no runtime logic |
| `.github/` | Repository funding metadata |
| `src/` | Python porting workspace |
| `tests/` | Python verification suite |
| `rust/` | Rust workspace with crates for runtime/CLI/tools/api/plugins |

---

## 3 · Python Workspace (`src/`) Architecture

### 3.1 · High-Value Python Modules (Real Logic)

#### `src/main.py` — CLI Dispatch and Initialization

`src/main.py` orchestrates the Python CLI surface. Every subcommand flows through a single entry point that parses arguments, builds the port manifest once, and dispatches to the appropriate runtime function. This design centralizes CLI routing and ensures consistent setup.

**CLI Dispatch Pattern:**
```python
def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)
    manifest = build_port_manifest()
    
    if args.command == 'summary':
        print(QueryEnginePort(manifest).render_summary())
        return 0
    if args.command == 'route':
        matches = PortRuntime().route_prompt(args.prompt, limit=args.limit)
        for match in matches:
            print(f'{match.kind}\t{match.name}\t{match.score}\t{match.source_hint}')
        return 0
```

> **What it does:** Centralizes argument parsing and manifest construction, then branches to subcommand handlers (route, bootstrap, turn-loop, etc.).
>
> **Why it matters:** Single entry point ensures all subcommands inherit the same initialization logic and avoid duplicating manifest building.
>
> **Key insight:** The manifest is built once and passed to all downstream operations. This is cheap because it only counts files, not parsing code.

---

#### `src/runtime.py`

- `PortRuntime.route_prompt` — tokenize + score + select matches
- `PortRuntime.bootstrap_session` — compose session from matched commands/tools
- `PortRuntime.run_turn_loop` — multi-turn simulation
- Core lightweight matching + session assembly for the Python mirror

---

#### `src/query_engine.py` — Turn Simulation and Session Persistence

`QueryEnginePort` models bounded turn execution with usage tracking and durable session state. It enforces hard stops (`max_turns`, `max_budget`) synchronously during turn submission, ensuring the caller always knows why a turn was rejected.

**Session Durability Pattern:**
```python
def persist_session(self) -> str:
    self.flush_transcript()
    path = save_session(
        StoredSession(
            session_id=self.session_id,
            messages=tuple(self.mutable_messages),
            input_tokens=self.total_usage.input_tokens,
            output_tokens=self.total_usage.output_tokens,
        )
    )
    return str(path)
```

> **What it does:** Flushes in-memory transcript, wraps session state into a `StoredSession` dataclass, serializes to JSON in `.port_sessions/`, and returns the path.
>
> **Why it matters:** Sessions survive process restarts. The session ID acts as a recovery key, allowing clients to resume from the last checkpoint.
>
> **Key insight:** Session state is immutable once persisted (new session ID = new file), preventing accidental overwrites of partial sessions.

---

#### `src/commands.py` and `src/tools.py` — Snapshot-Backed Registry

Both modules load command and tool metadata from static JSON snapshots in `src/reference_data/`. The `@lru_cache` decorator ensures a single parse per process, making repeated registry lookups (`get_commands`/`get_tools`) O(1).

**Snapshot Loading Pattern:**
```python
@lru_cache(maxsize=1)
def load_command_snapshot() -> tuple[PortingModule, ...]:
    raw_entries = json.loads(SNAPSHOT_PATH.read_text())
    return tuple(
        PortingModule(
            name=entry['name'],
            responsibility=entry['responsibility'],
            source_hint=entry['source_hint'],
            status='mirrored',
        )
        for entry in raw_entries
    )
```

> **What it does:** Reads JSON snapshot only once (via `@lru_cache`), constructs `PortingModule` tuples, and caches the result for permanent process lifetime.
>
> **Why it matters:** Trades runtime flexibility for predictability — snapshot changes require a process restart. Acceptable for a mirroring/testing surface where registries rarely change.
>
> **Key insight:** Snapshots are the source-of-truth for metadata. Execution Registry wraps them into executable handlers, but never contradicts snapshot inventory.

---

#### `src/execution_registry.py`

- `build_execution_registry` — wraps mirrored modules into executable handlers
- Provides `command(name)` and `tool(name)` lookups with `execute()` methods

---

### 3.2 · Supporting Python Modules

| Module | Function |
|--------|----------|
| `src/port_manifest.py` | `build_port_manifest` scans Python files and produces subsystem manifest |
| `src/context.py` | `build_port_context` computes source/test/assets/archive counts and paths |
| `src/setup.py` | `run_setup` composes prefetch + deferred init status |
| `src/system_init.py` | `build_system_init_message` summarizes startup inventory |
| `src/session_store.py` | `save_session`/`load_session` to `.port_sessions/` |
| `src/transcript.py` | `TranscriptStore` append/compact/replay/flush |
| `src/permissions.py` | `ToolPermissionContext` deny-list + deny-prefix filtering |
| `src/command_graph.py` | Command segmentation by `source_hint` (builtins/plugin-like/skill-like) |
| `src/tool_pool.py` | `assemble_tool_pool` applies simple/MCP/permission filters |
| `src/parity_audit.py` | Archive-vs-port surface comparison and ratios |

---

### 3.3 · Placeholder Subsystem Packages

Many folders under `src/` (`assistant`, `bridge`, `utils`, `services`, etc.) are metadata placeholders.

**Pattern:**
- `src/<subsystem>/__init__.py` loads `src/reference_data/subsystems/<subsystem>.json`
- Exposes `ARCHIVE_NAME`, `MODULE_COUNT`, `SAMPLE_FILES`, `PORTING_NOTE`

**Example:** `src/assistant/__init__.py` + `src/reference_data/subsystems/assistant.json`

> These packages mostly describe archived structure, not full Python runtime behavior.

---

## 4 · Rust Workspace (`rust/`) Architecture

### 4.1 · Crates

Defined via `rust/Cargo.toml` workspace `members = crates/*`:

| Crate | Role |
|-------|------|
| `rust/crates/claw-cli` | Executable CLI/repl (`main.rs`) |
| `rust/crates/runtime` | Conversation loop, config, permissions, hooks, session, MCP |
| `rust/crates/tools` | Tool registry, schemas, execution dispatch |
| `rust/crates/commands` | Slash command registry/parsing/help |
| `rust/crates/api` | Model/provider client and streaming types |
| `rust/crates/plugins` | Plugin manager/hook integration |
| `rust/crates/compat-harness` | Parity extraction utilities |

---

### 4.2 · Rust Runtime Layering

```
CLI shell and modes        rust/crates/claw-cli/src/main.rs
        │
        ▼
Core loop                  rust/crates/runtime/src/conversation.rs
        │
        ├─► Policy checks  rust/crates/runtime/src/permissions.rs
        ├─► Hook execution  rust/crates/runtime/src/hooks.rs
        └─► Tool catalog    rust/crates/tools/src/lib.rs
                            rust/crates/commands/src/lib.rs
```

---

## 5 · Entry Points

### 🐍 Python

- **Module CLI entrypoint:** `src/main.py`
- **Guard:** `if __name__ == '__main__': raise SystemExit(main())`

### 🦀 Rust

- **Binary entrypoint:** `rust/crates/claw-cli/src/main.rs` → `fn main -> run`

---

## 6 · Architecture Relationships (Python Side)

Main dependency chain for a typical mirrored run:

```
1. src/main.py           dispatches CLI command
        │
        ▼
2. src/runtime.py        PortRuntime routes prompt
        │                using command/tool metadata from:
        │                src/commands.py + src/tools.py
        ▼
3. src/execution_registry.py  converts matches → executable wrappers
        │
        ▼
4. src/query_engine.py   simulates turn output/usage/stream events
        │
        ▼
5. src/session_store.py  persists session state
   src/transcript.py     compacts transcript
```

---

## 7 · What Is Clearly Simulated vs Fully Implemented

**⚠️ Clearly simulated in Python mirror:**

- Prefetch side effects in `src/prefetch.py` (returns descriptive placeholders)
- Deferred init in `src/deferred_init.py` (trust-gated booleans)
- Remote/SSH/teleport/direct/deep-link mode handlers in `src/remote_runtime.py` and `src/direct_modes.py` (report structs)
- Command/tool execution in `src/commands.py` and `src/tools.py` (message stubs)

**✅ More operational in Rust workspace:**

- Multi-iteration tool loop, permission prompting, hook runner, session state, tool execution plumbing

---

## 8 · Unclear Areas (Explicit)

- Exact feature parity target between the Python mirror and Rust runtime is not fully codified in one machine-readable contract file.
- Several Python files (`dialogLaunchers.py`, `ink.py`, `interactiveHelpers.py`, `replLauncher.py`, `projectOnboardingState.py`) exist for surface parity but contain little/no active logic in this workspace context.
- Some Rust behavior breadth versus original TypeScript system is described in [analysis/PARITY_ANALYSIS.md](../analysis/PARITY_ANALYSIS.md), but that file is a narrative audit and not an enforceable compatibility test spec.

---

## 📌 Key Takeaways

1. **Two-surface design** — Python is a testable mirror with metadata-driven logic; Rust is the operational engine with real policy enforcement and tool execution.

2. **Snapshot-backed registries** — Commands and tools are loaded from static JSON once and cached. Changes require process restart — a reasonable tradeoff for a stable testing surface.

3. **Manifest as shape metric** — The port manifest provides a measurable snapshot of Python module inventory, used for tracking and validating porting progress.

4. **Layered dispatch** — Every CLI command flows through `main()` dispatch, ensuring consistent initialization and avoiding scattered routing logic.

5. **Session durability as first-class** — Session persistence is not an afterthought; `QueryEnginePort` enforces it automatically, making recovery predictable and transparent.

---

**Next →** [EXECUTION_FLOW.md](EXECUTION_FLOW.md) — Trace the step-by-step runtime path from CLI prompt to session persisted state.

**↑ Up:** [Index](../README.md)

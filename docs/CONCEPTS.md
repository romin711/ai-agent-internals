# 💡 CONCEPTS

> Core mental models and design theory for the two-surface harness architecture.

---

## 1 · Core Idea In Plain Language

This codebase is a **two-surface harness project**:

| Surface | Location | Role |
|---------|----------|------|
| 🐍 Python | `src/` | Mirror and learning/testable abstraction of command/tool/runtime surfaces |
| 🦀 Rust | `rust/crates/` | Executable runtime path with real CLI loop, permission policy, hooks, and tool execution |

> Think of Python as a parity-oriented model of system structure, and Rust as the operational engine.

---

## 2 · Concept: Surface Parity vs Runtime Parity

### 🗂️ Surface Parity

Implemented by metadata snapshots and placeholder packages:

- `src/reference_data/commands_snapshot.json`
- `src/reference_data/tools_snapshot.json`
- `src/reference_data/subsystems/*.json`
- `src/commands.py` and `src/tools.py` loaders
- `src/parity_audit.py` ratios

**Meaning:** Names, inventory size, and subsystem mapping can be mirrored without implementing all runtime effects.

---

### ⚙️ Runtime Parity

Implemented where behavior actually executes:

- Rust conversation loop: `rust/crates/runtime/src/conversation.rs`
- Rust permission policy: `rust/crates/runtime/src/permissions.rs`
- Rust tool registry/execution: `rust/crates/tools/src/lib.rs`
- Rust CLI command handling: `rust/crates/claw-cli/src/main.rs`

**Meaning:** A turn can call tools, enforce policy, run hooks, and produce iterative outcomes.

---

## 3 · Concept: The Port Manifest As Ground Truth Snapshot

| | |
|---|---|
| **File** | `src/port_manifest.py` |
| **Function** | `build_port_manifest` |

This function scans `src/` recursively for `.py` files and computes top-level module counts.

**Recursive Module Enumeration:**
```python
def build_port_manifest() -> dict[str, int]:
    """Recursively scan src/ for Python files, count top-level modules."""
    from pathlib import Path
    from collections import Counter
    
    src_dir = Path('src')
    module_counter = Counter()
    
    for py_file in src_dir.rglob('*.py'):
        # Extract top-level module name from path
        # e.g., src/assistant/__init__.py -> "assistant"
        parts = py_file.relative_to(src_dir).parts
        if parts[0] != '__init__.py':
            module_counter[parts[0]] += 1
    
    return dict(module_counter)
```

> **What it does:** Walks `src/` recursively finding all `.py` files, extracts the top-level folder name from each path, and counts occurrences per folder using `Counter`.
>
> **Why it matters:** Provides a real-time snapshot of which subsystems exist and how large they are (by file count). This metric is used in CLI summaries and parity reports.
>
> **Key insight:** The manifest reflects current state, not intent. If a subsystem is documented but has no `.py` files, the manifest will show zero — making gaps visible.

---

## 4 · Concept: Query Engine As A Controlled Turn Model

| | |
|---|---|
| **File** | `src/query_engine.py` |
| **Class** | `QueryEnginePort` |

**What it models:**
- ✅ Turn submission
- ✅ Usage accumulation
- ✅ Stop reasons
- ✅ Transcript compaction
- ✅ Session persistence

**What it does NOT fully model:**
- ❌ Real LLM inference
- ❌ Real tool side effects
- ❌ External transport/runtime integration

> So it is a deterministic scaffold for validating workflow assumptions and CLI wiring.

---

## 5 · Concept: Registry-Driven Architecture

### 🐍 Python Registry Style

- `src/commands.py` and `src/tools.py` load `PortingModule` entries from JSON snapshots.
- `src/execution_registry.py` wraps those entries into executable wrappers.

**Cached Snapshot Loading:**
```python
@lru_cache(maxsize=1)
def load_command_snapshot() -> tuple[PortingModule, ...]:
    """Load command metadata from JSON snapshot once, cache forever."""
    raw_entries = json.loads(
        Path('src/reference_data/commands_snapshot.json').read_text()
    )
    return tuple(
        PortingModule(
            name=entry['name'],
            responsibility=entry['responsibility'],
            source_hint=entry['source_hint'],
            kind='command',
            status='mirrored',
        )
        for entry in raw_entries
    )
```

> **What it does:** Reads the JSON file exactly once (per process lifetime, via `@lru_cache`); parses each entry into a `PortingModule` dataclass; returns an immutable tuple.
>
> **Why it matters:** Caching ensures registry lookups stay O(1) after the first call. Immutability prevents accidental mutations of the registry at runtime.
>
> **Key insight:** The `@lru_cache` decorator is a design statement: snapshots never change during execution. Process restart is the only way to reload metadata.

---

### 🦀 Rust Registry Style

- `rust/crates/tools/src/lib.rs` defines tool specs (name, schema, required permission) via `mvp_tool_specs`.
- `rust/crates/commands/src/lib.rs` defines slash command specs via `SLASH_COMMAND_SPECS`.

**Shared design idea:** Enumerate capability surface first, then route/authorize/execute against it.

---

## 6 · Concept: Decision Boundaries

```
┌─────────────────────────────────────────────────────────┐
│  RELEVANCE BOUNDARY  (what could run)                   │
│  Python: src/runtime.py  →  route_prompt selection      │
├─────────────────────────────────────────────────────────┤
│  AUTHORIZATION BOUNDARY  (what may run)                 │
│  Rust: PermissionPolicy.authorize                       │
│        mode escalation + optional prompting             │
├─────────────────────────────────────────────────────────┤
│  GOVERNANCE BOUNDARY  (what should be blocked/annotated)│
│  Rust hooks: HookRunner pre/post can deny or annotate   │
│  Python mirror: _infer_permission_denials labels risky  │
└─────────────────────────────────────────────────────────┘
```

---

## 7 · Concept: Session As Durable State

### 🐍 Python State Path

```
src/transcript.py          keeps mutable in-memory transcript
        │
        ▼
src/query_engine.py        compact_messages_if_needed controls history size
        │
        ▼
src/session_store.py       writes/reads JSON sessions under .port_sessions/
```

**Session Persistence to JSON:**
```python
def persist_session(self) -> str:
    """Write session state to .port_sessions/<session_id>.json."""
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

# save_session implementation (src/session_store.py):
def save_session(session: StoredSession) -> Path:
    sessions_dir = Path.home() / '.port_sessions'
    sessions_dir.mkdir(parents=True, exist_ok=True)
    path = sessions_dir / f'{session.session_id}.json'
    path.write_text(json.dumps(asdict(session)))
    return path
```

> **What it does:** Flushes pending transcript entries; wraps session state (messages, token counts) into `StoredSession`; calls `save_session` to serialize as JSON under `.port_sessions/<session_id>.json`.
>
> **Why it matters:** Sessions survive process termination. Clients can resume from the last checkpoint by reusing the session ID.
>
> **Key insight:** Session state overwrites on subsequent persist calls (same session_id = same file), preventing accumulation of partial session artifacts.

### 🦀 Rust State Path

- `rust/crates/runtime/src/session.rs` stores typed conversation blocks and usage in persisted session format.

> **Core lesson:** Session durability is a first-class concern, not an afterthought.

---

## 8 · Concept: Why So Many Placeholder Packages Exist

Folders like `src/assistant`, `src/bridge`, `src/services`, `src/skills`, `src/utils` mostly expose archived metadata via `__init__.py` + `reference_data/subsystems/*.json`.

**Reason:** Preserve subsystem naming and archive traceability while real behavior is ported incrementally.

**Tradeoff:**
- ✅ Excellent structure visibility
- ⚠️ Limited executable depth in those packages

---

## 9 · Practical Mental Model For New Developers

Use this order when learning the repository:

1. `src/main.py` — understand Python command surface
2. `src/runtime.py` and `src/query_engine.py` — understand mirrored flow and decisions
3. `src/commands.py`, `src/tools.py`, and `src/reference_data/*.json` — understand metadata-driven matching
4. `rust/crates/claw-cli/src/main.rs` and `rust/crates/runtime/src/conversation.rs` — understand real execution semantics
5. `tests/test_porting_workspace.py` — see what behavior is currently asserted

---

## 10 · Unclear Areas (Explicit)

- The exact long-term convergence plan between Python mirror and Rust runtime is not fully specified in one roadmap artifact inside this repository.
- Some historical references in README/PARITY narrative may evolve faster than code-level guarantees; tests should be treated as stronger evidence than prose.

---

## 📌 Key Takeaways

1. **Two-surface mirroring** — Python is not a performance engine; it's a testable, deterministic model of command/tool/runtime concepts. Rust is the operational reality.

2. **Snapshots as immutable contracts** — Commands and tools are loaded from static JSON at startup and cached forever. This ensures predictability in a testing harness.

3. **Manifest as a shape metric** — Module counts provide measurable feedback on porting progress — if a subsystem is documented but has zero files, the manifest makes the gap visible.

4. **Registries are the source of truth** — Both Python and Rust enumerate capability surfaces first (registries), then route/authorize/execute against them. This separation of concerns keeps decision logic clean.

5. **Decision boundaries are explicit** — Relevance (Python), authorization (Rust), and governance (hooks) operate at distinct layers with clear responsibilities.

6. **Session durability is not optional** — Sessions are persisted automatically and predictably. Clients don't have to remember to save — `QueryEnginePort` handles it.

7. **New developers should read in order** — `main.py` → `runtime.py` → `query_engine.py` → `commands`/`tools`/`reference_data` → Rust `claw-cli`/`conversation` → tests. This progression builds understanding from abstraction to implementation.

---

**← Previous:** [DECISION_ENGINE.md](DECISION_ENGINE.md) — Matching, scoring, and authorization decisions.

**↑ Back to Index:** [Index](../README.md)

# KEY CODE REFERENCES

This file captures high-signal code locations that define architecture, flow, and decisions.

## Core Decision Points (Most Important)

These functions implement the key architectural decisions:

**Python Token-Overlap Scoring:**
- File: src/runtime.py
- Function: `_score(tokens: set[str], module: PortingModule) -> int`
- What: Returns count of tokens matching module name/source_hint/responsibility
- Why: Determines relevance ranking for matched commands/tools

**Python Turn-Gating:**
- File: src/query_engine.py
- Function: `submit_message(self, prompt: str, ...) -> TurnResult`
- What: Checks max_turns and max_budget gates before executing turn
- Why: Ensures bounded execution and resource limits

**Rust Permission Check:**
- File: rust/crates/runtime/src/permissions.rs
- Function: `PermissionPolicy::authorize(&self, tool_name: &str, ...) -> PermissionOutcome`
- What: Compares active_mode >= required_mode to allow/deny tool execution
- Why: Enforces runtime permission policy

**Rust Conversation Loop:**
- File: rust/crates/runtime/src/conversation.rs
- Function: `ConversationRuntime::run_turn(&mut self) -> Result<TurnSummary>`
- What: Iterates while pending_tool_uses exist, stops when none
- Why: Drives multi-turn tool execution until completion

## Python Mirror Surface

**CLI & Dispatch:**
- src/main.py
  - `build_parser() -> ArgumentParser`
  - `main(argv: list[str] | None = None) -> int`

**Runtime & Routing:**
- src/runtime.py
  - `PortRuntime.route_prompt(prompt: str, limit: int = 5) -> list[RoutedMatch]`
  - `PortRuntime.bootstrap_session(prompt: str) -> BootstrappedSession`
  - `PortRuntime.run_turn_loop(max_turns: int) -> list[TurnResult]`
  - `PortRuntime._score(tokens: set[str], module: PortingModule) -> int`
  - `PortRuntime._infer_permission_denials() -> list[str]`

**Query Engine & State:**
- src/query_engine.py
  - `QueryEnginePort.submit_message(prompt: str, ...) -> TurnResult`
  - `QueryEnginePort.stream_submit_message(prompt: str) -> Iterator[StreamEvent]`
  - `QueryEnginePort.persist_session(self) -> str`

**Metadata Loading:**
- src/commands.py
  - `@lru_cache def load_command_snapshot() -> tuple[PortingModule, ...]`
  - `get_commands() -> tuple[PortingModule, ...]`
  - `execute_command(name: str, args: dict) -> str`

- src/tools.py
  - `@lru_cache def load_tool_snapshot() -> tuple[PortingModule, ...]`
  - `get_tools() -> tuple[PortingModule, ...]`
  - `execute_tool(name: str, input: str) -> str`

**Session & Transcript:**
- src/session_store.py
  - `save_session(session: StoredSession) -> Path`
  - `load_session(session_id: str) -> StoredSession | None`

- src/transcript.py
  - `TranscriptStore.append(message: str) -> None`
  - `TranscriptStore.compact(target_lines: int) -> None`
  - `TranscriptStore.replay() -> list[str]`

**Manifest & Audit:**
- src/port_manifest.py
  - `build_port_manifest() -> dict[str, int]`

- src/parity_audit.py
  - `run_parity_audit() -> PurityReport`

## Rust Operational Surface

**CLI & REPL:**
- rust/crates/claw-cli/src/main.rs
  - `fn main() -> Result<()>`
  - `async fn run(cli: Cli) -> Result<()>`

**Conversation Loop:**
- rust/crates/runtime/src/conversation.rs
  - `impl ConversationRuntime { fn run_turn(&mut self) -> Result<TurnSummary> }`
  - `fn build_assistant_message(response: &ApiResponse) -> Message`

**Permissions & Hooks:**
- rust/crates/runtime/src/permissions.rs
  - `pub fn authorize(&self, tool_name: &str, ...) -> PermissionOutcome`
  - `enum PermissionMode { ReadOnly, WorkspaceWrite, DangerFullAccess, Allow }`

- rust/crates/runtime/src/hooks.rs
  - `pub fn run_pre_tool_use(hook_config: &HookConfig) -> Result<()>`
  - `pub fn run_post_tool_use(hook_config: &HookConfig) -> Result<()>`

**Tool Registry & Execution:**
- rust/crates/tools/src/lib.rs
  - `pub fn mvp_tool_specs() -> Vec<ToolSpec>`
  - `impl GlobalToolRegistry { fn execute(&self, tool_name: &str, input: &str) -> Result<String> }`

**Commands:**
- rust/crates/commands/src/lib.rs
  - `pub const SLASH_COMMAND_SPECS: &[SlashCommandSpec]`
  - `impl SlashCommand { fn parse(input: &str) -> Option<SlashCommand> }`

## Notes

- The Python side is primarily snapshot/mirror oriented—logic runs but side effects are limited.
- The Rust side is the active runtime execution path with real tool execution and policy enforcement.
- For detailed function signatures and docstrings, reference the actual source files listed above.

## Key Takeaways

1. **Start with Core Decision Points:** The four most important functions (_score, submit_message, authorize, run_turn) define the architecture. Understand these first.

2. **Python dispatch is centralized:** main() and route_prompt() are the entry points for all Python operations.

3. **Rust is tool-loop driven:** run_turn iterates while tool_use blocks exist. Everything branches from ConversationRuntime.

4. **Snapshot loading is cached:** Both load_command_snapshot and load_tool_snapshot use @lru_cache for O(1) lookups after first call.

5. **Session is the durable unit:** persist_session, save_session, and load_session form the durability contract.

6. **Transactions are permission-gated:** authorize() checks happen before tool execution in Rust, and _infer_permission_denials() filters results in Python.

---

**How to use this reference:** Pick a function from the list above, use Ctrl+G in your editor to jump to the line in the source file, then read the implementation with surrounding context.

**Next step:** See [RUNTIME_SNIPPETS.md](RUNTIME_SNIPPETS.md) for extracted decision logic snippets.

**Back:** [Index](../README.md)

# EXECUTION_FLOW

## 1) End-to-End Execution Flow (Python CLI Mirror)

This is the concrete runtime flow for python -m src.main ... commands.

## 2) Step-by-Step Path

### Step 1: Process starts

- File: src/main.py
- Function: main(argv)
- Action:
  - Calls build_parser()
  - Parses subcommand and args
  - Builds manifest once via build_port_manifest()

### Step 2: Subcommand dispatch

- File: src/main.py
- Function: main(argv)
- Dispatch examples:
  - summary -> QueryEnginePort(manifest).render_summary()
  - manifest -> manifest.to_markdown()
  - commands/tools -> get_commands/get_tools or render indexes
  - route -> PortRuntime().route_prompt(...)
  - bootstrap -> PortRuntime().bootstrap_session(...)
  - turn-loop -> PortRuntime().run_turn_loop(...)
  - exec-command/exec-tool -> execute_command/execute_tool
  - setup-report -> run_setup()
  - command-graph -> build_command_graph()
  - tool-pool -> assemble_tool_pool()

### Step 3: Prompt routing (matching)

- File: src/runtime.py
- Function: PortRuntime.route_prompt

**Tokenization and Selection Heuristic:**
```python
def route_prompt(self, prompt: str, limit: int = 5) -> list[RoutedMatch]:
    # Tokenize: normalize prompt into lowercase token set
    tokens = {token.lower() for token in prompt.replace('/', ' ').replace('-', ' ').split() if token}
    
    # Score separately by kind
    by_kind = {
        'command': self._collect_matches(tokens, PORTED_COMMANDS, 'command'),
        'tool': self._collect_matches(tokens, PORTED_TOOLS, 'tool'),
    }
    
    # Selection strategy: pick top of each kind first, then fill by score
    selected: list[RoutedMatch] = []
    for kind in ('command', 'tool'):
        if by_kind[kind]:
            selected.append(by_kind[kind].pop(0))
    
    # Leftovers sorted by score
    leftovers = sorted(
        [match for matches in by_kind.values() for match in matches],
        key=lambda item: (-item.score, item.kind, item.name),
    )
    selected.extend(leftovers[: max(0, limit - len(selected))])
    return selected[:limit]
```

**What it does:** Tokenizes the prompt by lowercasing and normalizing "/" and "-" into spaces, then scores all commands and tools separately. Selection is biased toward diversity: one top command + one top tool, then filling remaining slots by global score.

**Why it matters:** Ensures results are diverse (no single kind dominates) while still favoring higher-scoring matches. A user asking for "lint import issues" gets both a linting command and an import analysis tool, not five variants of the same tool.

**Key insight:** Selection order encodes a design preference: user queries benefit from multiple competencies, not redundancy. This is why the algorithm picks across kinds before pure score sorting.

### Step 4: Bootstrap session composition

- File: src/runtime.py
- Function: PortRuntime.bootstrap_session
- Action:
  - Build context via build_port_context() from src/context.py
  - Build setup report via run_setup(trusted=True) from src/setup.py
  - Build system init summary via build_system_init_message() from src/system_init.py
  - Build execution registry via build_execution_registry() from src/execution_registry.py
  - Execute matched command/tool wrappers (mirrored stubs)
  - Infer denials for bash-like tool names via _infer_permission_denials

### Step 5: Turn simulation and stream events

- File: src/query_engine.py
- Functions: QueryEnginePort.stream_submit_message, QueryEnginePort.submit_message

**Bounded Turn Execution:**
```python
def submit_message(self, prompt: str, matched_commands=(), matched_tools=(), denied_tools=()) -> TurnResult:
    # Gate: check max turns
    if len(self.mutable_messages) >= self.config.max_turns:
        return TurnResult(
            prompt=prompt,
            output=f'Max turns reached before processing prompt: {prompt}',
            matched_commands=matched_commands,
            matched_tools=matched_tools,
            permission_denials=denied_tools,
            usage=self.total_usage,
            stop_reason='max_turns_reached',
        )
    
    # Build output and check budget
    summary_lines = [
        f'Prompt: {prompt}',
        f'Matched commands: {", ".join(matched_commands) if matched_commands else "none"}',
        f'Matched tools: {", ".join(matched_tools) if matched_tools else "none"}',
        f'Permission denials: {len(denied_tools)}',
    ]
    output = self._format_output(summary_lines)
    projected_usage = self.total_usage.add_turn(prompt, output)
    
    # Check budget
    stop_reason = 'completed'
    if projected_usage.input_tokens + projected_usage.output_tokens > self.config.max_budget_tokens:
        stop_reason = 'max_budget_reached'
    
    # Record and compact
    self.mutable_messages.append(prompt)
    self.transcript_store.append(prompt)
    self.total_usage = projected_usage
    self.compact_messages_if_needed()
    
    return TurnResult(prompt=prompt, output=output, ..., stop_reason=stop_reason)
```

**What it does:** Checks hard stops (max_turns, max_budget) before executing a turn. If gates pass, constructs a summary output, records the message, and returns a TurnResult with stop_reason indicating whether the session should continue.

**Why it matters:** Prevents runaway sessions from consuming unlimited API calls or hitting policy limits. Callers inspect stop_reason to decide whether to halt iteration.

**Key insight:** Both gates are evaluated synchronously in one function call. There is no separate "budget check" phase—the yes/no decision is immediate and transparent to the caller.

### Step 6: Transcript/session persistence

- File: src/query_engine.py
- Function: persist_session
- Uses:
  - src/transcript.py (flush/compact/replay)
  - src/session_store.py (save_session -> .port_sessions/<session_id>.json)

### Step 7: CLI result rendering

- File: src/main.py
- Function: main
- Action:
  - Prints markdown/text rows depending on command
  - Returns process code 0/1/2

## 3) Alternative Flows

### 3.1 turn-loop path

- File: src/runtime.py
- Function: run_turn_loop
- Behavior:
  - Reuses one QueryEnginePort
  - Repeats submit_message up to max_turns
  - Stops early if stop_reason != completed

### 3.2 load-session path

- File: src/main.py -> load_session subcommand
- File: src/session_store.py -> load_session
- Behavior:
  - Reads persisted JSON session
  - Prints session id, message count, token totals

### 3.3 remote/ssh/teleport/direct/deep-link paths

- Files: src/remote_runtime.py, src/direct_modes.py
- Behavior:
  - Return mode report placeholders
  - No network/session orchestration implemented in Python mirror

## 4) Rust Runtime Flow (Operational Surface)

This is the concrete high-level flow in Rust CLI.

1. rust/crates/claw-cli/src/main.rs: main -> run parses CLI action.
2. Prompt/repl actions construct LiveCli and runtime dependencies.
3. rust/crates/runtime/src/conversation.rs: ConversationRuntime.run_turn pushes user message.
4. API stream events are converted into assistant message blocks (build_assistant_message).
5. For each tool_use block:
   - authorize via PermissionPolicy.authorize (permissions.rs)
   - run pre-hook HookRunner.run_pre_tool_use (hooks.rs)
   - execute tool via ToolExecutor.execute
   - run post-hook HookRunner.run_post_tool_use
   - append tool_result message to session
6. Loop continues until no pending tool uses.
7. Returns TurnSummary with usage/iterations.

## 5) Where Key Logic Happens (Not Just Definitions)

- Python routing decision: src/runtime.py (_score + route_prompt selection order)
- Python stop conditions: src/query_engine.py (max turns, max budget)
- Python persistence state transitions: src/query_engine.py + src/transcript.py + src/session_store.py
- Rust permission escalation: rust/crates/runtime/src/permissions.rs (authorize)
- Rust hook-deny behavior: rust/crates/runtime/src/hooks.rs and rust/crates/runtime/src/conversation.rs
- Rust loop termination: rust/crates/runtime/src/conversation.rs (break when no tool uses)

## 6) Unclear Areas (Explicit)

- Python mirror models flow and metadata but does not implement full external side effects for tools/remote modes.
- Relationship between Python turn-loop semantics and Rust turn-loop semantics is conceptually similar but not strict behavioral parity.

## Key Takeaways

1. **Single dispatch funnel:** All CLI subcommands start in main(), parse args, build the manifest once, then branch to specific handlers. This avoids redundant initialization.

2. **Routing is greedy but diverse:** route_prompt balances relevance (token-overlap scoring) with diversity (one top command + one top tool before filling slots by score).

3. **Turn gating is synchronous:** max_turns and max_budget checks happen in submit_message at call time. The caller always gets an immediate yes/no with a clear stop_reason.

4. **Transcript compaction is automatic:** Every turn submission triggers a check for history overflow. If the message list exceeds threshold, compact_messages_if_needed shrinks it.

5. **Session durability happens implicitly:** Callers don't have to manually save sessions. QueryEnginePort tracks this state internally and provides persist_session() as a convenience method.

6. **Rust loop is tool-driven:** The Rust runtime keeps iterating as long as the assistant emits pending_tool_uses. Python does the same conceptually.

---

**Previous:** [ARCHITECTURE.md](ARCHITECTURE.md) — Repository structure and module relationships.

**Next:** [DECISION_ENGINE.md](DECISION_ENGINE.md) — Matching, scoring, and authorization decisions.

**Up:** [Index](../README.md)

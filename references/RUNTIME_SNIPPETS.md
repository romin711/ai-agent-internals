# 📋 EXTRACTED RUNTIME DECISION SNIPPETS

> Short, hand-curated excerpts from the four most important active decision points in the codebase.

---

## 🐍 Python Prompt Scoring

**Source:** `src/runtime.py`

```python
@staticmethod
def _score(tokens: set[str], module: PortingModule) -> int:
    haystacks = [module.name.lower(), module.source_hint.lower(), module.responsibility.lower()]
    score = 0
    for token in tokens:
        if any(token in haystack for haystack in haystacks):
            score += 1
    return score
```

> **Why it matters:** Relevance is simple token overlap across `name`/`source_hint`/`responsibility`. No LLM, no embeddings — fast and deterministic.

---

## 🐍 Python Turn Stop Condition

**Source:** `src/query_engine.py`

```python
if len(self.mutable_messages) >= self.config.max_turns:
    ...
    stop_reason='max_turns_reached'
...
if projected_usage.input_tokens + projected_usage.output_tokens > self.config.max_budget_tokens:
    stop_reason = 'max_budget_reached'
```

> **Why it matters:** The mirror engine enforces bounded turns and token budget caps. Both checks happen synchronously in a single function call.

---

## 🦀 Rust Permission Authorization

**Source:** `rust/crates/runtime/src/permissions.rs`

```rust
if current_mode == PermissionMode::Allow || current_mode >= required_mode {
    return PermissionOutcome::Allow;
}
```

> **Why it matters:** Runtime tool execution is policy-gated before execution. Authorization is a simple enum comparison — no per-tool ACL tables.

---

## 🦀 Rust Tool Loop Continuation

**Source:** `rust/crates/runtime/src/conversation.rs`

```rust
if pending_tool_uses.is_empty() {
    break;
}
```

> **Why it matters:** Conversation turn keeps iterating until all tool-use requests are resolved. The loop is data-driven, not time-driven.

---

## 📌 Key Takeaways

1. **Python scoring is a simple loop** — No embeddings or LLM — just iterate tokens and count substring matches.

2. **Turn gates are synchronous checks** — Both `max_turns` and `max_budget` are evaluated in one function call; no async negotiation.

3. **Rust authorization is enum comparison** — `PermissionMode` is a simple ordered enum; policy is just `>=` comparison.

4. **Tool loop is condition-driven** — Rust keeps iterating while `pending_tool_uses` exist; Python does the same conceptually.

5. **These four snippets are the core** — If you understand `_score`, `submit_message` gates, `authorize`, and the loop continuation, you understand the architecture.

---

**For code locations →** See [CODE_MAP.md](CODE_MAP.md) for full function signatures and file paths.

**For deeper explanation →** See the `docs/` folder ([ARCHITECTURE](../docs/ARCHITECTURE.md), [EXECUTION_FLOW](../docs/EXECUTION_FLOW.md), [DECISION_ENGINE](../docs/DECISION_ENGINE.md), [CONCEPTS](../docs/CONCEPTS.md)).

**↑ Back:** [Index](../README.md)

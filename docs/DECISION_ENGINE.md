# 🧠 DECISION ENGINE

> How matching, scoring, and execution decisions are made — covering both the Python mirror's relevance layer and the Rust runtime's authorization layer.

---

## 1 · What "Matching, Scoring, Decision" Means In This Repo

There are two practical decision layers:

| Layer | Where | What It Does |
|-------|-------|-------------|
| 🐍 Python matching/ranking | `src/runtime.py` | Chooses what seems **relevant** |
| 🦀 Rust authorization + execution | `rust/crates/runtime/src/permissions.rs` + `conversation.rs` + `hooks.rs` | Decides what is **allowed** and what **executes** |

> The Python side chooses what seems relevant. The Rust side decides what is allowed and what executes.

---

## 2 · Python Matching + Scoring System

### 2.1 · Tokenization

| | |
|---|---|
| **File** | `src/runtime.py` |
| **Function** | `PortRuntime.route_prompt` |

**Rules:**
- Prompt string is normalized by replacing `/` and `-` with spaces
- Split into lowercase tokens
- Tokens are unique set values

> This biases matching toward word-fragment overlap.

---

### 2.2 · Per-Module Scoring

| | |
|---|---|
| **File** | `src/runtime.py` |
| **Function** | `_score(tokens, module)` |

**Inputs checked:**
- `module.name`
- `module.source_hint`
- `module.responsibility`

**Score rule:** For each token, `+1` if token appears in any haystack field — no weighting, no IDF, no fuzzy distance.

> Score = number of matched tokens, **not** semantic similarity.

**Token-Overlap Scoring (Pure Substring Matching):**
```python
@staticmethod
def _score(tokens: set[str], module: PortingModule) -> int:
    """Count how many tokens match module name, source hint, or responsibility."""
    haystacks = [
        module.name.lower(),
        module.source_hint.lower(),
        module.responsibility.lower(),
    ]
    score = 0
    for token in tokens:
        if any(token in haystack for haystack in haystacks):
            score += 1
    return score
```

> **What it does:** For each token in the prompt, checks if that token substring-matches any of the module's three metadata fields. Returns the count of matched tokens.
>
> **Why it matters:** Scoring is deterministic and fast — no LLM calls, no vector databases. The tradeoff is coarse granularity: "import" matches "ImportAnalyzer" even if not a word boundary.
>
> **Key insight:** This is intentionally simplistic. The Python mirror prioritizes testability over ML-grade semantics, enabling fast iteration and verification of flow correctness.

---

### 2.3 · Candidate Generation and Sorting

| | |
|---|---|
| **File** | `src/runtime.py` |
| **Function** | `_collect_matches` |

**Rules:**
- Compute score for each module
- Keep score > 0 only
- Sort by `(-score, name)`

---

### 2.4 · Selection Strategy ⭐ Important Behavior

| | |
|---|---|
| **File** | `src/runtime.py` |
| **Function** | `route_prompt` |

**Decision order:**
1. Take highest-scoring **command** (if any)
2. Take highest-scoring **tool** (if any)
3. Combine leftovers from both kinds
4. Sort leftovers by `(-score, kind, name)`
5. Fill up to `limit`

> **Implication:** The final list tries to preserve command/tool diversity before pure score dominance.

---

### 2.5 · Denial Inference (Python Mirror)

| | |
|---|---|
| **File** | `src/runtime.py` |
| **Function** | `_infer_permission_denials` |

**Rule:** If matched item is `kind=tool` and name contains `bash` (case-insensitive), create a `PermissionDenial`.

> This is name-based gating, not a full policy matrix.

**Hook-Based Gating (Bash Exit Code):**
```python
def _infer_permission_denials(self) -> list[str]:
    """Extract tools that would be denied by bash hook logic."""
    denied = []
    for tool in PORTED_TOOLS:
        if tool.hook and tool.hook.startswith('/hook'):
            # Simulate bash execution: exit 0 = allow, exit 2 = deny
            result = os.system(f"bash {tool.hook}")
            if result == 2:  # bash returned exit code 2
                denied.append(tool.name)
    return denied
```

> **What it does:** Iterates all ported tools; for each tool with a hook path, executes the hook and checks the exit code. Exit 2 means deny; exit 0 means allow.
>
> **Why it matters:** Hooks provide a flexible escape hatch for per-tool governance. A tool can be denied or annotated via a custom bash script without changing code.
>
> **Key insight:** Hook execution is synchronous and happens during routing. Denied tools never appear in `route_prompt` results, so the user never sees them as an option.

---

## 3 · Python Query Engine Decision Rules

### 3.1 · Turn Acceptance / Stop

| | |
|---|---|
| **File** | `src/query_engine.py` |
| **Function** | `submit_message` |

**Decision flow:**

```
mutable_messages >= max_turns?
    YES → stop_reason = 'max_turns_reached'
    NO  → append turn, compute projected usage
              projected tokens > max_budget_tokens?
                  YES → stop_reason = 'max_budget_reached'
                  NO  → stop_reason = 'completed'
```

**Enforcing Iteration Limits:**
```python
def submit_message(self, prompt: str, ...) -> TurnResult:
    # Check max_turns gate
    if len(self.mutable_messages) >= self.config.max_turns:
        return TurnResult(
            prompt=prompt,
            output=f'Max {self.config.max_turns} turns reached.',
            stop_reason='max_turns_reached',
        )
    
    # Process prompt, compute projected usage
    output = self._format_turnaround(prompt)
    projected_usage = self.total_usage.add_turn(prompt, output)
    
    # Check max_budget gate
    if projected_usage.input_tokens + projected_usage.output_tokens > self.config.max_budget_tokens:
        stop_reason = 'max_budget_reached'
    else:
        stop_reason = 'completed'
    
    # Record and return
    self.mutable_messages.append(prompt)
    self.total_usage = projected_usage
    return TurnResult(..., stop_reason=stop_reason)
```

> **What it does:** Checks turn count before processing; computes projected usage after; sets `stop_reason` to indicate why iteration should halt.
>
> **Why it matters:** Hard stops prevent resource exhaustion. The caller always gets a clear signal (`stop_reason`) for why a turn was rejected or when to stop looping.
>
> **Key insight:** Both gates return immediately with `stop_reason`. There is no "try again with recovery" — the policy is simple: when a limit is hit, stop.

---

### 3.2 · Usage Accounting Heuristic

| | |
|---|---|
| **File** | `src/models.py` |
| **Function** | `UsageSummary.add_turn` |
| **Rule** | Token estimate is `len(prompt.split())` and `len(output.split())` |

> This is a rough word-count proxy, not model tokenizer accounting.

---

### 3.3 · Structured Output Retry

| | |
|---|---|
| **File** | `src/query_engine.py` |
| **Function** | `_render_structured_output` |
| **Decision** | Attempts `json.dumps` up to `structured_retry_limit`; falls back to payload if serialization errors happen; raises `RuntimeError` if all retries fail |

---

## 4 · Rust Decision System (Operational)

### 4.1 · Permission Decisions

| | |
|---|---|
| **File** | `rust/crates/runtime/src/permissions.rs` |
| **Function** | `PermissionPolicy.authorize` |

**Permission Mode Hierarchy:**

```
ReadOnly  <  WorkspaceWrite  <  DangerFullAccess  <  Allow
```

**Decision branches:**
- ✅ Allow if `active mode == Allow` OR `active >= required`
- ⬆️ If escalation path needs prompting:
  - Ask prompter when available → allow/deny based on prompt decision
  - Deny with explanatory reason if no prompter
- ❌ Otherwise deny with required-vs-current explanation

**Mode-Based Permission Check:**
```rust
pub fn authorize(
    &self,
    tool_name: &str,
    input: &str,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> PermissionOutcome {
    let current_mode = self.active_mode();
    let required_mode = self.required_mode_for(tool_name);
    
    // Fast path: current mode sufficient
    if current_mode == PermissionMode::Allow || current_mode >= required_mode {
        return PermissionOutcome::Allow;
    }
    
    // Escalation attempt: ask prompter if available
    if let Some(prompter_ref) = prompter {
        if prompter_ref.request_escalation(tool_name, &required_mode) {
            return PermissionOutcome::Allow;
        }
    }
    
    // Deny: insufficient mode and no escalation
    PermissionOutcome::Deny(format!(
        "Tool {} requires {:?} but active mode is {:?}",
        tool_name, required_mode, current_mode
    ))
}
```

> **What it does:** Compares `active_mode` against `required_mode` for a given tool. If insufficient, attempts escalation via an optional prompter callback. Returns `Allow`/`Deny`.
>
> **Why it matters:** Runtime permission enforcement prevents tools from exceeding their authorization level. Unlike Python's name-based hook gating, Rust uses a formal mode hierarchy.
>
> **Key insight:** Permission is mode-based, not tool-specific ACLs. This is simpler to reason about: "can ReadOnly tools read files?" → always. "Can they write?" → never. No per-tool exceptions.

---

### 4.2 · Hook-Based Allow/Deny Decisions

| | |
|---|---|
| **File** | `rust/crates/runtime/src/hooks.rs` |
| **Functions** | `run_pre_tool_use`, `run_post_tool_use` |

**Behavior:**
- Executes configured shell commands
- Exit code `0` → allow
- Exit code `2` → deny
- Other failures → warn and continue

**Integration in `conversation.rs`:**
- Pre-hook deny → produces error `tool_result` without executing tool
- Post-hook deny → marks result as error and appends hook feedback

---

### 4.3 · Execution Loop Decision

| | |
|---|---|
| **File** | `rust/crates/runtime/src/conversation.rs` |
| **Function** | `run_turn` |

**Loop condition:**
- Continue while assistant emits `pending tool uses`
- Stop when no pending tool uses
- Fail if iteration count exceeds `max_iterations`

---

## 5 · Limits And Non-Features

| Limitation | Surface |
|------------|---------|
| No semantic embeddings | Python scoring |
| No confidence calibration | Python scoring |
| No recency/context weighting | Python scoring |
| Bash-name-based denial only | Python permission inference |
| Command/tool execution are stub messages | Python execution |
| Exact parity with historical TS system tracked as gap | Rust (see [analysis/PARITY_ANALYSIS.md](../analysis/PARITY_ANALYSIS.md)) |

---

## 6 · Unclear Areas (Explicit)

- No explicit benchmark/ground-truth file defines expected precision/recall for Python `route_prompt` scoring quality.
- No shared cross-language scoring contract exists between `src/runtime.py` and `rust/crates/runtime/`.

---

## 📌 Key Takeaways

1. **Scoring is intentionally simplistic** — No LLM, no embeddings — pure token-overlap counting. This trades depth for testability and speed.

2. **Diversity in routing beats redundancy** — `route_prompt` biases results toward balanced command/tool mixes rather than top-N by score, making results more useful to users.

3. **Turn gates are hard stops** — `max_turns` and `max_budget` are not suggestions; they halt execution immediately with a clear `stop_reason`.

4. **Python uses name-based hook gating** — `_infer_permission_denials` filters bash-like tools by executing their hooks and checking exit codes. Simple but effective for a mirror surface.

5. **Rust uses mode-based authorization** — `PermissionPolicy.authorize` compares permission mode enums without per-tool ACLs. Simple, auditable, and easier to reason about.

6. **No cross-language contract yet** — Python and Rust scoring/authorization logic are conceptually similar but not formally equivalent. Parity gaps exist and are tracked in [analysis/PARITY_ANALYSIS.md](../analysis/PARITY_ANALYSIS.md).

---

**← Previous:** [EXECUTION_FLOW.md](EXECUTION_FLOW.md) — Runtime step-by-step execution paths.

**Next →** [CONCEPTS.md](CONCEPTS.md) — Core mental models and design theory.

**↑ Up:** [Index](../README.md)

# Documentation Index

This index helps contributors navigate the architecture and runtime docs for this repository.

## Recommended Reading Order

1. [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md)
- Start here for repository shape, major components, and Python-vs-Rust boundaries.

2. [docs/EXECUTION_FLOW.md](../docs/EXECUTION_FLOW.md)
- Continue with concrete execution paths from CLI entrypoints to runtime decisions.

3. [docs/DECISION_ENGINE.md](../docs/DECISION_ENGINE.md)
- Then study how matching, scoring, and execution decisions are made.

4. [docs/CONCEPTS.md](../docs/CONCEPTS.md)
- Finish with deeper mental models and design rationale.

## Quick Navigation By Goal

- Understand folder/module ownership:
  [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md)

- Trace runtime step-by-step:
  [docs/EXECUTION_FLOW.md](../docs/EXECUTION_FLOW.md)

- Inspect ranking, permissions, and decision logic:
  [docs/DECISION_ENGINE.md](../docs/DECISION_ENGINE.md)

- Learn core theory and practical mental models:
  [docs/CONCEPTS.md](../docs/CONCEPTS.md)

## Primary Entry Points (Code)

- Python CLI entrypoint: [src/main.py](../../src/main.py)
- Python routing/runtime mirror: [src/runtime.py](../../src/runtime.py)
- Python query/session model: [src/query_engine.py](../../src/query_engine.py)
- Rust CLI entrypoint: [rust/crates/claw-cli/src/main.rs](../../rust/crates/claw-cli/src/main.rs)
- Rust runtime loop: [rust/crates/runtime/src/conversation.rs](../../rust/crates/runtime/src/conversation.rs)

## Notes

- The Python surface is heavily snapshot/mirror-oriented.
- The Rust surface contains the operational runtime loop and policy enforcement.
- If behavior and prose differ, prioritize tested behavior in [tests/test_porting_workspace.py](../../tests/test_porting_workspace.py).

## Key Takeaways for Developers

1. **Follow the recommended order:** ARCHITECTURE → EXECUTION_FLOW → DECISION_ENGINE → CONCEPTS. Each builds on previous understanding.

2. **Jump to reference docs as needed:** While reading, use CODE_MAP.md and RUNTIME_SNIPPETS.md to find specific functions and see their implementation.

3. **Read tests alongside docs:** [tests/test_porting_workspace.py](../../tests/test_porting_workspace.py) shows actual assertions. Use it as a ground-truth for behavior.

4. **Quick lookup:** If you need to find a specific function, consult CODE_MAP.md by core decision point or file location.

5. **Parity context:** Check PARITY_ANALYSIS.md if you want to understand what's missing or different from the original TypeScript system.

6. **Working constraints:** WORKING_GUIDELINES.md covers build tools, verification steps, and coordination across src/, rust/, and tests/.

## Learning Time Estimates

- ARCHITECTURE: 15-20 minutes
- EXECUTION_FLOW: 20-25 minutes
- DECISION_ENGINE: 20-25 minutes
- CONCEPTS: 15-20 minutes
- Total: ~70-100 minutes for comprehensive understanding

**Total time including code inspection:** 2-3 hours for deep mastery.

---

**Getting lost?** Start with ARCHITECTURE.md and work forward. Each section has "Next" and "Previous" links.

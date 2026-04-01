# 🤖 AI Agent Internals

> Independent analysis and reconstruction of architectural patterns found in modern AI coding agent systems — built for learning and research.

![License: Research](https://img.shields.io/badge/License-Research-blue.svg)
![Language: Python](https://img.shields.io/badge/Language-Python-3776AB.svg)
![Language: Rust](https://img.shields.io/badge/Language-Rust-orange.svg)
![Status: Active](https://img.shields.io/badge/Status-Active-brightgreen.svg)

---

## 📰 Background

In March 2026, a source map file published in an Anthropic package unintentionally exposed parts of the internal structure of their coding agent system.

This repository is **not** that source code.

Instead, it is an independent analysis and reconstruction of similar architectural patterns, created for **learning and research purposes**.

> **Why this matters:** Modern AI tools like coding assistants and autonomous agents rely on structured tool systems, iterative reasoning loops, and context-aware decision making. Understanding these patterns is more valuable than accessing any single codebase.

---

## ⚠️ Disclaimer

> - This project does **not** contain proprietary or leaked source code
> - All implementations and explanations are based on publicly observed patterns and independent analysis
> - The goal is to understand how modern AI agent systems are designed, not to replicate any private system

---

## 📚 Documentation Sections

| Folder | File | Description |
|--------|------|-------------|
| `docs/` | [ARCHITECTURE.md](docs/ARCHITECTURE.md) | System structure and component relationships |
| `docs/` | [EXECUTION_FLOW.md](docs/EXECUTION_FLOW.md) | Step-by-step runtime and CLI flow |
| `docs/` | [DECISION_ENGINE.md](docs/DECISION_ENGINE.md) | Matching, scoring, and execution/authorization decisions |
| `docs/` | [CONCEPTS.md](docs/CONCEPTS.md) | Core mental models and design theory |
| `analysis/` | [PARITY_ANALYSIS.md](analysis/PARITY_ANALYSIS.md) | Parity gaps and implementation status observations |
| `analysis/` | [WORKING_GUIDELINES.md](analysis/WORKING_GUIDELINES.md) | Practical working constraints and process notes |
| `references/` | [CODE_MAP.md](references/CODE_MAP.md) | High-signal files/functions for quick lookup |
| `references/` | [RUNTIME_SNIPPETS.md](references/RUNTIME_SNIPPETS.md) | Curated runtime decision snippets |
| `navigation/` | [LEARNING_PATH.md](navigation/LEARNING_PATH.md) | Guided reading sequence for contributors |

---

## 🗺️ Recommended Reading Order

1. 🏛️ [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — Start with system structure
2. ⚙️ [docs/EXECUTION_FLOW.md](docs/EXECUTION_FLOW.md) — Trace the runtime path
3. 🧠 [docs/DECISION_ENGINE.md](docs/DECISION_ENGINE.md) — Understand matching & authorization
4. 💡 [docs/CONCEPTS.md](docs/CONCEPTS.md) — Deepen with mental models
5. 🔍 [analysis/PARITY_ANALYSIS.md](analysis/PARITY_ANALYSIS.md) — Explore implementation gaps
6. 🗂️ [references/CODE_MAP.md](references/CODE_MAP.md) — Quick code lookup
7. 📋 [references/RUNTIME_SNIPPETS.md](references/RUNTIME_SNIPPETS.md) — Key decision snippets
8. 🧭 [navigation/LEARNING_PATH.md](navigation/LEARNING_PATH.md) — Full guided path

> **New here?** See [navigation/LEARNING_PATH.md](navigation/LEARNING_PATH.md) for estimated reading times and a step-by-step guide.
# AI Agent Internals

This folder centralizes internal documentation for architecture, runtime flow, decision logic, and parity analysis of this repository.

## Sections

- docs/
  - ARCHITECTURE.md: system structure and component relationships
  - EXECUTION_FLOW.md: step-by-step runtime and CLI flow
  - DECISION_ENGINE.md: matching, scoring, and execution/authorization decisions
  - CONCEPTS.md: core mental models and design theory

- analysis/
  - PARITY_ANALYSIS.md: parity gaps and implementation status observations
  - WORKING_GUIDELINES.md: practical working constraints and process notes

- references/
  - CODE_MAP.md: high-signal files/functions for quick lookup
  - RUNTIME_SNIPPETS.md: curated runtime decision snippets

- navigation/
  - LEARNING_PATH.md: guided reading sequence for contributors

## Recommended Reading Order

1. docs/ARCHITECTURE.md
2. docs/EXECUTION_FLOW.md
3. docs/DECISION_ENGINE.md
4. docs/CONCEPTS.md
5. analysis/PARITY_ANALYSIS.md
6. references/CODE_MAP.md
7. references/RUNTIME_SNIPPETS.md
8. navigation/LEARNING_PATH.md

## 📰 Background

In March 2026, a source map file published in an Anthropic package unintentionally exposed parts of the internal structure of their coding agent system.

This repository is **not** that source code.

Instead, it is an independent analysis and reconstruction of similar architectural patterns, created for learning and research purposes.

## ⚠️ Disclaimer

- This project does **not** contain proprietary or leaked source code  
- All implementations and explanations are based on publicly observed patterns and independent analysis  
- The goal is to understand how modern AI agent systems are designed, not to replicate any private system

## 🎯 Why this matters

Modern AI tools like coding assistants and autonomous agents rely on:

- structured tool systems  
- iterative reasoning loops  
- context-aware decision making  

Understanding these patterns is more valuable than accessing any single codebase.
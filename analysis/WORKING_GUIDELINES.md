# CLAW.md

This file provides guidance to Claw Code when working with code in this repository.

## Detected stack
- Languages: Rust.
- Frameworks: none detected from the supported starter markers.

## Verification
- Run Rust verification from `rust/`: `cargo fmt`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`
- `src/` and `tests/` are both present; update both surfaces together when behavior changes.

## Repository shape
- `rust/` contains the Rust workspace and active CLI/runtime implementation.
- `src/` contains source files that should stay consistent with generated guidance and tests.
- `tests/` contains validation surfaces that should be reviewed alongside code changes.

## Working agreement
- Prefer small, reviewable changes and keep generated bootstrap files aligned with actual repo workflows.
- Keep shared defaults in `.claw.json`; reserve `.claw/settings.local.json` for machine-local overrides.
- Do not overwrite existing `CLAW.md` content automatically; update it intentionally when repo workflows change.

## Key Takeaways

1. **Three surfaces to keep in sync:** When behavior changes, update Rust code, Python mirror, and tests together—they are interdependent.

2. **Cargo is the primary build tool:** Run `cargo fmt`, `clippy`, and `test` from the rust/ folder. No npm or Python package builds needed for core logic.

3. **Bootstrap files are generated:** `.claw.json` and `.claw/settings.local.json` coordinate configuration. Don't manually edit bootstrap files unless updating workflows.

4. **Intent matters:** CLAW.md documents process agreements, not just current state. Update it when workflows change, not automatically.

---

**Related:** See PARITY_ANALYSIS.md for TS-to-Rust gap audit.

**Back:** [Index](../README.md)

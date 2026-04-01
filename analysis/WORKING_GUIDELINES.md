# CLAW.md (WORKSPACE EVIDENCE)

Scope: examples below are copied from files in this codespace only.

## detected stack

### Example: Rust workspace root

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.package]
version = "0.1.0"
edition = "2021"
```

### Example: repository detection marks Rust and other surfaces

```rust
RepoDetection {
	rust_workspace: cwd.join("rust").join("Cargo.toml").is_file(),
	rust_root: cwd.join("Cargo.toml").is_file(),
	python: cwd.join("pyproject.toml").is_file()
		|| cwd.join("requirements.txt").is_file()
		|| cwd.join("setup.py").is_file(),
	src_dir: cwd.join("src").is_dir(),
	tests_dir: cwd.join("tests").is_dir(),
	rust_dir: cwd.join("rust").is_dir(),
	package_json: cwd.join("package.json").is_file(),
}
```

---

## verification

### Example: contributor verification commands

```md
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo check --workspace
cargo test --workspace
```

### Example: workspace quick-start commands

```bash
cd rust/
cargo build --release
```

---

## repository shape

### Example: crates present in this workspace

```text
rust/crates/
  api/
  claw-cli/
  commands/
  compat-harness/
  plugins/
  runtime/
  tools/
```

### Example: source and test surfaces are present

```text
src/
tests/test_porting_workspace.py
```

---

## working agreement

### Example: generated init guidance includes the same policy text

```rust
lines.push("- Keep shared defaults in `.claw.json`; reserve `.claw/settings.local.json` for machine-local overrides.".to_string());
lines.push("- Do not overwrite existing `CLAW.md` content automatically; update it intentionally when repo workflows change.".to_string());
```

### Example: init writes baseline config and gitignore entries

```rust
const STARTER_CLAW_JSON: &str = concat!(
	"{\n",
	"  \"permissions\": {\n",
	"    \"defaultMode\": \"dontAsk\"\n",
	"  }\n",
	"}\n",
);

const GITIGNORE_ENTRIES: [&str; 2] = [".claw/settings.local.json", ".claw/sessions/"];
```

### Example: init does not overwrite existing files

```rust
fn write_file_if_missing(path: &Path, content: &str) -> Result<InitStatus, std::io::Error> {
	if path.exists() {
		return Ok(InitStatus::Skipped);
	}
	fs::write(path, content)?;
	Ok(InitStatus::Created)
}
```

---

## Key Takeaways

1. The Rust workspace, command flow, and verification workflow are directly represented in local files.
2. Guidance text in this repo is generated from real code in the init path, not separate manual docs only.
3. `src/`, `tests/`, and `rust/` are all discoverable surfaces in the repository and in detection logic.

---

**Related:** [PARITY_ANALYSIS.md](./PARITY_ANALYSIS.md)

**Back:** [Index](../README.md)

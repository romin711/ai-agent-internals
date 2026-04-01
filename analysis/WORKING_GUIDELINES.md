# 📋 WORKING GUIDELINES (WORKSPACE EVIDENCE)

> All examples below are copied from files in this codespace only.

---

## 🔍 Detected Stack

### Rust Workspace Root

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.package]
version = "0.1.0"
edition = "2021"
```

### Repository Detection Marks Rust and Other Surfaces

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

## ✅ Verification

### Contributor Verification Commands

```md
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo check --workspace
cargo test --workspace
```

### Workspace Quick-Start Commands

```bash
cd rust/
cargo build --release
```

---

## 📁 Repository Shape

### Crates Present in This Workspace

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

### Source and Test Surfaces Are Present

```text
src/
tests/test_porting_workspace.py
```

---

## 🤝 Working Agreement

### Generated Init Guidance Includes Policy Text

```rust
lines.push("- Keep shared defaults in `.claw.json`; reserve `.claw/settings.local.json` for machine-local overrides.".to_string());
lines.push("- Do not overwrite existing `CLAW.md` content automatically; update it intentionally when repo workflows change.".to_string());
```

### Init Writes Baseline Config and Gitignore Entries

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

### Init Does Not Overwrite Existing Files

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

## 📌 Key Takeaways

1. **Local files are the ground truth** — The Rust workspace, command flow, and verification workflow are directly represented in local files.
2. **Init code generates guidance** — Guidance text in this repo is generated from real code in the init path, not separate manual docs only.
3. **All surfaces are discoverable** — `src/`, `tests/`, and `rust/` are all discoverable surfaces in the repository and in detection logic.

---

**Related:** [PARITY_ANALYSIS.md](./PARITY_ANALYSIS.md)

**↑ Back:** [Index](../README.md)

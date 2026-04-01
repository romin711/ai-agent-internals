# PARITY GAP ANALYSIS (WORKSPACE EVIDENCE)

Scope: examples below are taken only from files in this codespace.

Method: each section uses direct snippets from local Rust crates so every claim is traceable to code.

## tools/

### Example: built-in tool registry

```rust
name: "bash",
name: "read_file",
name: "write_file",
name: "edit_file",
name: "glob_search",
name: "grep_search",
name: "WebFetch",
name: "WebSearch",
name: "TodoWrite",
name: "Skill",
name: "Agent",
name: "ToolSearch",
name: "NotebookEdit",
name: "Sleep",
name: "SendUserMessage",
name: "Config",
name: "StructuredOutput",
name: "REPL",
```

### Example: skill tool definition

```rust
ToolSpec {
  name: "Skill",
  description: "Load a local skill definition and its instructions.",
  input_schema: json!({
    "type": "object",
    "properties": {
      "skill": { "type": "string" },
      "args": { "type": "string" }
    },
    "required": ["skill"],
    "additionalProperties": false
  }),
  required_permission: PermissionMode::ReadOnly,
}
```

---

## hooks/

### Example: hook events and runner

```rust
pub enum HookEvent {
  PreToolUse,
  PostToolUse,
}

pub fn run_pre_tool_use(&self, tool_name: &str, tool_input: &str) -> HookRunResult
pub fn run_post_tool_use(
  &self,
  tool_name: &str,
  tool_input: &str,
  tool_output: &str,
  is_error: bool,
) -> HookRunResult
```

### Example: hook execution is wired into the conversation loop

```rust
let pre_hook_result = self.hook_runner.run_pre_tool_use(&tool_name, &input);
if pre_hook_result.is_denied() {
  let deny_message = format!("PreToolUse hook denied tool `{tool_name}`");
  ConversationMessage::tool_result(
    tool_use_id,
    tool_name,
    format_hook_message(&pre_hook_result, &deny_message),
    true,
  )
} else {
  let (mut output, mut is_error) = match self.tool_executor.execute(&tool_name, &input) {
    Ok(output) => (output, false),
    Err(error) => (error.to_string(), true),
  };

  let post_hook_result = self
    .hook_runner
    .run_post_tool_use(&tool_name, &input, &output, is_error);
  if post_hook_result.is_denied() {
    is_error = true;
  }
  ConversationMessage::tool_result(
    tool_use_id,
    tool_name,
    output,
    is_error,
  )
}
```

### Example: hooks parsed from config

```rust
Ok(RuntimeHookConfig {
  pre_tool_use: optional_string_array(hooks, "PreToolUse", "merged settings.hooks")?
    .unwrap_or_default(),
  post_tool_use: optional_string_array(hooks, "PostToolUse", "merged settings.hooks")?
    .unwrap_or_default(),
})
```

---

## plugins/

### Example: plugin subsystem exists

```rust
pub enum PluginKind {
  Builtin,
  Bundled,
  External,
}

pub struct PluginManifest {
  pub name: String,
  pub version: String,
  pub description: String,
  pub permissions: Vec<PluginPermission>,
  pub default_enabled: bool,
  pub hooks: PluginHooks,
  pub lifecycle: PluginLifecycle,
  pub tools: Vec<PluginToolManifest>,
  pub commands: Vec<PluginCommandManifest>,
}
```

### Example: plugin manager operations

```rust
pub fn list_installed_plugins(&self) -> Result<Vec<PluginSummary>, PluginError>
pub fn aggregated_hooks(&self) -> Result<PluginHooks, PluginError>
pub fn aggregated_tools(&self) -> Result<Vec<PluginTool>, PluginError>
pub fn install(&mut self, source: &str) -> Result<InstallOutcome, PluginError>
pub fn enable(&mut self, plugin_id: &str) -> Result<(), PluginError>
pub fn disable(&mut self, plugin_id: &str) -> Result<(), PluginError>
pub fn uninstall(&mut self, plugin_id: &str) -> Result<(), PluginError>
pub fn update(&mut self, plugin_id: &str) -> Result<UpdateOutcome, PluginError>
```

### Example: plugin slash command and aliases

```rust
SlashCommandSpec {
  name: "plugin",
  aliases: &["plugins", "marketplace"],
  summary: "Manage Claw Code plugins",
  argument_hint: Some(
    "[list|install <path>|enable <name>|disable <name>|uninstall <id>|update <id>]",
  ),
  resume_supported: false,
}
```

---

## skills/ and CLAW.md discovery

### Example: `/skills` command and source roots

```rust
pub fn handle_skills_slash_command(args: Option<&str>, cwd: &Path) -> std::io::Result<String> {
  match normalize_optional_args(args) {
    None | Some("list") => {
      let roots = discover_skill_roots(cwd);
      let skills = load_skills_from_roots(&roots)?;
      Ok(render_skills_report(&skills))
    }
    Some("-h" | "--help" | "help") => Ok(render_skills_usage(None)),
    Some(args) => Ok(render_skills_usage(Some(args))),
  }
}

"  Sources          .codex/skills, .claw/skills, legacy /commands"
```

### Example: CLAW instruction file discovery

```rust
for candidate in [
  dir.join("CLAW.md"),
  dir.join("CLAW.local.md"),
  dir.join(".claw").join("CLAW.md"),
  dir.join(".claw").join("instructions.md"),
] {
  push_context_file(&mut files, candidate)?;
}
```

---

## cli/

### Example: slash command coverage in current Rust code

```rust
const SLASH_COMMAND_SPECS: &[SlashCommandSpec] = &[
  SlashCommandSpec {
    name: "help",
    aliases: &[],
    summary: "Show available slash commands",
    argument_hint: None,
    resume_supported: true,
  },
  SlashCommandSpec {
    name: "skills",
    aliases: &[],
    summary: "List available skills",
    argument_hint: None,
    resume_supported: true,
  },
  SlashCommandSpec {
    name: "plugin",
    aliases: &["plugins", "marketplace"],
    summary: "Manage Claw Code plugins",
    argument_hint: Some(
      "[list|install <path>|enable <name>|disable <name>|uninstall <id>|update <id>]",
    ),
    resume_supported: false,
  },
];
```

### Example: prompt mode starts with tools enabled

```rust
CliAction::Prompt {
  prompt,
  model,
  output_format,
  allowed_tools,
  permission_mode,
} => LiveCli::new(model, true, allowed_tools, permission_mode)?
  .run_turn_with_output(&prompt, output_format)?,
```

---

## assistant/ (agentic loop, streaming, tool calling)

### Example: loop limit and iteration check

```rust
Self {
  session,
  api_client,
  tool_executor,
  permission_policy,
  system_prompt,
  max_iterations: usize::MAX,
  usage_tracker,
  hook_runner: HookRunner::from_feature_config(&feature_config),
}

iterations += 1;
if iterations > self.max_iterations {
  return Err(RuntimeError::new(
    "conversation loop exceeded the maximum number of iterations",
  ));
}
```

### Example: tool uses are extracted from assistant blocks

```rust
let pending_tool_uses = assistant_message
  .blocks
  .iter()
  .filter_map(|block| match block {
    ContentBlock::ToolUse { id, name, input } => {
      Some((id.clone(), name.clone(), input.clone()))
    }
    _ => None,
  })
  .collect::<Vec<_>>();
```

---

## services/ (API client, auth, models, MCP)

### Example: API surface exports

```rust
pub use providers::claw_provider::{ClawApiClient, ClawApiClient as ApiClient, AuthSource};
pub use providers::openai_compat::{OpenAiCompatClient, OpenAiCompatConfig};
pub use providers::{
  detect_provider_kind, max_tokens_for_model, resolve_model_alias, ProviderKind,
};
```

### Example: OAuth request models

```rust
pub struct OAuthAuthorizationRequest {
  pub authorize_url: String,
  pub client_id: String,
  pub redirect_uri: String,
  pub scopes: Vec<String>,
  pub state: String,
  pub code_challenge: String,
  pub code_challenge_method: PkceChallengeMethod,
  pub extra_params: BTreeMap<String, String>,
}
```

### Example: MCP bootstrap transport mapping

```rust
pub enum McpClientTransport {
  Stdio(McpStdioTransport),
  Sse(McpRemoteTransport),
  Http(McpRemoteTransport),
  WebSocket(McpRemoteTransport),
  Sdk(McpSdkTransport),
  ManagedProxy(McpManagedProxyTransport),
}
```

### Example: remote/upstream proxy state

```rust
pub struct UpstreamProxyBootstrap {
  pub remote: RemoteSessionContext,
  pub upstream_proxy_enabled: bool,
  pub token_path: PathBuf,
  pub ca_bundle_path: PathBuf,
  pub system_ca_path: PathBuf,
  pub token: Option<String>,
}
```

---

## current defaults and behavior examples

### Example: default permission mode

```rust
fn default_permission_mode() -> PermissionMode {
  env::var("CLAW_PERMISSION_MODE")
    .ok()
    .as_deref()
    .and_then(normalize_permission_mode)
    .map_or(PermissionMode::DangerFullAccess, permission_mode_from_label)
}
```

### Example: clap default permission mode

```rust
#[arg(long, value_enum, default_value_t = PermissionMode::DangerFullAccess)]
pub permission_mode: PermissionMode,
```

### Example: init template writes `dontAsk`

```rust
const STARTER_CLAW_JSON: &str = concat!(
  "{\n",
  "  \"permissions\": {\n",
  "    \"defaultMode\": \"dontAsk\"\n",
  "  }\n",
  "}\n",
);
```

---

## Key Takeaways

1. The current Rust workspace includes concrete implementations for tools, hooks, plugins, skills, CLI commands, and service modules.
2. The strongest evidence is in executable code paths (command parsing, runtime loop, plugin manager, and hook runner), not only docs.
3. This document now uses only local code examples from this codespace.

---

**Related:** [WORKING_GUIDELINES.md](./WORKING_GUIDELINES.md)

**Back:** [Index](../README.md)

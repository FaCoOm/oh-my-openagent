# OpenCode Harness Configuration Guide & Synthesis

This document provides a comprehensive mapping of all documentation files covering the configuration for the OpenCode harness (`omo-opencode`), followed by an exhaustive synthesis of all applicable configuration categories, fields, behaviors, and runtime options.

---

## 1. Documentation Map: Files Covering OpenCode Harness Configuration

The documentation in `docs/` is organized into reference specifications, user guides, examples, and troubleshooting notes. Below are the authoritative documentation files that govern OpenCode configuration:

### Core Configuration References
- [configuration.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/reference/configuration.md)
  - **Role:** The primary, comprehensive configuration reference for Oh My OpenCode.
  - **Scope:** Details the unified `omo.jsonc` file layout, the `[opencode]` block, agent overrides, category delegation, background execution, features (tmux, skills, memory, comment-checker, git-master, notification, MCPs, LSP), runtime fallback, model capabilities cache, hashline edit, experimental switches, and environment variables.
- [opencode-config.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/reference/opencode-config.md)
  - **Role:** Dedicated OpenCode edition (legacy/exclusive) reference.
  - **Scope:** Documents keys and hooks read strictly by the OpenCode plugin: the 11-agent core roster (`sisyphus`, `hephaestus`, `prometheus`, `oracle`, `librarian`, `explore`, `multimodal-looker`, `metis`, `momus`, `atlas`, `sisyphus-junior`), `agent_order`, `disabled_agents`, `sisyphus_agent` orchestration controls, `sisyphus.tasks` file persistence, OpenCode-specific hooks (`prometheus-md-only`, `no-sisyphus-gpt`, `no-hephaestus-non-gpt`, `atlas`, `sisyphus-junior-notepad`), and the `/ulw-execute` hook.
- [omo-json.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/reference/omo-json.md)
  - **Role:** Harness-spanning configuration architecture specification.
  - **Scope:** Defines how `omo.jsonc` merges across layers (User `~/.omo/omo.jsonc` vs. Project `.omo/omo.jsonc`), how `[opencode]` is treated as a freeform plugin record, how shared keys (`categories`, `agents`, `models`, `disabled_skills`, `git_master`) behave, profile activation (`OMO_PROFILE`, `OCX_PROFILE`), and the migration engine from legacy files.

### Feature & Functionality References
- [features.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/reference/features.md)
  - **Role:** Exhaustive feature-by-feature reference and architecture snapshot.
  - **Scope:** Details default agent model chains, tool restrictions, background subagents (`task()`, `background_output()`), visual tmux multi-agent, 3-tier MCP system, hook tiers, tool gating, and OpenClaw daemon integration.
- [monitor.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/reference/monitor.md)
  - **Role:** Dedicated reference for the OpenCode-only `monitor` feature.
  - **Scope:** Configuration fields under `[opencode].monitor`, security allowlists (`allowed_commands`), tools (`monitor_start`, `monitor_stop`, `monitor_list`, `monitor_output`), buffering parameters, and untrusted output envelope structure.

### Practical Guides & Concepts
- [agent-model-matching.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/guide/agent-model-matching.md)
  - **Role:** Guide for choosing models and understanding prompt optimization.
  - **Scope:** Model profile lanes (`daily-normal`, `daily-heavy`, `geeky-normal`, `geeky-heavy`, `recommended`), model families, tuned prompt presets per model family, and prompting strategy differences between Claude (mechanics-driven) and GPT (principle-driven).
- [team-mode.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/guide/team-mode.md)
  - **Role:** User guide for parallel multi-agent Team Mode.
  - **Scope:** 11 configuration fields under `[opencode].team_mode`, team definition schema (`~/.omo/teams/{name}/config.json`), member types (`category` vs `subagent_type`), member eligibility, the 12 `team_*` tools, and lifecycle.
- [btw.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/guide/btw.md)
  - **Role:** User guide for BTW / `/side` retained side conversations.
  - **Scope:** Explains message boundary isolation, TUI picker (`Ctrl+/`), prompt status reporting, context capping (64 messages / 64 KiB), and lifecycle controls.
- [orchestration.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/guide/orchestration.md)
  - **Role:** Orchestration design guide.
  - **Scope:** Explains how Sisyphus, Prometheus, Atlas, and worker agents coordinate execution, goal setting, and work tracking.
- [installation.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/guide/installation.md) & [migrating-from-opencode.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/guide/migrating-from-opencode.md)
  - **Role:** Installation and legacy migration guide.
  - **Scope:** Guided setup with `bunx oh-my-openagent install`, config file paths, and migration mechanics.

### Configuration Blueprints & Templates
- [docs/examples/default.jsonc](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/examples/default.jsonc) — Balanced baseline configuration.
- [docs/examples/coding-focused.jsonc](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/examples/coding-focused.jsonc) — Optimized for high-throughput coding tasks.
- [docs/examples/planning-focused.jsonc](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/examples/planning-focused.jsonc) — Optimized for architectural analysis and rigorous planning.
- [docs/templates/AGENTS.md.example](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/templates/AGENTS.md.example) — Template for project-level instructions and rules.

### Operational & Diagnostics References
- [cli.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/reference/cli.md) — CLI commands including `oh-my-openagent config migrate`, `doctor`, `refresh-model-capabilities`.
- [ollama.md](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/docs/troubleshooting/ollama.md) — Provider-specific instructions for local Ollama usage (disabling streaming to prevent JSON parse truncation).

---

## 2. Complete Synthesis of OpenCode Harness Configuration

Configuration for the OpenCode harness is defined in JSONC (`omo.jsonc` or `omo.json`). OpenCode-specific settings reside inside the `[opencode]` block, while shared base keys configure common abstractions across all harnesses.

```
~/.omo/omo.jsonc (User Layer)  <-- lowest precedence
      ▲
<project>/.omo/omo.jsonc (Project Layer) <-- nearest project file wins
      ▲
profiles.<name>.[opencode] (Active Profile Layer) <-- highest precedence
```

### 2.1 File Locations & Layer Precedence
1. **User Layer:** `~/.omo/omo.jsonc` (or fallback `~/.omo/omo.json`).
2. **Project Layer:** `.omo/omo.jsonc` walked from current working directory upward to `$HOME`.
3. **Resolution Order:** Base shared keys -> `[opencode]` block -> `profiles.<name>` -> `profiles.<name>.[opencode]`.
4. **Profile Activation:** Determined by `OMO_PROFILE`, `OCX_PROFILE`, or lexical tail of `OPENCODE_CONFIG_DIR` (`profiles/<name>`).
5. **Schema Pointer:**
   ```json
   {
     "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/assets/omo.schema.json"
   }
   ```

---

### 2.2 Models, Catalogs & Runtime Fallbacks

#### 2.2.1 Shared Model Catalog (`models`)
Maps aliases to concrete configurations:
```jsonc
{
  "models": {
    "primary-coding": { "model": "anthropic/claude-opus-5-5", "reasoning": "medium" },
    "fast-helper": { "model": "openai/gpt-6-luna-fast", "reasoning": "low" }
  }
}
```

#### 2.2.2 Model Normalization
Model parameters are automatically normalized against capabilities data:
- `reasoning`: Canonical setting (e.g. `off`, `low`, `medium`, `high`, `xhigh`, `max`). Deprecated `variant` and `reasoningEffort` are automatically migrated.
- `temperature` & `top_p`: Stripped if unsupported by target model.
- `maxTokens` / `max_tokens`: Capped to model limits.
- `providerOptions`: Dedicated provider overrides (e.g., Anthropic thinking budgets).

#### 2.2.3 Runtime Fallback (`runtime_fallback`)
Auto-recovers from API failures (rate limits, context limits, outage HTTP codes):
```jsonc
{
  "[opencode]": {
    "runtime_fallback": {
      "enabled": true,
      "retry_on_errors": [429, 500, 502, 503, 504],
      "max_fallback_attempts": 3,
      "cooldown_seconds": 60,
      "timeout_seconds": 30, // Set to 0 to disable timeout escalation
      "notify_on_fallback": true,
      "restore_primary_after_cooldown": false
    }
  }
}
```

#### 2.2.4 Model Capabilities Cache (`model_capabilities`)
Controls cached capability data from models.dev:
```jsonc
{
  "[opencode]": {
    "model_capabilities": {
      "enabled": true,
      "auto_refresh_on_start": true,
      "refresh_timeout_ms": 5000,
      "source_url": "https://models.dev/api.json"
    }
  }
}
```

---

### 2.3 Agent Configuration (`agents`)

#### 2.3.1 Built-in Agent Roster
- **Curated Read-Only:** `explore`, `librarian`, `plan-consultant`, `plan-reviewer`.
- **OpenCode Core Suite:** `sisyphus`, `hephaestus`, `prometheus`, `oracle`, `librarian`, `explore`, `multimodal-looker`, `metis`, `momus`, `atlas`, `sisyphus-junior`.

#### 2.3.2 Agent Options Table
| Option | Type | Description |
| :--- | :--- | :--- |
| `model` | string | Concrete model identifier (`provider/model`) or catalog reference. |
| `models` | array | Ordered model candidate list. |
| `fallback_models` | string / array | Ordered fallback model strings or object definitions. |
| `reasoning` | string | Canonical reasoning level (`off`, `low`, `medium`, `high`, `xhigh`, `max`). |
| `temperature` | number | Sampling temperature. |
| `top_p` | number | Nucleus sampling probability. |
| `prompt` | string | Complete prompt replacement. Supports `file://` URIs. |
| `prompt_append` | string | Prompt addendum. Supports `file://` URIs. |
| `tools` | record<string, boolean> | Granular tool enable/disable map. |
| `disable` | boolean | Disables the agent definition. |
| `permission` | object | Per-tool permissions: `edit`, `bash`, `webfetch`, `task`, `doom_loop`, `external_directory` (`ask`/`allow`/`deny`). |
| `mode` | string | `subagent`, `primary`, or `all`. |
| `color` | string | Hex UI color (`#RRGGBB`). |
| `displayName` | string | Label shown in UI. |
| `category` | string | Inherit model configuration from a category. |
| `skills` | string[] | Skill names injected into system prompt. |
| `providerOptions` | object | Provider-specific request options. |
| `ultrawork` | object | Overrides for ultrawork mode sessions. |
| `compaction` | object | Model overrides applied during context compaction. |

#### 2.3.3 Special Behaviors & Constraints
- **Prometheus Prompt Rule:** Prometheus always preserves its mandatory planner base prompt; both `prompt` and `prompt_append` are appended to it.
- **`agent_order`:** Customizes TUI tab cycling order (default: `sisyphus`, `hephaestus`, `prometheus`, `atlas`).
- **`disabled_agents`:** Array of agent names completely suppressed from OpenCode.

---

### 2.4 Task Delegation & Categories (`categories`)

Delegated tasks routed via `task(category: "...")` resolve to predefined domain categories:
- **Built-ins:** `quick`, `unspecified-low`, `unspecified-high`, `writing`, `visual-engineering`, `architect`, `deep-low`, `deep-high`, `artistry`, `ultrabrain`.
- **Category Options:**
  - `model`, `models`, `fallback_models`, `reasoning`, `temperature`, `top_p`, `max_tokens` / `maxTokens`.
  - `provider_options` / `providerOptions`, `prompt_append` (supports `file://`), `tools`.
  - `is_unstable_agent`: Forces background mode and monitoring (auto-activated for gemini and minimax models).
  - `disable`: Disables category from delegation routing.

---

### 2.5 Task & Concurrency System (`background_task`)

Controls subagent execution limits:
```jsonc
{
  "[opencode]": {
    "background_task": {
      "defaultConcurrency": 5,
      "providerConcurrency": { "anthropic": 3, "openai": 5 },
      "modelConcurrency": { "anthropic/claude-opus-5-5": 2 },
      "maxDepth": 3,
      "staleTimeoutMs": 2700000,
      "messageStalenessTimeoutMs": 3600000,
      "taskTtlMs": 1800000,
      "sessionGoneTimeoutMs": 60000,
      "taskCleanupDelayMs": 600000,
      "syncPollTimeoutMs": 60000,
      "maxToolCalls": 4000,
      "circuitBreaker": {
        "enabled": true,
        "maxToolCalls": 50,
        "consecutiveThreshold": 5
      }
    }
  }
}
```

---

### 2.6 Orchestration & Sisyphus Settings

- **`sisyphus_agent`:**
  - `disabled`: Disables Sisyphus orchestration.
  - `tdd`: TDD execution mode.
  - `default_builder_enabled`: Enables OpenCode-Builder agent.
  - `planner_enabled`: Enables Prometheus planner.
  - `replace_plan`: Replaces default plan agent.
- **`experimental.task_system`:** Intercepts `TodoWrite`/`TodoRead` tools and replaces them with file-persisted task tools (`task_create`, `task_get`, `task_list`, `task_update`).
- **`sisyphus.tasks`:**
  - `storage_path`: Custom path for task lists.
  - `task_list_id`: Explicit task list ID.
  - `claude_code_compat`: Claude Code compatibility mode.

---

### 2.7 Specialized Functionalities

#### 2.7.1 Team Mode (`team_mode`)
Multi-agent parallel collaboration (OFF by default):
```jsonc
{
  "[opencode]": {
    "team_mode": {
      "enabled": true,
      "max_parallel_members": 4,
      "max_members": 8,
      "tmux_visualization": false,
      "max_messages_per_run": 10000,
      "max_wall_clock_minutes": 120,
      "max_member_turns": 500,
      "base_dir": "~/.omo",
      "message_payload_max_bytes": 32768,
      "recipient_unread_max_bytes": 262144,
      "mailbox_poll_interval_ms": 3000
    }
  }
}
```
Enables 12 `team_*` tools and reads specs from `.omo/teams/{name}/config.json`.

#### 2.7.2 Monitor (`monitor`)
OpenCode-only non-interactive background process observation:
```jsonc
{
  "[opencode]": {
    "monitor": {
      "enabled": true,
      "live_mode_enabled": false,
      "allowed_commands": ["bun", "npm", "cargo", "tail"],
      "max_monitors_per_session": 3,
      "max_runtime_ms": 1800000,
      "batch_max_lines": 50,
      "batch_max_bytes": 16384,
      "flush_interval_ms": 1000,
      "ring_max_lines": 1000,
      "line_max_bytes": 8192,
      "pattern_max_length": 512
    }
  }
}
```
Exposes `monitor_start`, `monitor_stop`, `monitor_list`, and `monitor_output`.

#### 2.7.3 Tmux Integration (`tmux`)
```jsonc
{
  "[opencode]": {
    "tmux": {
      "enabled": true,
      "layout": "main-vertical", // main-vertical, main-horizontal, tiled, even-horizontal, even-vertical
      "main_pane_size": 60,
      "main_pane_min_width": 120,
      "agent_pane_min_width": 40,
      "isolation": "inline" // inline, window, session
    }
  }
}
```

#### 2.7.4 Browser Automation (`browser_automation_engine`)
- `provider`: `"playwright"` (MCP, default), `"dev-browser"` (skill), or `"playwright-cli"` (CLI).
- Security rule: `playwright_mcp_args` is strictly restricted to user-level configuration.

#### 2.7.5 Git Master (`git_master`)
- `commit_footer`: Boolean or custom text string opting into commit footers (no `Co-authored-by` trailer is ever emitted).
- `git_env_prefix`: Shell prefix for git executions (default `"GIT_MASTER=1"`).

#### 2.7.6 Comment Checker (`comment_checker`)
- `custom_prompt`: Custom instructions for code comment linting. Use `{{comments}}` placeholder.

#### 2.7.7 Hashline Edit (`hashline_edit`)
- `hashline_edit: true`: Replaces the standard edit tool with hash-anchored `LINE#ID` edits and activates `hashline-read-enhancer`.

#### 2.7.8 Memory System (`memory`)
Full git-backed memory repository:
- `enabled`: Master switch.
- `reflection`: Periodic conversation review (`trigger.step_count: 25`, `trigger.on_compaction: true`).
- `nudge`: Prompt nudging for unsaved insights (`every_user_turns: 10`).
- `recall`: Kibitzer sidecar evaluating background facts and surfacing hints.
- `facts`: Background extraction on settled turns.
- `dream`: Idle consolidation passes.
- `sync` / `search`: Git remote synchronization and semantic search.

#### 2.7.9 MCPs & LSP
- Built-in MCPs: `websearch`, `context7`, `grep_app`, `lsp`.
- Disabling: `disabled_mcps: ["websearch", "lsp"]`.
- Custom LSP mapping: Defined via `.opencode/lsp.json` or `.omo/lsp.json`.

---

### 2.8 Lifecycle Hooks & Commands Control

#### 2.8.1 `disabled_hooks`
Allows disabling any of the 50+ lifecycle hooks. Hooks exclusive to OpenCode:
- `prometheus-md-only`: Restricts Prometheus to plan markdown files.
- `no-sisyphus-gpt`: Guards against running Sisyphus on incompatible GPT versions.
- `no-hephaestus-non-gpt`: Restricts Hephaestus to GPT models.
- `atlas`: Continuation-tier boulder orchestrator.
- `sisyphus-junior-notepad`: Manages notepad states for junior agents.

#### 2.8.2 `disabled_commands`
Disables built-in slash commands: `goal`, `refactor`, `ulw-execute`, `stop-continuation`, `remove-ai-slops`, `hyperplan`.

#### 2.8.3 `disabled_skills`
Cross-harness skill suppression array (e.g. `["frontend", "visual-qa"]`).

---

### 2.9 Experimental Flags (`experimental`)

```jsonc
{
  "[opencode]": {
    "experimental": {
      "task_system": true,
      "truncate_all_tool_outputs": false,
      "aggressive_truncation": false,
      "disable_omo_env": false,
      "preemptive_compaction": true,
      "plugin_load_timeout_ms": 10000,
      "safe_hook_creation": true,
      "model_fallback_title": false,
      "max_tools": 38,
      "disable_live_parent_wake_routing": false,
      "dynamic_context_pruning": {
        "enabled": false,
        "notification": "detailed", // off, minimal, detailed
        "turn_protection": { "enabled": true, "turns": 3 },
        "protected_tools": ["task", "todowrite", "todoread", "lsp_rename", "session_read"],
        "strategies": {
          "deduplication": { "enabled": true },
          "supersede_writes": { "enabled": true, "aggressive": false },
          "purge_errors": { "enabled": true, "turns": 5 }
        }
      }
    }
  }
}
```

---

### 2.10 Telemetry & Environment Variables

- **Telemetry:**
  - `[opencode].telemetry`: Boolean (`true`/`false`) in OpenCode config.
  - Shared base uses `telemetry: { "enabled": false }`.
- **Key Environment Variables:**
  - `OPENCODE_CONFIG_DIR`: Path to OpenCode config root.
  - `OPENGATEWAY_API_KEY`: Injects the OpenGateway provider.
  - `OMO_DEBUG`: When set to `1`, prints component debug diagnostics to stderr.
  - `OMO_SEND_ANONYMOUS_TELEMETRY`: Set to `0`, `false`, or `no` to disable telemetry.
  - `LSP_TOOLS_MCP_INSTALL_DECISIONS`: Custom path for LSP installation approvals.

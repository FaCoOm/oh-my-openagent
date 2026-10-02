# OpenCode Harness: Agent Deprecation, Profile Configuration & Runtime Model Resolution Analysis

This document provides a definitive, source-verified analysis addressing the three architectural inquiries regarding Oh My OpenCode / Oh My OpenAgent (`omo`), followed by the corrective revision of `~/.omo/omo.jsonc`.

---

## 1. Are Core Agents (Sisyphus, Atlas, Prometheus, etc.) Deprecated?

### **No. They are NOT deprecated in the OpenCode harness (`omo-opencode`).**

### Context & Why Newer Docs Refer to "Legacy"
1. **Multi-Harness Architecture Refactor:**
   - The repository has been reorganized from a monolithic OpenCode plugin into a modular multi-harness Agent OS (`packages/AGENTS.md`).
   - The project supports three distinct runtime editions:
     1. **Ultimate Edition (`packages/omo-opencode`):** The OpenCode plugin.
     2. **Light Edition (`packages/omo-codex` / `lazycodex`):** Codex CLI plugin.
     3. **Native Edition (`packages/omo-native` / `packages/omo-senpi`):** Standalone agent engine powered by Senpi and the `senpi-task` DAG engine.
2. **Where the 11 Agents Live:**
   - The **11 specialist agents** (`sisyphus`, `hephaestus`, `prometheus`, `oracle`, `librarian`, `explore`, `multimodal-looker`, `metis`, `momus`, `atlas`, and `sisyphus-junior`) are the **first-class core roster of the OpenCode harness** (`packages/omo-opencode/src/agents/`).
   - In contrast, the newer **Native / Senpi harness** does not use this 11-agent suite; it executes through a single-agent loop coordinated with DAG background subagents (`task()`), referencing general cross-harness base abstractions (`explore`, `librarian`, `plan-consultant`, `plan-reviewer`).
3. **What "Legacy" Actually Refers To:**
   - **Legacy File Format:** Previous versions stored configuration in `~/.config/opencode/oh-my-openagent.json` or `oh-my-opencode.json`. The new system migrates all harnesses to a single, unified configuration file: `~/.omo/omo.jsonc`.
   - **Legacy Plugin Architecture vs. Multi-Harness OS:** When multi-harness documentation refers to "legacy OpenCode config" or "legacy architecture," it is referring to the OpenCode-exclusive agent/hook architecture as opposed to the new cross-harness abstractions shared between Codex and Senpi. Inside OpenCode, Sisyphus and its sibling agents remain the primary operating system.

---

## 2. Profiles: What They Are & How to Configure Them

### **Do you need a folder named `profiles` inside `.omo`?**
### **NO. You do NOT need a folder named `profiles` inside `.omo`.**

OMO supports two distinct profile concepts, neither of which requires creating a `profiles/` directory in `.omo`:

### A. Configuration Profiles (`profiles` in `omo.jsonc`)
A VSCode-style configuration overlay object defined **directly inside `omo.jsonc`**.

#### Configuration Structure:
```jsonc
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/assets/omo.schema.json",

  // Base settings apply everywhere
  "[opencode]": {
    "model_fallback": true
  },

  // In-file profile overlays:
  "profiles": {
    "deep-work": {
      "[opencode]": {
        "agents": {
          "sisyphus": {
            "model": "9router/cx/gpt-5.6-sol-high"
          }
        }
      }
    },
    "fast-iteration": {
      "[opencode]": {
        "agents": {
          "sisyphus": {
            "model": "9router/cx/gpt-5.6-luna-medium"
          }
        }
      }
    }
  }
}
```

#### Activation:
- Set via environment variables before launching:
  ```bash
  export OMO_PROFILE=deep-work
  # or
  export OCX_PROFILE=deep-work
  ```
- **Note on OpenCode's native profiles:** Upstream OpenCode has its own profile folder structure at `~/.config/opencode/profiles/<name>/`. If OpenCode sets `OPENCODE_CONFIG_DIR=.../profiles/my-profile`, OMO detects `<name>` automatically. However, inside `~/.omo/`, everything is maintained in the single `omo.jsonc` document under `"profiles"`.

### B. Model Profiles (`model_profile` and `model_profiles`)
- A model profile lane selection (e.g. `"daily-normal"`, `"daily-heavy"`, `"geeky-normal"`, `"geeky-heavy"`, `"recommended"`).
- **Scope:** Primarily consumed by the **Native / Senpi harness** (`packages/omo-senpi/src/components/model-profile/`) to adjust the main session model ladder.
- **OpenCode behavior:** The OpenCode plugin ignores `model_profile` and derives its agent models from `[opencode].agents` and `[opencode].categories`.

---

## 3. Why `[opencode]` Failed to Apply Configured Models in `omo.jsonc`

When comparing `C:\Users\Fate_Conqueror\.omo\omo.jsonc` against `omo-Legacy.jsonc`, the root causes of the runtime model failure were:

### 1. Provider Disconnect (Direct OpenAI vs. 9router Proxy)
- In OpenCode's primary configuration (`~/.config/opencode/opencode.json`), the user's active model providers are:
  - `9router` (connecting to OmniRoute at `https://omniroute.aidevs.me/v1`)
  - `omniroute`
  - `newapi`
- There is **no direct `openai` provider** configured with OpenAI API keys in `opencode.json`.
- In `omo-Legacy.jsonc`, all agents and categories were routed through `9router` (e.g. `9router/cx/gpt-5.6-sol-medium`, `9router/ds/deepseek-v4-flash`).
- In the previously modified `omo.jsonc`, everything was changed to direct OpenAI IDs (`openai/gpt-6-luna`, `openai/gpt-5.6-sol`, `openai/gpt-6-sol`).
- During startup, `applyAgentConfig` fetches available models via `fetchAvailableModels` from connected providers. Because `openai` was not connected, the model resolution engine (`resolveModelPipeline` in `model-core`) detected the requested `openai` models as **unavailable**, rejected the override, and defaulted back to OpenCode's system default model (`9router/codex/gpt-5.6-sol-medium`).

### 2. Leaking Top-Level `agents` and `categories`
- The previous `omo.jsonc` declared top-level `categories` and `agents` hardcoded with `openai/...`.
- In `loadOmoOpenCodeConfigChain` (`omo-config-chain.ts`), the loader extracts the base view, the harness block, and then runs `modelView(resolved.config)` which folds top-level records. Having mismatched top-level `openai` models leaked into the plugin resolution chain.

### 3. Concurrency Locking
- `background_task.providerConcurrency` was set strictly to `openai: 4` in `omo.jsonc`, whereas the active provider is `9router: 8` and `newapi: 8`.

---

## 4. Remediation Implemented

`C:\Users\Fate_Conqueror\.omo\omo.jsonc` has been revised to:
1. **Restore 9router / OmniRoute Model Mappings:** Restored exact, verified model IDs from `omo-Legacy.jsonc` across all 11 core agents:
   - `sisyphus`: `9router/cx/gpt-5.6-sol-medium` (fallbacks: `glm-5.2`, `deepseek-v4-flash`)
   - `hephaestus`: `9router/cx/gpt-5.6-sol-medium` (fallback: `deepseek-v4-flash`)
   - `oracle`: `9router/cx/gpt-5.6-sol-medium` (with explicit permission restrictions)
   - `librarian` & `explore`: `9router/cx/gpt-5.6-luna-medium` (with read-only permissions)
   - `multimodal-looker`: `9router/cx/gpt-5.6-terra-medium`
   - `prometheus`: `9router/cx/gpt-5.6-sol-medium` (with planning delegation instructions)
   - `metis`, `momus`, `atlas`: `9router/cx/gpt-5.6-...`
   - `sisyphus-junior`: `9router/ds/deepseek-v4-flash`
2. **Align All Delegation Categories:** Updated `visual-engineering`, `ultrabrain`, `deep`, `artistry`, `quick`, `unspecified-low`, `unspecified-high`, and `writing` to use `9router` models.
3. **Harmonize Base Layers:** Aligned top-level cross-harness base `categories` and `agents` with `9router` so multi-harness components do not crash on direct OpenAI.
4. **Restore Concurrency & Tmux Settings:** Reinstated `providerConcurrency` (`9router`: 8, `newapi`: 8, `k40`: 2), model concurrency limits, and tmux pane configuration.
5. **Backed Up Prior File:** Saved prior configuration to `~/.omo/omo.jsonc.bak.2026-09-28-before-fix`.

# OpenCode Model Architecture: Built-in OpenAI vs. Custom Providers Analysis

**Document Version:** 1.0.0  
**Date:** 2026-09-28  
**Scope:** Investigation into OpenCode model loading mechanics, `opencode.json` requirements, and agent configuration failure root causes for OpenAI models.

---

## 1. Executive Summary

- **Do OpenAI models need to be declared inside `opencode.json`?**  
  **NO.** Native providers (`openai`, `anthropic`, `google`, `github-copilot`) do **not** need model declarations in `~/.config/opencode/opencode.json`. Their models are automatically indexed and refreshed by OpenCode from `models.dev` into local caches (`~/.cache/opencode/models.json`).
- **Why do custom proxy providers (e.g. `9router`, `newapi`, `omniroute`) require `opencode.json`?**  
  Custom providers use `@ai-sdk/openai-compatible` pointing to private base URLs. OpenCode has no built-in schema or upstream catalog for arbitrary endpoints, so custom models must be explicitly declared under `provider.<name>.models`.
- **Why did OpenAI models fail to load onto specialist agents despite being usable in OpenCode?**  
  The failure was caused by a combination of:
  1. **Config sanitization dropping `"reasoning"`:** `omo-config-chain.ts` only whitelists `"reasoningEffort"`, silently discarding `"reasoning": "medium"`.
  2. **Initialization deadlock prevention (`client = undefined`):** `fetchAvailableModels` intentionally bypasses OpenCode's live client during plugin startup (avoiding issue #1301) and reads local cache files.
  3. **Model ID qualification mismatch:** Attempting proxy-style variant naming (e.g., `openai/gpt-5.6-sol-medium`) failed against OpenAI's official catalog (`openai/gpt-5.6-sol`).
  4. **Silent fallback to default session model:** When agent model resolution or validation fails, OpenCode silently falls back to the default session model (`9router`), giving the false impression that OpenAI was ignored.

---

## 2. OpenCode Provider Architecture: Native vs. Custom

OpenCode distinguishes between two fundamental provider categories:

### A. Built-in / Native Providers (`openai`, `anthropic`, `google`)
- **Authentication:** OAuth tokens stored in `~/.local/share/opencode/auth.json` or standard environment variables (`OPENAI_API_KEY`).
- **Model Discovery:** Maintained upstream via `models.dev` and synced directly into `~/.cache/opencode/models.json`.
- **`opencode.json` requirement:** **None.** Declaring native models in `opencode.json` is redundant and can cause schema conflicts.

### B. Custom / Compatible Providers (`9router`, `newapi`, `omniroute`)
- **Authentication:** Custom API keys and endpoints in `~/.config/opencode/opencode.json`.
- **Model Discovery:** Static declarations required under `provider.<id>.models` because OpenCode cannot introspect custom proxy endpoints.

---

## 3. Root Cause Analysis: Why Agent Loading Failed for OpenAI

### 3.1. Field Whitelist Stripping in `omo-config-chain.ts`
In [`packages/omo-opencode/src/plugin-config/omo-config-chain.ts`](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/packages/omo-opencode/src/plugin-config/omo-config-chain.ts#L66-L72):
```typescript
function modelInput(view: Readonly<Record<string, unknown>>): Record<string, unknown> {
  const agents = isPlainRecord(view.agents)
    ? Object.fromEntries(Object.entries(view.agents).flatMap(([name, definition]) => {
      const fields = recordFields(definition, [
        "description", "prompt", "model", "models", "variant",
        "reasoningEffort", "tools", "temperature", "disable"
      ])
      return fields === undefined ? [] : [[name, fields]]
    }))
    : undefined
```
- **The Bug / Limitation:** The whitelist contains `"reasoningEffort"`, NOT `"reasoning"`.
- When `omo.jsonc` defines `"reasoning": "medium"`, `recordFields` drops the property.

### 3.2. Lifecycle Deadlock Prevention (#1301)
In [`packages/omo-opencode/src/agents/builtin-agents.ts`](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/packages/omo-opencode/src/agents/builtin-agents.ts#L85-L90):
```typescript
// IMPORTANT: Do NOT call OpenCode client APIs during plugin initialization.
// This function is called from config handler, and calling client API causes deadlock.
// See: https://github.com/code-yeongyu/oh-my-openagent/issues/1301
const availableModels = await fetchAvailableModels(undefined, {
  connectedProviders: mergedConnectedProviders.length > 0 ? mergedConnectedProviders : undefined,
})
```
- The plugin cannot query the running OpenCode server (`client.model.list()`) during the `config` hook.
- It must rely entirely on:
  1. `~/.cache/oh-my-opencode/provider-models.json`
  2. `~/.cache/opencode/models.json`
- If cache files contain base model IDs without provider prefixes or if provider sets do not align with connected keys, model resolution falls back.

### 3.3. Model Identifier Discrepancy
- In `9router`, models are named monolithically, e.g.:
  `9router/codex/gpt-5.6-sol-medium`
- In native `openai`, the model ID in the cache is:
  `openai/gpt-5.6-sol`
  Reasoning effort is passed via request parameters (`reasoningEffort: "medium"`) or OpenCode variants, not embedded in the model ID string. If configured with proxy-style IDs, capability checks reject the model.

### 3.4. Fallback Chain Behavior
In [`packages/omo-opencode/src/plugin-handlers/agent-config-assembly.ts`](file:///c:/Users/Fate_Conqueror/GitHub/oh-my-openagent/packages/omo-opencode/src/plugin-handlers/agent-config-assembly.ts), if an agent's configured model:
- Fails schema validation (e.g. invalid fields or dropped keys), OR
- Fails capability checks / availability resolution,
the agent loader silently falls back to `systemDefaultModel` (which was set to `9router/codex/gpt-5.6-sol-medium`). Consequently, the session booted with 9router across all agents without throwing an explicit crash.

---

## 4. How to Properly Configure Native OpenAI Models for Agents

When using direct OpenAI (OAuth via `auth.json` or `OPENAI_API_KEY`):

1. **Do NOT add OpenAI models to `opencode.json`.**
2. **In `~/.omo/omo.jsonc`, use the exact canonical model ID and `"reasoningEffort"`:**
```jsonc
{
  "opencode": {
    "agents": {
      "sisyphus": {
        "model": "openai/gpt-5.6-sol",
        "reasoningEffort": "medium"
      },
      "hephaestus": {
        "model": "openai/gpt-5.6-sol",
        "reasoningEffort": "medium"
      },
      "oracle": {
        "model": "openai/gpt-6-sol",
        "reasoningEffort": "high"
      },
      "explore": {
        "model": "openai/gpt-5.6-terra"
      }
    }
  }
}
```
Alternatively, OpenCode's native variant syntax:
```jsonc
{
  "opencode": {
    "agents": {
      "sisyphus": {
        "model": "openai/gpt-5.6-sol",
        "variant": "medium"
      }
    }
  }
}
```

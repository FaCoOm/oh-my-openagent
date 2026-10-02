# Hephaestus Agent Registration & Loading Failure Root Cause Analysis

## Executive Summary

When launching OpenCode with `oh-my-openagent`, pressing the `Tab` key cycles through the **primary agents**. By design, there are **4 primary agents**:
1. `sisyphus` (Primary general worker / orchestrator)
2. `hephaestus` (Primary autonomous deep worker)
3. `prometheus` (Primary strategic plan builder)
4. `atlas` (Primary plan executor)

Currently, the TUI displays only **3** primary agents (`sisyphus`, `prometheus`, `atlas`) because the **Hephaestus agent registration gate silently skips loading Hephaestus** when its configured model fails strict architecture requirements.

---

## Detailed Root Cause Analysis

### 1. Evidence from Logs
In `%TEMP%\oh-my-opencode.log`, the runtime log documents the exact skip event:
```json
[2026-10-02T07:22:41.326Z] [agent-registration] Agent skipped: unsupported Hephaestus model {"agent":"hephaestus","configuredModel":"openai/gpt-6-sol"}
[2026-10-02T07:23:40.879Z] [auto-compact] session.error received {"sessionID":"...","error":{"name":"UnknownError","data":{"message":"Agent not found: \"hephaestus\". Available agents: Sisyphus - ultraworker, Atlas - Plan Executor, Prometheus - Plan Builder, ..."}}}
```

### 2. The Model Validation Gate (`isHephaestusSupportedModel`)
In `oh-my-openagent` agent registration (`packages/omo-opencode/src/agents/builtin-agents/hephaestus-agent.ts`):
```typescript
if (!isHephaestusSupportedModel(hephaestusModel)) {
  log("[agent-registration] Agent skipped: unsupported Hephaestus model", {
    agent: "hephaestus",
    configuredModel: hephaestusModel,
  });
  return undefined; // Agent is not added to the primary agents map!
}
```

In the currently active release (`oh-my-openagent@4.19.4` installed in `~/.cache/opencode/packages/`):
```javascript
var GPT_5_3_CODEX_RE = /^gpt-5[.-]3-codex(?:$|[.-])/i;
var GPT_5_4_RE = /^gpt-5[.-]4(?:$|[.-])/i;
var GPT_5_5_RE = /^gpt-5[.-]5(?:$|[.-])/i;
var GPT_5_6_RE = /^gpt-5[.-]6(?:$|[.-])/i;

function extractModelName(model) {
  return model.includes("/") ? model.split("/").pop() ?? model : model;
}

function isHephaestusSupportedModel(model) {
  if (!model) return false;
  const modelName = extractModelName(model);
  return (
    GPT_5_3_CODEX_RE.test(modelName) ||
    GPT_5_4_RE.test(modelName) ||
    GPT_5_5_RE.test(modelName) ||
    GPT_5_6_RE.test(modelName)
  );
}
```

### 3. What Went Wrong Across Configurations
There are two reasons Hephaestus failed under your configuration:

1. **Attempt with `openai/gpt-6-sol`**:
   - In published version `4.19.4`, `GPT_6_RE` was **not yet supported** in `isHephaestusSupportedModel`. The regex list stopped at `GPT_5_6_RE`. (Support for GPT-6 has been added to the unreleased repository source on the `dev` branch, but is not present in the installed `4.19.4` bundle).
   - Because `gpt-6-sol` failed `isHephaestusSupportedModel`, Hephaestus was skipped.

2. **Attempt with `opencode/space-bunny-free`**:
   - In `C:\Users\Fate_Conqueror\.config\opencode\oh-my-openagent.json`, `hephaestus.model` was subsequently set to:
     ```json
     "hephaestus": {
       "model": "opencode/space-bunny-free",
       ...
     }
     ```
   - Hephaestus is designed strictly and exclusively for OpenAI GPT Codex family models (it has dedicated system prompts and hooks like `no-hephaestus-non-gpt`).
   - The string `"space-bunny-free"` does not match any GPT regex, so `isHephaestusSupportedModel` evaluated to `false`, and registration returned `undefined`.

### 4. Why Reinstalling Did Not Change Anything
1. Reinstalling pulls down the published npm package (`4.19.4`), which still contains the `4.19.4` validation gate.
2. Reinstalling does not alter existing configuration files in `~/.config/opencode/oh-my-openagent.json` or `~/.omo/omo.jsonc`.
3. If no explicit model is configured, Hephaestus's default fallback chain requires `openai`, `chatgpt-subscription`, `github-copilot`, or `opencode` with a valid GPT-5.x model. Your OpenCode environment currently has custom providers configured (`newapi`, `omniroute`, `9router`), so the default fallback cannot auto-resolve without an override.

---

## Solutions & Recommended Actions

### Solution A: Configure a Supported GPT-5.6 Model (Recommended)
You already have providers with supported GPT-5.6 models available in your OpenCode configuration. In `C:\Users\Fate_Conqueror\.config\opencode\oh-my-openagent.json`, configure Hephaestus to use one of your working GPT-5.6 endpoints:

#### Option A1: Via `omniroute`
```json
"hephaestus": {
  "model": "omniroute/codex/gpt-5.6-sol-medium",
  "variant": "medium",
  "reasoning": "medium"
}
```

#### Option A2: Via `9router`
```json
"hephaestus": {
  "model": "9router/cx/gpt-5.6-sol[1m]",
  "variant": "medium",
  "reasoning": "medium"
}
```

Both models match the `GPT_5_6_RE` gate in version `4.19.4`. Once updated and OpenCode is restarted, all **4** primary agents (`sisyphus`, `hephaestus`, `prometheus`, `atlas`) will cycle with `Tab`.

### Solution B: If You Want to Use `gpt-6` (e.g. `openai/gpt-6-sol` or `omniroute/codex/gpt-6-sol-medium`)
If you prefer running GPT-6 models for Hephaestus, version `4.19.4` must be updated with the `GPT_6_RE` regex:
- Update `dist/index.js` in `C:\Users\Fate_Conqueror\.cache\opencode\packages\oh-my-openagent\node_modules\oh-my-openagent\dist\index.js` or build the local repo (`bun run build`).
- In `dist/index.js`, add `const GPT_6_RE = /^gpt-6(?:$|[.-])/i;` to `isHephaestusSupportedModel`.

### Note on Non-GPT Models (e.g., `space-bunny-free`)
Non-GPT models cannot be used with Hephaestus. For non-GPT models, open-weights models, or Claude/Kimi/GLM models, use **Sisyphus** (`sisyphus`), which supports arbitrary model architectures without restriction.

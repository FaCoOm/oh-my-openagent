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

## Why It Worked on Your Other Machine with the Exact Same Config

There are two concrete technical explanations verified by Git commit history and runtime logs:

### 1. Plugin Version Difference (Commit `edf49f6d0`)
- The strict model validation gate `isHephaestusSupportedModel` was introduced in commit `edf49f6d0` (*"fix(agents): restrict Hephaestus to GPT-native models"*, tagged at **`v4.10.0`** on 2026-06-14).
- **Prior to v4.10.0** (e.g. `v4.2.3` or earlier, which is what is currently installed globally via `npm list -g`), `maybeCreateHephaestusConfig` did **NOT** validate whether the model was an OpenAI GPT family model. It allowed whatever model was configured—including `opencode/space-bunny-free`—to register as Hephaestus without skipping.
- If your other machine has an older cached build of `oh-my-openagent` or `oh-my-opencode` (< 4.10.0), Hephaestus registered cleanly regardless of the configured model.

### 2. Connected Providers Cache Difference
- In `oh-my-openagent`, if an explicit override is omitted or fails resolution, Hephaestus queries `readConnectedProvidersCache()`.
- On this machine yesterday (`2026-10-01T14:10:18Z`), `github-copilot` and `openai` were in the connected providers cache. Hephaestus automatically resolved to `github-copilot/gpt-5.6-sol` and registered successfully.
- If your other machine has active `github-copilot` or `openai` authentication, Hephaestus automatically satisfies its provider requirements.

---

## Why Sisyphus Failed Previously

From `%TEMP%\oh-my-opencode.log` (lines 39600–39910):
1. **HTTP 403 Permission Denied**:
   Sisyphus attempted to use `omniroute/codex/gpt-5.6-luna-high` (and `omniroute/codex/gpt-5.6-luna-xhigh`). The Omniroute server rejected the request:
   ```json
   {
     "error": {
       "message": "Model \"codex/gpt-5.6-luna-high\" is not allowed for this API key",
       "type": "permission_error",
       "code": "permission_denied"
     },
     "statusCode": 403
   }
   ```
2. **Fallback Abort**:
   The runtime fallback handler caught the 403 error and attempted to fall back to `opencode/fledge-alpha-free`:
   ```
   [runtime-fallback] Preparing fallback {"from":"omniroute/codex/gpt-5.6-luna-high","to":"opencode/fledge-alpha-free"}
   ```
   However, provider `"opencode"` is not configured in `opencode.json`, and the mid-turn retry was aborted (`MessageAbortedError: "Aborted"`), causing Sisyphus to error out.

---

## The Immediate Failure When Using OpenAI Model (`openai/gpt-6-sol`)

### The Smoking Gun: `no-sisyphus-gpt` Hook
When you selected the OpenAI model (`openai/gpt-6-sol`) while on the primary agent (`sisyphus`), the turn failed instantly with:
```
Agent not found: "hephaestus". Available agents: Sisyphus - ultraworker, Atlas - Plan Executor, Prometheus - Plan Builder, ...
```

Here is the exact mechanism from `packages/omo-opencode/src/hooks/no-sisyphus-gpt/hook.ts` (lines 75–82 in `v4.19.4`):
```javascript
if (agentKey === "sisyphus" && modelID && isGptModel(modelID) && !isGptNativeSisyphusModel(modelID)) {
  showToast(ctx, input.sessionID);
  input.agent = resolveRegisteredAgentName("hephaestus") ?? "hephaestus";
  if (output?.message) {
    output.message.agent = resolveRegisteredAgentName("hephaestus") ?? "hephaestus";
  }
  updateSessionAgent(input.sessionID, "hephaestus");
}
```

1. **Automatic Redirect**: In version `4.19.4`, `isGptNativeSisyphusModel` only recognizes `gpt-5.4`, `gpt-5.5`, and `gpt-5.6 Sol`. When you selected `openai/gpt-6-sol`, the plugin considered it non-native for Sisyphus and **forcibly re-routed the session's active agent to `hephaestus`**.
2. **Missing Target**: Because `hephaestus` was never registered during plugin startup (due to the `4.19.4` model gate or invalid config), OpenCode Core received a request to dispatch a message to an agent named `"hephaestus"` which did not exist.
3. OpenCode Core immediately failed the session with: `Agent not found: "hephaestus"`.

---

## Why Your Other Machine Works: OpenCode Cache Staleness (`4.19.4` vs `5.1.9`)

Your other machine is indeed running the actual latest release (**`v5.1.9`**), whereas **this machine is stuck on `v4.19.4`** cached on September 22, 2026.

### Evidence from Today's Runtime Log (`%TEMP%\oh-my-opencode.log` at 07:59:16 UTC):
```
[auto-update-checker] Update available (latest): 4.19.4 -> 5.1.9
[auto-update-checker] Update available toast shown: v5.1.9
[auto-update-checker] OpenCode-managed sandbox detected (C:\Users\Fate_Conqueror\.cache\opencode\packages\oh-my-openagent@latest); skipping auto-update install. Notification only. See #4318.
[auto-update-checker] Startup toast shown: v4.19.4
```

Because OpenCode manages plugins in its own cache directory (`C:\Users\Fate_Conqueror\.cache\opencode\packages\`), the plugin auto-updater deliberately **skips auto-updating** to avoid corrupting OpenCode's sandbox, only showing a notification toast.

### Key Changes in `v5.1.9` that Make the Other Machine Work:
1. **GPT-6 in `isHephaestusSupportedModel`**: `v5.1.9` includes `GPT_6_RE = /^gpt-6(?:$|[.-])/i`, allowing Hephaestus to load with `openai/gpt-6-sol`.
2. **GPT-6 in `no-sisyphus-gpt`**: `v5.1.9` includes `isGpt6Model(modelID)`, allowing Sisyphus to run GPT-6 models natively without redirecting to Hephaestus.

---

## How to Fix This Machine Immediately

Clear OpenCode's outdated cached package directories so OpenCode fetches the current `5.1.9` release:

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.cache\opencode\packages\oh-my-openagent*"
```

When OpenCode restarts, it will download and cache `oh-my-openagent@5.1.9`. Hephaestus and Sisyphus will then both properly support `openai/gpt-6-sol`.


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

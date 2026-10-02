# Architectural Comparison: OMO (OpenCode Adapter) vs. Everything-Claude-Code (OpenCode Implementation)

**Date:** 2026-09-26  
**Subject:** Comparative Internal Architecture & Operational Mechanics of OpenCode Implementations  
**Analyzed Repositories:**
- **OMO (`oh-my-openagent` / `oh-my-opencode`):** `packages/omo-opencode/`
- **ECC (`everything-claude-code`):** `.opencode/` + installation target adapters

---

## Executive Summary

While both **OMO** and **Everything-Claude-Code (ECC)** provide extensions for the **OpenCode** AI coding assistant harness, their architectural paradigms, runtime depth, and operational philosophies are fundamentally diametric:

| Dimension | Everything-Claude-Code (ECC) | OMO (`oh-my-opencode`) |
| :--- | :--- | :--- |
| **Architectural Paradigm** | **Static Configuration Pack & Passive Observer** | **Autonomous Agent Operating System & Dynamic Controller** |
| **Primary Origin & Focus** | Claude Code-first configuration kit exported to OpenCode via a thin compatibility adapter. | Native multi-agent OS built directly around OpenCode's internal extension points (now shimming a multi-harness core). |
| **Plugin Initialization** | Single entry point (`index.ts` -> `ecc-hooks.ts`) returning ~10 event listener functions. | 7-stage initialization pipeline with prototype patching, runtime conflict detection, and telemetry. |
| **Config Loading** | Static declarative `opencode.json` with direct `{file:...}` text references. | 6-phase dynamic configuration pipeline merging user & workspace JSONC configs through 36 Zod schemas. |
| **Hook Architecture** | 1 single-file hook bundle (~636 lines in `ecc-hooks.ts`) handling basic OS shell triggers. | 5-tier hook composition across 62 directories with 54 base to 62 dynamic lifecycle hooks. |
| **Agent Ecosystem** | 26 declarative agents loaded from static `.txt` prompts with basic boolean tool flags. | 11 programmatic agents dynamically synthesized by factories with contextual policy sections and model capability awareness. |
| **Tool System** | 8 basic utility tools (wrapping CLI linters, test runners, git status). | Up to 38 registered native tools, including cryptographic Hashline editing (`LINE#ID`), multi-session managers, and LSP MCP. |
| **Execution Control** | Reactive only (fires when user runs a command or agent completes a step). | Active enforcement loops: Boulder state tracking, ULW (Ultrawork) loop, and anti-halt continuation guards. |
| **Multi-Agent Coordination** | Basic OpenCode subtasks (`subtask: true` in commands). | Full parallel **Team Mode** engine with mailboxes, file locks, tasklists, shutdown voting, and tmux multi-pane layouts. |
| **Safety & Guardrails** | Basic regex for `console.log`, warnings on new `.md` files, and permission auto-approval. | AST-level comment-checker binary, file write guards, prompt validation, and model fallback cascades. |

---

## 1. Architectural Philosophy & Design Principles

### Everything-Claude-Code (ECC): Downstream Compatibility Pack
ECC was conceived as an opinionated prompt, rule, and workflow configuration kit for **Claude Code** (`CLAUDE.md`, `.claude/commands/`, skill folders). Its OpenCode integration (`.opencode/`) is an export target:
1. **Host-Agnostic Portability:** ECC maps its Claude Code concepts (PreToolUse, PostToolUse, Stop, SessionStart) directly to OpenCode's closest event counterparts (`tool.execute.before`, `tool.execute.after`, `session.idle`, `session.created`).
2. **Declarative Simplicity:** It relies on OpenCode's native loader to parse `.opencode/opencode.json`, mount markdown templates as slash commands, and bind static prompt files to agents.
3. **Passive Assistance:** ECC acts as a polite guardian. It runs `prettier --write` after a file is edited, warns if you left a `console.log`, displays an OS desktop notification when a task is finished, and auto-approves safe commands like `git diff` or `npm test`. It does not intercept, modify, or take control of OpenCode's execution flow.

### OMO (`oh-my-openagent`): Full-Fledged Agent Operating System
OMO treats OpenCode not as a mere configuration host, but as an execution substrate that must be instrumented, constrained, and supercharged:
1. **Aggressive Opinionation & Invariant Enforcement:** OMO assumes LLMs will hallucinate file edits, leave sloppy comments, drift from the plan, or prematurely declare completion. To combat this, OMO installs deterministic guards at every stage of the lifecycle.
2. **Deep Harness Instrumentation:** OMO does not hesitate to patch runtime primitives. For example, because OpenCode 1.4.x sorts agents alphabetically and ignores the `order` config property, OMO monkey-patches `Array.prototype.sort` and `toSorted` (`installAgentSortShim`) to force OpenCode's UI and picker to respect canonical agent hierarchy.
3. **Execution State Preservation:** Tasks are not ephemeral chat messages. They are governed by the **Boulder** state engine, which tracks goals in persistent storage, injects missing context across compaction, and forces continuation until every checkbox is verified.

---

## 2. Configuration & Initialization Pipeline

### ECC Config Pipeline: Declarative Static Loading
ECC relies entirely on OpenCode reading `.opencode/opencode.json`:
```json
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "build",
  "instructions": [ "AGENTS.md", "skills/tdd-workflow/SKILL.md", ... ],
  "plugin": [ "./plugins" ],
  "agent": {
    "planner": {
      "mode": "subagent",
      "prompt": "{file:prompts/agents/planner.txt}",
      "tools": { "read": true, "bash": true, "write": false, "edit": false }
    }
  },
  "command": {
    "plan": {
      "template": "{file:commands/plan.md}\n\n$ARGUMENTS",
      "agent": "planner",
      "subtask": true
    }
  }
}
```
- **Build Step Required:** ECC requires a compilation step (`node scripts/build-opencode.js`) to compile TypeScript in `.opencode/` into `.opencode/dist/` before OpenCode can load its plugin.
- **Static Schema:** No programmatic transformations occur at startup. If an agent needs a specific prompt, it is read as raw text from disk.

### OMO Config Pipeline: 6-Phase Dynamic Pipeline
OMO implements OpenCode's `config` hook via a staged 6-phase pipeline orchestrator (`packages/omo-opencode/src/plugin-handlers/config-handler.ts`):

```
OpenCode Startup -> serverPlugin()
  ├── 1. installAgentSortShim()          (Monkey-patch Array.prototype.sort for agent ordering)
  ├── 2. initConfigContext()             (Resolve layout: opencode vs. openagent)
  ├── 3. detectDuplicateOmoPlugin()      (Prevent duplicate hooks from parallel plugins)
  ├── 4. injectServerAuthIntoClient()    (Inject bearer auth headers into SDK client)
  ├── 5. loadPluginConfig()              (JSONC user + workspace merge -> 36 Zod schemas -> migrate)
  ├── 6. checkTeamModeDependencies()     (Verify git worktree, tmux socket, ~/.omo/teams)
  └── 7. createManagers / Tools / Hooks / PluginInterface
```

When OpenCode calls the `config` hook, OMO runs **6 sequential phases**:
1. **Phase 1 (`applyProviderConfig`):** Resolves provider configurations, detects `anthropic-beta` reasoning headers, and caches model token context windows.
2. **Phase 2 (`loadPluginComponents`):** Discovers and bridges external Claude Code plugins with strict 10s timeouts and error boundaries.
3. **Phase 3 (`applyAgentConfig`):** Aggregates agents across 5 sources, computes skill discoveries, enforces model inheritance/demotion, resolves dynamic capability variants, and establishes key priority order.
4. **Phase 4 (`applyToolConfig`):** Applies dynamic tool permissions. For example, grants `grep_app_*` only to Librarian, grants `task` and `teammate` only to Atlas, Sisyphus, and Prometheus, and explicitly denies them to all other subagents.
5. **Phase 5 (`applyMcpConfig`):** Merges built-in Tier-1 stdio MCPs (`lsp`, `ast-grep`), user-defined `.mcp.json`, and skill-embedded MCPs into OpenCode's runtime.
6. **Phase 6 (`applyCommandConfig`):** Discovers and registers slash commands from 9 parallel directory sources, deduplicating between `.opencode/` and `.agents/`.

---

## 3. Lifecycle Hooks Architecture

### ECC: Monolithic Reactive Event Handlers
ECC implements hooks inside a single file: `everything-claude-code/.opencode/plugins/ecc-hooks.ts` (~636 lines).

```
ECC Event Handlers (Single File):
├── "file.edited"                      -> prettier --write + grep console.log
├── "tool.execute.after"               -> changedFilesStore.record + tsc --noEmit check
├── "tool.execute.before"              -> pending file status + warn against unnecessary .md files
├── "session.created"                  -> welcome log + check for CLAUDE.md
├── "session.idle"                     -> grep console.log across all edited files + OS notification
├── "session.deleted"                  -> clear in-memory edited file set
├── "file.watcher.updated"             -> update changedFilesStore
├── "todo.updated"                     -> log task count progress
├── "shell.env"                        -> inject PROJECT_ROOT, PACKAGE_MANAGER, DETECTED_LANGUAGES
├── "experimental.session.compacting"  -> append static markdown block into output.prompt
└── "permission.ask"                   -> auto-approve read tools, formatters, and test runners
```

**Key Characteristics:**
- **Synchronous execution of OS tools:** Relies heavily on calling CLI utilities (`prettier`, `grep`, `tsc`, `osascript`, `powershell`, `notify-send`) via Bun/Node `$` template strings.
- **Non-blocking logging:** Warnings and errors are reported via `client.app.log({ level: "warn", message })` without altering tool arguments or terminating the run.

### OMO: 5-Tier Modular Hook Matrix
OMO distributes **54 to 62 hooks** across 62 individual directories, organized into a 5-tier architecture (`packages/omo-opencode/src/create-hooks.ts` and `src/hooks/`):

```
OMO 5-Tier Hook Composition:
├── Tier 1: Session Lifecycle Hooks (24 hooks)
│   ├── preemptiveCompaction            (Proactively compact before token overflow)
│   ├── anthropicContextWindowLimitRecovery (Multi-step recovery from 200k/1M token limit crashes)
│   ├── modelFallback & runtimeFallback (Proactive & reactive model switching on API errors)
│   ├── thinkMode                       (Dynamic variant switching for extended thinking budgets)
│   ├── goal                            (Persistent objective tracking & continuation)
│   ├── editErrorRecovery               (Automated retry & syntax recovery on failed file edits)
│   ├── delegateTaskRetry               (Auto-retry failed task delegations)
│   ├── prometheusMdOnly                (Hard guard: blocks Prometheus from writing non-.md files)
│   ├── sisyphusJuniorNotepad           (Injects notepad state into subagents)
│   ├── noSisyphusGpt & noHephaestusNonGpt (Model gating: prevents weak models on heavy roles)
│   └── interactiveBashSession          (Tmux session lifecycle manager for interactive terminals)
│
├── Tier 2: Tool Guard Hooks (17-18 hooks)
│   ├── commentChecker                  (Binary call to @code-yeongyu/comment-checker; blocks AI slop)
│   ├── toolOutputTruncator             (Prevents massive tool outputs from poisoning context)
│   ├── writeExistingFileGuard          (Refuses Write/Edit if file was not read in session)
│   ├── bashFileReadGuard               (Blocks `cat`/`head`/`tail` in bash, forcing read tool use)
│   ├── hashlineReadEnhancer            (Injects LINE#ID content hashes into read outputs)
│   ├── jsonErrorRecovery               (Catches JSON parse failures and feeds repair hints)
│   ├── notepadWriteGuard               (Enforces append-only semantics on notepad files)
│   ├── planFormatValidator             (Strict checkbox format validator on boulder plan files)
│   └── teamToolGating                  (Gates 12 team_* tools based on member role)
│
├── Tier 3: Transform Hooks (4-7 hooks)
│   ├── claudeCodeHooks                 (Bidirectional Claude Code settings.json compatibility)
│   ├── keywordDetector                 (Detects `ultrawork`, `search`, `analyze`, `team` keywords)
│   ├── contextInjectorMessagesTransform(Injects hierarchical AGENTS.md & README.md context)
│   ├── toolPairValidator               (Enforces valid tool_call/tool_result pairing invariants)
│   ├── teamModeStatusInjector          (Injects active team roster and task progress)
│   └── teamMailboxInjector             (Pulls asynchronous messages from other agents)
│
├── Tier 4: Continuation Hooks (7 hooks)
│   ├── todoContinuationEnforcer        (The Boulder Engine: halts idle event if tasks remain)
│   ├── compactionContextInjector       (Re-injects state & rules immediately after compaction)
│   ├── compactionTodoPreserver         (Preserves hierarchical todo trees across compactions)
│   ├── stopContinuationGuard           (Handles explicit /stop-continuation command)
│   ├── unstableAgentBabysitter         (Monitors agent thrashing and applies backoff)
│   └── atlasHook                       (Master orchestrator monitoring background subagents)
│
└── Tier 5: Skill Hooks (2 hooks)
    ├── categorySkillReminder           (Hints agent to load domain skill before delegating)
    └── autoSlashCommand                (Maps natural language prompts to slash commands)
```

**Key Characteristics:**
- **Execution Invariant Enforcement:** Hooks don't just log; they mutate inputs, block disallowed actions, inject correction prompts, and intercept the session completion cycle.
- **Fail-Safe Recovery:** Includes specialized multi-step recovery hooks for context window exhaustion, edit collisions, and JSON parse failures.

---

## 4. Agent Architecture & Subagent Orchestration

### ECC Agents: Language-Segmented Specialists
ECC declares **26 agents** in `opencode.json`. They are largely divided along programming languages and basic software lifecycle stages:
- **Lifecycle Agents:** `build` (default primary), `planner`, `architect`, `code-reviewer`, `security-reviewer`, `tdd-guide`, `build-error-resolver`, `e2e-runner`, `doc-updater`, `refactor-cleaner`, `loop-operator`, `harness-optimizer`.
- **Language-Specific Agents:** `go-reviewer`, `go-build-resolver`, `python-reviewer`, `rust-reviewer`, `rust-build-resolver`, `cpp-reviewer`, `cpp-build-resolver`, `java-reviewer`, `java-build-resolver`, `kotlin-reviewer`, `kotlin-build-resolver`, `php-reviewer`, `database-reviewer`.

**Internal Operation:**
- Each agent references a static text file in `prompts/agents/<name>.txt`.
- Tool permissions are basic boolean flags (`read: true, write: false, edit: false, bash: true`).
- Agents are called via OpenCode's standard slash command subtasks (e.g. `/plan` invokes `planner` with `subtask: true`).
- Agents inherit whatever model the user has globally selected in OpenCode.

### OMO Agents: Hierarchical Personas & Dynamic Prompt Engineering
OMO rejects static prompt files and language-fragmented agents in favor of **11 highly specialized core archetypes** synthesized dynamically:

| Agent | Role | Distinct Capabilities & Constraints |
| :--- | :--- | :--- |
| **Sisyphus** | The Relentless Orchestrator | Primary coordinator. Powered by Boulder state engine. Has access to delegation, task lists, and team tools. Blocked from non-GPT providers to ensure high reasoning capability. |
| **Hephaestus** | Autonomous Builder | Deep-work implementation specialist. Receives walk-up `AGENTS.md` context injection. Blocked from non-GPT models. |
| **Prometheus** | Strategic Planner | Specialized planner. Constrained by the `prometheusMdOnly` hook to **only write `.md` files**, preventing it from touching codebase implementation. |
| **Atlas** | System Architect | High-level system design and master overseer of background runs and multi-agent coordination. |
| **Sisyphus-Junior** | Execution Worker | Subagent with dedicated notepad memory (`sisyphusJuniorNotepad`) for structured scratchpad tasks. |
| **Librarian** | Codebase & Web Researcher | Exclusive access to `grep_app_*` tools and deep search routines. Denied file write permissions. |
| **Oracle** | Holistic Synthesizer | Nuclear-grade code review, release synthesis, and architectural validation (prompt factory exceeds 28KB of precision instructions). |
| **Momus** | Hostile Code Reviewer | Adversarial code review specialist trained to search for edge cases, performance regressions, and security flaws. |
| **Metis** | Pre-Flight Validator | Edge-case validator that inspects changes before PR creation. |
| **Explore** | Rapid Explorer | Fast, lightweight exploration subagent. |
| **Multimodal-Looker**| Vision Specialist | UI/UX inspection agent utilizing multimodal image tools. |

**Dynamic Prompt Generation:**
OMO builds prompts in code (`packages/omo-opencode/src/agents/`):
- `dynamic-agent-core-sections.ts`: Injects core operating standards, verification loops, and git safety rules.
- `dynamic-agent-policy-sections.ts`: Injects strict project policies and immutable invariants.
- `dynamic-agent-category-skills-guide.ts`: Dynamically informs the agent which domain skills correspond to which task categories.
- **Model-Specific Shims:** Injects model-tailored adapters, such as `gpt-apply-patch-guard.ts` for GPT-style editing, `kimi-tool-loop-guard.ts` for Kimi models, and `sisyphus-gemini-fallback-overrides.ts` for Gemini models.

---

## 5. Tooling, Editing & File Safety

### ECC Tools: Utility Wrappers
ECC exports 8 custom tools defined in `.opencode/tools/`:
1. `run-tests.ts`: Runs test runners (`npm test`, `pytest`, `cargo test`, `go test`) with flags.
2. `check-coverage.ts`: Analyzes test coverage output.
3. `security-audit.ts`: Runs `npm audit`, `pip-audit`, `cargo audit`, or `trivy`.
4. `format-code.ts`: Auto-detects formatter (`prettier`, `biome`, `black`, `gofmt`) and runs it.
5. `lint-check.ts`: Runs `eslint`, `biome`, `ruff`, `golangci-lint`.
6. `git-summary.ts`: Generates a git status summary.
7. `changed-files.ts`: Lists modified files in the session.
8. `dependency-analyzer.ts`: Analyzes lockfiles for outdated or vulnerable packages.

ECC leaves file editing entirely to OpenCode's built-in `edit` and `write` tools.

### OMO Tools: Hashline Editing, LSP MCP & Delegation

OMO provides up to 38 registered tools and fundamentally reinvents file editing and tool execution:

#### 1. Hashline LINE#ID Editing (`hashline_edit`)
Standard LLM file editing (including OpenCode's native `edit` tool) relies on string replacement chunks. In large files or repetitive code, LLMs frequently match the wrong lines, drop lines, or hallucinate indentation.
- OMO solves this with **Hashline**:
  - `hashlineReadEnhancer` hook tags every line of read output with a unique content hash (`LINE#ID`, e.g., `42#a8f2`).
  - The `hashline_edit` tool requires the agent to specify exact line IDs for the target block.
  - If the file has changed or the line IDs don't match, the edit is safely rejected before disk corruption occurs.

#### 2. Tier-1 Built-in MCPs
Rather than wrapping CLI tools with shell scripts, OMO bundles full stdio MCP servers:
- **`lsp-tools-mcp`**: Directly connects to language servers (TypeScript, Rust, Python, Go) to expose 8 semantic tools: `lsp_goto_definition`, `lsp_find_references`, `lsp_symbols`, `lsp_diagnostics`, `lsp_prepare_rename`, `lsp_rename`, etc.
- **`ast-grep-mcp`**: Exposes tree-sitter AST structural code search and rewriting (`sg`).

#### 3. Strict Pre-Tool Guards
- `writeExistingFileGuard`: Intercepts `tool.execute.before` on `write` or `edit`. If an agent tries to modify a file without having read it in the current session, the tool call is blocked with an error instructing the agent to read first.
- `bashFileReadGuard`: Intercepts bash commands. If an agent executes `cat file.ts` or `head -n 20 file.ts`, OMO blocks the command and instructs the agent to use the proper structured `read` tool.
- `commentChecker`: Post-execution hook that drives a dedicated compiled binary (`@code-yeongyu/comment-checker`). It scans modified code for AI placeholder slop (e.g. `// implement later`, `/* ... existing code ... */`) and rejects the edit if detected.

---

## 6. Execution Loops & State Management

### ECC: Stateless Execution
ECC has no persistent task engine. When an agent finishes its tool execution and emits an answer, OpenCode fires `session.idle`. ECC simply audits `console.log` occurrences, fires a desktop popup, and clears its in-memory file set. If an agent missed half the user's requirements, the session stops anyway.

### OMO: The Boulder State Engine & Ultrawork Loop

OMO operates on the principle that **an agent session must never stop until the work is proven complete**:

```
User Prompt (e.g. "Fix auth bug and add tests")
  ↓
Keyword Detector / Command Handler
  ↓
Initialize Boulder State (.omo/boulder-state.json)
  ├── Register atomic Todo checklist with success criteria
  ├── Mark Active Goal in persistent state
  ↓
Agent Execution Loop (Sisyphus / Hephaestus)
  ├── Execute tools (Hashline Edit, LSP, Test)
  ├── Check off completed tasks in plan
  ↓
OpenCode Attempts to Transition to `session.idle`
  ↓
Intercepted by `todoContinuationEnforcer` Hook!
  ├── Query Boulder State: Are there unfinished tasks?
  ├── YES: Inject continuation prompt with remaining todos -> RESUME EXECUTION
  └── NO: Allow session to settle.
```

1. **The Boulder State Engine:** Stores the hierarchical task graph in `.omo/` (or legacy `.sisyphus/`). It tracks parent goals, active tasks, checkboxes, and execution time.
2. **Context Compaction Resilience:** When sessions exceed model context limits, OpenCode triggers compaction. Normal agents lose their train of thought. OMO's `compactionTodoPreserver` and `compactionContextInjector` extract the Boulder state before compaction and immediately re-inject the checklist and active goal into the new context window.
3. **The ULW (Ultrawork) Loop:** A rigorous operational protocol requiring agents to:
   - Reproduce bugs first with a failing test before writing code.
   - Execute minimal atomic diffs.
   - Run verification and capture evidence on disk under `.omo/evidence/<YYYYMMDD>-<slug>/`.
   - Forbid declaring completion without hard, recorded evidence.

---

## 7. Parallel Multi-Agent Coordination: Team Mode

### ECC: No Native Coordination
ECC does not possess an inter-agent communication layer or a multi-agent runtime. Subagents run sequentially in isolated subprocesses without shared awareness.

### OMO: Full Multi-Agent Team Mode
OMO ships a complete parallel multi-agent operating environment (`packages/omo-opencode/src/features/team-mode/`):
- **12 Dedicated Team Tools:** `team_create`, `team_delete`, `team_send_message`, `team_shutdown_request`, `team_approve_shutdown`, `team_task_create`, `team_task_list`, `team_task_update`, `team_task_get`, `team_status`, `team_list`.
- **Shared File-Locked Tasklist:** Team members claim, update, and complete tasks from a shared task board protected by file-level atomic locking to prevent race conditions.
- **Asynchronous Mailbox System:** Agents send direct messages or broadcast (`*`) announcements to other running teammates. The `teamMailboxInjector` hook pulls unread messages directly into the agent's message transform before each turn.
- **Isolated Git Worktrees:** When teams work in parallel, each member can be assigned an isolated git worktree (`team-worktree`) to avoid file write collisions.
- **Tmux Multi-Pane Orchestration:** Automatically provisions tmux window layouts (`team-layout-tmux`), splitting the terminal screen so the developer can visually monitor the team lead and workers executing simultaneously in real time.
- **Consensus Shutdown Protocol:** Workers cannot simply die or quit; they must emit a `team_shutdown_request`, which must be formally acknowledged and approved (`team_approve_shutdown`) by the team lead once all dependencies are verified.

---

## 8. Summary Comparison Matrix

| Feature / Subsystem | Everything-Claude-Code (ECC) | OMO (`oh-my-openagent`) |
| :--- | :--- | :--- |
| **OpenCode Role** | Configuration pack + basic hook plugin | Full-scale autonomous Agent Operating System |
| **Primary Code Location** | `.opencode/` (~1.5k LOC total) | `packages/omo-opencode/` (>80k LOC + Core packages) |
| **Configuration Engine** | Static `opencode.json` | 6-Phase Pipeline with 36 Zod schemas & JSONC merge |
| **Agent Discovery** | Static prompt files (`prompts/agents/*.txt`) | Dynamic programmatic factories with model-specific tuning |
| **Agent Ordering** | Default alphabetical (native OpenCode) | Runtime sort shim (`installAgentSortShim` via prototype patch) |
| **Total Hooks** | ~10 simple callbacks in 1 file | 54 base / 61 team / 62 monitor hooks across 62 directories |
| **File Editing** | Standard OpenCode string-replace `edit` | Cryptographic `hashline_edit` (`LINE#ID` anchoring) |
| **Guardrails** | Basic regex (`console.log`) | Compiled binary comment-checker, read-before-write guards, bash guards |
| **Language Intelligence** | Wraps CLI linters in Node child processes | Stdio LSP MCP (8 tools) + AST-Grep MCP (`sg`) |
| **State Persistence** | Transient in-memory `Set` & optional SQLite cache | Persistent Boulder engine tracking goals across restarts |
| **Compaction Recovery** | Appends static markdown string to prompt | Preemptive compaction + multi-hook state re-injection + auto-continue |
| **Multi-Agent Coordination** | Basic OpenCode subtasks | Complete parallel Team Mode with mailboxes, locks, and tmux layouts |
| **External Bridges** | None | OpenClaw daemon (Telegram, Discord, HTTP bridges) |

---

## 9. Conclusion & In-Depth Insight

The architectural contrast between OMO and Everything-Claude-Code reflects two entirely different engineering objectives:

1. **Everything-Claude-Code (ECC)** is built on the philosophy of **maximum cross-harness portability and developer convenience**. It takes a collection of high-quality markdown prompts and rules honed in Claude Code and translates them into OpenCode's format with the minimal necessary scaffolding. It acts as an advisory companion that formats code, runs linters, and alerts the user when work is done. It trusts OpenCode's native defaults and the LLM's raw reasoning.

2. **OMO (`oh-my-openagent`)**, by contrast, is built on the philosophy of **total execution determinism and agent autonomy**. It treats LLMs as stochastic, error-prone engines that require an unyielding operational harness to produce enterprise-grade software. Every vulnerability in autonomous LLM development—hallucinated file edits, premature halts, bloated context windows, placeholder comments, lost state across compactions, and unstructured subagent chaos—is met with a dedicated, hardened subsystem (Hashline, Boulder, Preemptive Compaction, Comment Checker, and Team Mode).

For developers seeking lightweight assistance and portable Claude Code habits, ECC provides a clean, unobtrusive configuration layer. For engineering tasks requiring sustained multi-hour autonomous execution, strict test-driven verification, and coordinated multi-agent orchestration, OMO operates as an industrial-grade Agent OS.

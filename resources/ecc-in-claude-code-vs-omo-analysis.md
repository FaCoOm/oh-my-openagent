# Deep Dive: ECC in Claude Code vs. OMO & What ECC Adds to Vanilla Claude Code

**Date:** 2026-09-26  
**Subject:** Comparative Autonomy Analysis & Feature Taxonomy of Everything-Claude-Code (ECC) in Native Claude Code  
**Analyzed Repositories:**
- **ECC (`everything-claude-code`):** `C:\Users\Fate_Conqueror\GitHub\everything-claude-code`
- **OMO (`oh-my-openagent` / `oh-my-opencode`):** `c:\Users\Fate_Conqueror\GitHub\oh-my-openagent`

---

## Executive Summary

1. **Does ECC in Claude Code provide the same deterministic autonomous workflow as OMO?**
   **No, it fundamentally does not.**
   - **OMO's autonomy is hard-enforced and runtime-invasive:** OMO actively hijacks the harness lifecycle. When an agent attempts to stop or yield, OMO's **Boulder State Engine** (`todoContinuationEnforcer`) intercepts the idle event, queries `.omo/boulder-state.json`, vetoes the halt if incomplete tasks exist, and forcibly injects continuation prompts. Coupled with cryptographic line-anchored editing (**Hashline** `LINE#ID`), pre-tool file access blockades, and isolated git worktrees per agent in **Team Mode**, OMO forces determinism at the mechanical runtime level.
   - **ECC's autonomy in Claude Code is soft-guided and advisory:** Anthropic's Claude Code extension API strictly sandboxes plugins. `PreToolUse` hooks can block commands, and `Stop` hooks can run asynchronous scripts, but **Claude Code does not provide a hook mechanism to trap the completion event and force the LLM into an infinite autonomous loop**. ECC's autonomy relies on high-quality prompt instructions (e.g. `/plan`, `/tdd`, `loop-operator`, `santa-loop`), developer discipline, and shell-level quality checks. If Claude hallucinates or yields control prematurely, ECC cannot force it to continue.

2. **What does ECC provide to Claude Code beyond the vanilla built-in version?**
   Vanilla Claude Code is an unopinionated, clean-slate CLI: a bare system prompt, 12 basic tools (`Bash`, `FileEdit`, `Glob`, `Grep`, `Agent`, etc.), zero predefined hooks, zero subagent personas, zero domain skills, and no cross-session learning.
   **ECC transforms vanilla Claude Code into a comprehensive software engineering framework** by adding:
   - **68 specialized agent personas** across architecture, testing, security, and 15+ language ecosystems.
   - **292 domain skills** covering frontend, backend, security, RAG, and AI evals.
   - **94 workflow slash commands** (such as `/plan`, `/tdd`, `/santa-loop`, `/security`, `/e2e`).
   - **Automated hook runtime** (`hooks/hooks.json`): auto-formatting (Prettier/Black), real-time typechecking (`tsc`), pre-commit quality gates, dev server blockers, and desktop alerts.
   - **Continuous learning & instinct system**: extracts patterns and user corrections from sessions, clustering them into permanent project rules.
   - **22 modular rule packs** enforcing immutability, test-driven development, and architectural invariants.

---

## Part 1: Why ECC in Claude Code Cannot Replicate OMO's Deterministic Autonomy

To understand why ECC cannot achieve the same level of deterministic autonomy as OMO, one must compare the **enforcement mechanisms** available in Claude Code versus OpenCode.

### 1. The Autonomous Continuation Loop (Boulder vs. Prompt-Based Loops)

```
OMO Runtime-Enforced Continuation (Deterministic):
┌────────────────────────────────────────────────────────────────────────┐
│ Agent finishes turn → OpenCode fires `session.idle`                   │
│         ↓                                                              │
│ [OMO Hook: todoContinuationEnforcer]                                  │
│   ├── Inspects `.omo/boulder-state.json`                               │
│   ├── Unfinished tasks exist?                                          │
│   │     ├── YES: VETO `session.idle`! Inject continuation prompt.     │
│   │     │        Agent is FORCED to continue. Cannot exit.            │
│   │     └── NO:  Allow session to settle.                              │
└────────────────────────────────────────────────────────────────────────┘

ECC in Claude Code (Advisory / Prompt-Driven):
┌────────────────────────────────────────────────────────────────────────┐
│ Claude finishes turn → Claude Code fires `Stop` event                 │
│         ↓                                                              │
│ [ECC Hook: stop:check-console-log, stop:cost-tracker, etc.]           │
│   ├── Runs async Node scripts (audits logs, desktop alert)            │
│   └── CANNOT veto or resume execution! Turn finishes.                 │
│                                                                        │
│ * Autonomy relies on Claude *choosing* to run another tool or         │
│   user typing another command. If Claude stops, the session halts.    │
└────────────────────────────────────────────────────────────────────────┘
```

- **In OMO:** The **Boulder State Engine** is a hard state machine. An agent operating in Ultrawork (ULW) mode or under an active boulder plan is trapped in a deterministic execution loop. OpenCode's internal event loop is directly patched and intercepted.
- **In Claude Code with ECC:** Claude Code's plugin API only exposes six hook events: `PreToolUse`, `PostToolUse`, `SessionStart`, `SessionEnd`, `PreCompact`, and `Stop`. While a `PreToolUse` hook can return exit code `2` to block an individual tool call, **the `Stop` hook cannot programmatically re-open the generation stream**. ECC commands like `/loop-start` or `/santa-loop` generate runbooks or tell Claude in markdown prompts to keep iterating, but they rely entirely on Claude's self-discipline. If the model hallucinates completion, the harness stops.

### 2. File Editing Integrity (Hashline LINE#ID vs. String Diffing)

- **In OMO:** OMO addresses the single greatest cause of agent failure in large codebases: **hallucinated line numbers and drift during file edits**.
  - OMO's `hashlineReadEnhancer` tags every line in `read` output with a cryptographic line hash (`LINE#ID`, e.g., `124#7b9a`).
  - The `hashline_edit` tool strictly requires the agent to specify exact start/end `LINE#ID` tokens.
  - If a file has drifted or the hash is invalid, the edit fails immediately and safely before disk corruption occurs.
- **In Claude Code with ECC:** Claude Code uses native text replacement (`FileEdit` / `MultiEdit`) which relies on finding unique substring chunks. In large files, repetitive patterns, or complex refactors, Claude frequently mismatches blocks, drops lines, or introduces syntax errors. ECC runs `tsc --noEmit` *after* the damage is done to warn about errors, but cannot prevent the edit collision upfront.

### 3. Pre-Tool Blockades & Anti-Slop Enforcement

- **In OMO:**
  - `writeExistingFileGuard`: Physically prevents `write` or `edit` if the file has not been read in the current session.
  - `bashFileReadGuard`: Physically blocks raw shell reads (`cat`, `head`, `tail`), forcing structured tools.
  - `commentChecker`: Runs a compiled binary (`@code-yeongyu/comment-checker`) that parses AST diffs and rejects edits containing AI placeholder comments (`// implement later`, `/* ... */`).
- **In Claude Code with ECC:**
  - ECC's `PreToolUse` hooks provide valuable checks (blocking `npm run dev` outside tmux, checking staged files on `git commit`, warning about creating unnecessary `.md` files).
  - However, it does not enforce read-before-write invariants or run AST-level comment rejection.

### 4. Parallel Multi-Agent Coordination (Team Mode vs. Subagents)

- **In OMO:** Ships a multi-process, parallel **Team Mode**:
  - Independent git worktrees per agent to avoid merge conflicts.
  - Inter-agent asynchronous mailboxes (`team_send_message`).
  - File-locked shared task board.
  - Automated multi-pane tmux terminal layouts.
  - Formal consensus shutdown protocol (`team_shutdown_request` -> `team_approve_shutdown`).
- **In Claude Code with ECC:** Claude Code has a native `Agent` tool that spawns ephemeral subagents. ECC provides rich persona prompts (e.g. `code-reviewer`, `security-reviewer`), but subagents execute in isolation, cannot message each other, do not have dedicated worktrees, and cannot coordinate via shared lockfiles.

---

## Part 2: Comprehensive Breakdown of What ECC Adds to Vanilla Claude Code

Anthropic's vanilla Claude Code provides the underlying engine: the LLM connection, shell execution, basic file tools, and plugin infrastructure. ECC acts as an **industrial-grade software engineering distribution** on top of that engine.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        ECC (Version 2.2.2)                             │
│                                                                        │
│  ┌───────────────────────┐  ┌───────────────────────┐                  │
│  │ 68 Specialized Agents │  │ 292 Domain Skills     │                  │
│  │ (Architecture, TDD,   │  │ (Patterns, Frameworks,│                  │
│  │  Security, Languages) │  │  RAG, Evals, DB)      │                  │
│  └───────────────────────┘  └───────────────────────┘                  │
│  ┌───────────────────────┐  ┌───────────────────────┐                  │
│  │ 94 Workflow Commands  │  │ Automated Hook Runtime│                  │
│  │ (/plan, /tdd, /santa, │  │ (PreToolUse, PostTool,│                  │
│  │  /security, /e2e)     │  │  PreCompact, Stop)    │                  │
│  └───────────────────────┘  └───────────────────────┘                  │
│  ┌───────────────────────┐  ┌───────────────────────┐                  │
│  │ Continuous Learning   │  │ 22 Framework Rules    │                  │
│  │ (Instincts Engine,    │  │ (Immutability, TDD,   │                  │
│  │  Pattern Extraction)  │  │  Security Baselines)  │                  │
│  └───────────────────────┘  └───────────────────────┘                  │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (mounts into)
┌────────────────────────────────────────────────────────────────────────┐
│                      VANILLA CLAUDE CODE                               │
│  Clean-slate CLI, 12 basic tools (Bash, FileEdit, Glob, Agent...),     │
│  Zero default hooks, Zero agents, Zero skills, Zero rules              │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 1. The 68 Specialized Agent Personas (`agents/`)

Vanilla Claude Code operates with a single general-purpose agent personality. ECC provides **68 specialized agent definitions**, each with precise instructions, role boundaries, and tool recommendations:

#### A. Core Engineering Specialists
- `planner`: Implementation planning, dependency identification, and risk breakdown.
- `architect`: System design, modular decomposition, and scalability evaluations.
- `tdd-guide`: Strict test-driven development specialist enforcing RED-GREEN-REFACTOR and 80%+ test coverage.
- `code-reviewer`: Immediate post-implementation code review for quality, complexity, and maintainability.
- `security-reviewer`: Vulnerability hunter (OWASP Top 10, input sanitization, CSRF, auth flaws).
- `e2e-runner`: Playwright-based end-to-end testing specialist.
- `refactor-cleaner`: Dead code detection, duplicate elimination, and modular cleanup.
- `build-error-resolver`: Focused minimal-diff compiler and build fixer.
- `spec-miner`: Brownfield specification extraction for onboarding legacy projects.

#### B. Language & Framework Specialists
ECC provides pairs of **Reviewers** and **Build Resolvers** across 15+ stacks:
- **Rust:** `rust-reviewer`, `rust-build-resolver` (ownership, lifetimes, cargo).
- **Go:** `go-reviewer`, `go-build-resolver` (idiomatic concurrency, error handling, vet).
- **Python / Django:** `python-reviewer`, `django-reviewer`, `django-build-resolver`.
- **C / C++:** `cpp-reviewer`, `cpp-build-resolver` (memory safety, modern idioms, CMake).
- **Java / Kotlin:** `java-reviewer`, `java-build-resolver`, `kotlin-reviewer`, `kotlin-build-resolver`.
- **Modern Web:** `typescript-reviewer`, `react-native`, `database-reviewer` (Postgres/Supabase).
- **AI / ML:** `mle-reviewer` (production ML pipelines), `rag-pipeline-reviewer` (RAGAS evals, chunking), `pytorch-build-resolver`.

---

### 2. The 292 Domain Skills Library (`skills/`)

While vanilla Claude Code knows only what is present in its training weights, ECC equips Claude with **292 structured SKILL.md guides** that load into context when relevant:
- **Architecture & Design:** `api-design`, `backend-patterns`, `frontend-patterns`, `microservices`, `event-sourcing`.
- **Testing & Quality:** `tdd-workflow`, `e2e-testing`, `verification-loop`, `eval-harness`, `test-coverage`.
- **Security Hardening:** `security-review`, `auth-best-practices`, `secrets-management`, `input-validation`.
- **Operational & Context Management:** `strategic-compact` (suggests manual compaction every ~50 tool calls to prevent context degradation), `loop-operator`, `harness-optimizer`.
- **Business & Domain:** `market-research`, `investor-outreach`, `technical-writing`.

---

### 3. The 94 Workflow Slash Commands (`commands/`)

Vanilla Claude Code supports custom slash commands, but ships with none. ECC introduces **94 pre-engineered slash commands**:

| Command | Purpose |
| :--- | :--- |
| `/plan` | Creates a comprehensive implementation plan before touching code. |
| `/tdd` | Forces a strict RED-GREEN-REFACTOR development loop. |
| `/santa-loop` | Adversarial dual-review convergence loop: spawns Claude Opus + an external model (Codex or Gemini CLI via bash); both must return `NICE` before code can ship. |
| `/security` | Executes a complete vulnerability audit across modified files. |
| `/code-review` | Dispatches the `code-reviewer` agent to review unstaged diffs. |
| `/build-fix` | Invokes targeted build resolvers with minimal diff mandates. |
| `/e2e` | Generates and executes Playwright tests against user flows. |
| `/refactor-clean` | Scans for unused variables, dead functions, and redundant imports. |
| `/checkpoint` | Saves current verification state and progress to `.claude/plans/`. |
| `/eval` | Runs formal evaluation against predefined criteria. |
| `/instinct-*` | Commands to view, import, export, and evolve learned instincts (`/instinct-status`, `/evolve`, `/promote`). |

---

### 4. The Automated Hook Runtime (`hooks/hooks.json`)

Vanilla Claude Code has hook extension points in `settings.json`, but requires the user to write their own scripts. ECC bundles a complete **Node-based hook automation runtime**:

```
ECC Hook Lifecycle in Claude Code:
├── PreToolUse
│   ├── Dev server blocker      (Blocks `npm run dev` etc. outside tmux; exit code 2)
│   ├── Pre-commit quality gate (Lints staged files, validates commit msg, blocks secrets)
│   ├── Doc file warning        (Warns against creating random unnecessary .md files)
│   ├── Tmux reminder           (Suggests tmux for long test/build commands)
│   └── Strategic compact       (Reminds user to /compact every ~50 tool calls)
│
├── PostToolUse
│   ├── Prettier format         (Auto-runs `prettier --write` immediately after editing JS/TS)
│   ├── TypeScript check        (Auto-runs `tsc --noEmit` immediately after editing .ts/.tsx)
│   ├── console.log warning     (Warns if debug logging statements were added)
│   ├── Quality gate            (Runs quick linter/validator after edits)
│   └── PR logger               (Logs PR URL and review commands after `gh pr create`)
│
├── PreCompact
│   └── Memory persistence      (Saves session snapshot before context compaction wipes memory)
│
├── Stop
│   ├── Console.log audit       (Scans all modified files for lingering debug logs)
│   ├── Desktop notify          (Sends desktop notification via macOS osascript, Win PowerShell, Linux)
│   ├── Cost tracker            (Emits lightweight token and cost metrics)
│   └── Pattern extraction      (Evaluates session for continuous learning instincts)
│
└── SessionStart / SessionEnd
    ├── Detects project package managers, lockfiles, and toolchains
    └── Cleans up session temporary state
```

---

### 5. Continuous Learning & Instinct System (`continuous-learning-v2`)

In vanilla Claude Code, sessions are completely amnesic unless the developer manually updates `CLAUDE.md`.
ECC implements an **autonomous learning loop**:
1. **Observation:** Hooks (`pre:observe:continuous-learning`, `post:observe:continuous-learning`) track tool calls and user corrections.
2. **Evaluation:** When a session completes (`Stop` hook), ECC evaluates the trajectory for extracted patterns.
3. **Clustering & Promotion:** Extracted patterns are stored as local "instincts".
4. **Graduation:** Using `/evolve` and `/promote`, instincts that prove consistently beneficial across multiple sessions can be promoted into permanent project rules in `.claude/rules/`.

---

### 6. The 22 Modular Rule Packs (`rules/`)

Vanilla Claude Code relies on whatever the user puts in `CLAUDE.md`. ECC provides **22 battle-tested rule sets** for common ecosystems:
- `common/`: Immutability principles, input validation, small file sizes (<400 lines typical, 800 max), strict error handling.
- Frameworks: `typescript/`, `python/`, `rust/`, `golang/`, `react/`, `vue/`, `angular/`, `django/`, `swift/`, `cpp/`, etc.

---

## Part 3: Direct Feature Comparison Matrix

| Capability / Feature | Vanilla Claude Code | Claude Code + ECC | OMO (`oh-my-openagent`) |
| :--- | :--- | :--- | :--- |
| **Agent Personas** | 1 generic system prompt | 68 specialized domain agents | 11 dynamic personas with factory prompt engines |
| **Skills Library** | None built-in | 292 curated domain skills | Category skills + dynamically mounted skills |
| **Slash Commands** | None built-in | 94 workflow commands | Dynamic slash command injector + auto-execute |
| **Autonomous Continuation** | **No** (halts on completion) | **No** (prompt-guided only) | **Yes** (Boulder state machine vetoes halt) |
| **File Edit Safety** | Substring replace (`FileEdit`) | Substring replace + post-tsc | Cryptographic `LINE#ID` Hashline editing |
| **Automated Formatting** | None | Prettier / Biome / Black hooks | Hashline enhancer + fsync verification |
| **Pre-Commit Checks** | None | Secrets, lint, commit msg hook | ULW verification loop + evidence audit |
| **Comment Slop Prevention**| None | None | Compiled `@code-yeongyu/comment-checker` binary |
| **Multi-Agent Teams** | Basic sequential subagents | Basic sequential subagents | Full parallel Team Mode (worktrees, tmux, mailboxes) |
| **Compaction Recovery** | Unstructured summary | Snapshot saved in `PreCompact` | Multi-hook state re-injection + auto-continue |
| **Learning / Instincts** | Manual `CLAUDE.md` editing | Automated pattern extraction | Boulder state + rules engine persistence |
| **Language Server (LSP)** | None | CLI wrappers via bash | Stdio Tier-1 MCP (`lsp-tools-mcp`, 8 tools) |

---

## Conclusion & Practical Verdict

1. **If your goal is deterministic autonomous completion:**
   ECC running in Claude Code **will not** provide the same hard determinism that OMO delivers in OpenCode. Claude Code's extension architecture does not permit plugins to veto the `Stop` event or trap the agent in an unyielding state machine. OMO's **Boulder State Engine**, **Hashline LINE#ID editing**, and **Team Mode** remain unique to OMO's deep harness integration.

2. **What ECC provides to Claude Code:**
   ECC is not trying to be a rogue runtime controller; it is an **encyclopedic software engineering operating standard**. It elevates vanilla Claude Code from an unguided, reactive coding assistant into a structured, disciplined development environment equipped with 68 agents, 292 skills, 94 commands, automated quality hooks, and cross-session continuous learning.

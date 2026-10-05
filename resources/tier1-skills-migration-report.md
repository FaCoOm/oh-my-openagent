# Tier 1 Skills Migration & Deployment Report: Cross-Harness & Antigravity (AGY)

> **Document Status:** Complete & Verified  
> **Execution Date:** October 2026  
> **Source Packages:** `packages/shared-skills/skills/` and `.agents/skills/`  
> **Deployment Targets:**  
> 1. **Antigravity Global Configuration:** `C:\Users\Fate_Conqueror\.gemini\config\skills\`  
> 2. **Repository Workspace Root:** `.agents/skills\`  

---

## 1. Executive Summary

In accordance with the Cross-Harness Skill Adoption Strategy, the **Tier 1 Skills**—the foundational, high-impact, pure-procedure skills that require zero-to-low adaptation—have been systematically migrated, tuned, and deployed.

Every migrated skill has been equipped with a standardized **Cross-Harness Tool Compatibility & Execution Guide**, providing runtime translation between:
- **Google Antigravity (AGY)**
- **OpenCode (OMO Ultimate)**
- **Codex CLI (OMO Light)**
- **Claude Code / Generic LLM runners**

Both global Antigravity installations and repository-level multi-harness runtimes now have immediate access to all 9 core capabilities.

---

## 2. Inventory of Migrated Tier 1 Skills

| # | Skill Name | Primary Responsibility | Included Assets | Global AGY Target | Repo `.agents/` Target |
|---|---|---|---|---|---|
| 1 | **`remove-ai-slops`** | 9-category purge of AI boilerplate, obvious comments, over-defensive shims, and dead wrappers behind regression tests. | `SKILL.md` | `~/.gemini/config/skills/remove-ai-slops/` | `.agents/skills/remove-ai-slops/` |
| 2 | **`init-deep`** | Automated discovery of codebase structure via symbols and generation of hierarchical `AGENTS.md` context maps. | `SKILL.md` | `~/.gemini/config/skills/init-deep/` | `.agents/skills/init-deep/` |
| 3 | **`tech-debt-audit`** | 9-dimension health check (circular dependencies, type safety, dead code, architectural rot) emitting `TECH_DEBT_AUDIT.md`. | `SKILL.md` | `~/.gemini/config/skills/tech-debt-audit/` | `.agents/skills/tech-debt-audit/` |
| 4 | **`ulw-plan`** | Structured planning producing checklist tasks (`T1`, `T2`) and explicit verification waves (`F1`). | `SKILL.md`, `references/`, `scripts/`, `agents/` | `~/.gemini/config/skills/ulw-plan/` | `.agents/skills/ulw-plan/` |
| 5 | **`frontend`** | Curated design tokens, style gallery routing, and responsive layout architectures (`taste-skill`, `ui-ux-pro-max`). | `SKILL.md`, `references/{design,designpowers,perfection}` | `~/.gemini/config/skills/frontend/` | `.agents/skills/frontend/` |
| 6 | **`visual-qa`** | Dual-oracle visual verification across web and TUI: Apple HIG checklist, mobile/desktop viewports, and screenshot diffs. | `SKILL.md`, `references/`, `scripts/` | `~/.gemini/config/skills/visual-qa/` | `.agents/skills/visual-qa/` |
| 7 | **`git-master`** | Conservative Git history management: atomic commit staging, rebase, squash, blame, bisect, and reflog. | `SKILL.md` | `~/.gemini/config/skills/git-master/` | `.agents/skills/git-master/` |
| 8 | **`debugging`** | Hypothesis-driven debugging loop across Python, Node, Rust, Go, native binaries, DAP, and browser QA. | `SKILL.md`, `references/{methodology,runtimes,scripts,tools}` | `~/.gemini/config/skills/debugging/` | `.agents/skills/debugging/` |
| 9 | **`refactor`** | Behavior-preserving transformation: lock semantics with tests first, step-by-step refactoring, and continuous verification. | `SKILL.md` | `~/.gemini/config/skills/refactor/` | `.agents/skills/refactor/` |

---

## 3. The Cross-Harness Adaptation Layer

Every migrated `SKILL.md` now embeds the canonical translation matrix right after its YAML frontmatter:

```markdown
## Cross-Harness Tool Compatibility & Execution Guide

When executing this skill across different agent harnesses, translate workflow operations into the active environment's native primitives:

| Workflow Operation | OpenCode (OMO) | Google Antigravity (AGY) | Codex CLI (OMO Light) | Claude Code / Generic |
|---|---|---|---|---|
| **Spawn Subagent** | `call_omo_agent(...)` or `task(...)` | `invoke_subagent(Role, Prompt, Model)` | `multi_agent_v1.spawn_agent(...)` | Subagent tool / background CLI |
| **Define Persona** | `agentSources` / config | `define_subagent(name, prompt)` | `agents/openai.yaml` roles | System prompt prepend |
| **Background Task**| `task(run_in_background=true)` | `run_command(WaitMsBeforeAsync)` | `spawn_agent` + async wait | `nohup` / `&` background execution |
| **File Read** | `read` / `file_read` | `view_file(AbsolutePath, Start, End)` | `read_file` | Read tool / cat |
| **File Edit** | `edit` (hashline) / `apply_patch`| `replace_file_content` / `write_to_file` | `apply_patch` / `file_edit` | Edit tool / patch |
| **Code Intelligence**| `lsp_*` aliases / `grep` | `codegraph_*` MCP / `grep` | `lsp_*` / `grep` | grep / ripgrep |
| **Browser Action** | `browser` (`omowright`) | `agent-browser` MCP / `read_url_content`| `browser` tool | Playwright / curl |
| **Deliverables** | `.omo/plans/`, `.omo/evidence/` | `resources/` and brain artifacts | `.omo/evidence/` | Repo root / docs |
| **User Interaction**| First-turn chat prompt | `ask_question(...)` interactive modal | TUI stdin | CLI prompt / question tool |
```

---

## 4. Verification & Audit Results

### 4.1 Global Antigravity Installation (`~/.gemini/config/skills/`)
All 9 skill directories exist and contain valid `SKILL.md` documents along with all required subdirectories:
- `remove-ai-slops`: Verified (357 lines, 23KB)
- `init-deep`: Verified (324 lines, 13KB)
- `tech-debt-audit`: Verified (227 lines, 11KB)
- `ulw-plan`: Verified (Includes `references/`, `scripts/`, `agents/`)
- `frontend`: Verified (Includes `references/design`, `references/designpowers`, `references/perfection`)
- `visual-qa`: Verified (Includes `references/`, `scripts/`)
- `git-master`: Verified (124 lines, 7KB)
- `debugging`: Verified (Includes `references/methodology`, `references/runtimes`, `references/tools`, `references/scripts`)
- `refactor`: Verified (775 lines, 27KB)

### 4.2 Workspace Deployment (`.agents/skills/`)
- Verified all 9 directories populated.
- `.agents/skills/tech-debt-audit/SKILL.md` updated in-place with compatibility guide.
- Git status confirms clean addition of 8 directories and 1 modified file without modifying core plugin packages.

---

## 5. Usage Guide: Activating Migrated Skills in AGY

Because Antigravity leverages **Progressive Disclosure**, these skills do not consume context tokens until activated. You or the agent can invoke them anytime:

1. **AI-Slop Removal:**
   - Prompt: *"Please deslop our recent changes and remove obvious comments using the remove-ai-slops skill."*
2. **Hierarchical Codebase Mapping:**
   - Prompt: *"Run init-deep to inspect our project structure and generate AGENTS.md documentation."*
3. **Structured Work Planning:**
   - Prompt: *"Activate ulw-plan to draft an implementation plan for the new authentication module."*
4. **Visual & UI Quality Assurance:**
   - Prompt: *"Use visual-qa to review the rendered landing page across mobile (390px) and desktop (1440px)."*
5. **Technical Debt Audit:**
   - Prompt: *"Run tech-debt-audit to scan for circular dependencies, dead code, and type safety issues."*
6. **Hypothesis-Driven Debugging:**
   - Prompt: *"Use the debugging skill to trace the silent memory leak in our WebSocket handler."*

---

## 6. Next Steps

With Tier 1 complete:
- **Tier 2 Workflows:** Prepare `hyperplan` (5-persona adversarial debate), `work-with-pr` (worktree-isolated PR loop), and `ulw-research` (maximum saturation claim graphs) for their AGY-native subagent and MCP mappings.

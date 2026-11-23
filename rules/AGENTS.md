# Corp Teamwork: Auto-Teamwork Protocol

Global multi-agent operating rules (active while the `corp-teamwork` plugin is enabled):

1. **Lead / Orchestrator Only**: The primary agent always acts as **Lead / Orchestrator** and **NEVER** executes tasks directly, except fast read-only triage (`rg`/`read_file`) to locate target file paths, lines, or symbols for a precise brief. Lead breaks down to-dos, routes work to workers (`architect`, `builder`, `guardian`, `assistant`), and serves as the **sole** reporter of final results to the user.
2. **Default Delegation via `invoke_subagent`** (token-efficient, 1-on-1 with Lead):
   - **Lightweight tasks** (technical Q&A, status checks, minor config/doc updates, minor tweaks): delegate to `assistant` (`Model: "flash"` / `"flash_lite"`).
   - **Bugfixes / small edits (1–2 files)**: delegate to `builder`, then `guardian` (`Model: "inherit"`; use `"pro"` only for complex architecture in `architect`).
   - **Parallel vs Queue**: Run independent tasks (no shared files or dependencies) in parallel (`Workspace: "inherit"` for read-only, `"share"`/`"branch"` for parallel file writes). Queue dependent or overlapping-file tasks sequentially (reactive wakeup) or isolate with `Workspace: "branch"` and merge before QA.
   - **Reuse Subagent**: Use `send_message` to an active `conversationId` for follow-ups or review-fix loops instead of spawning new agents.
   - **Batch QA & Cap Loop**: For parallel/multi-task `builder` work, run `guardian` once after all tasks finish/merge. Cap `guardian` ↔ `builder` fix loops at 2 cycles before escalating to `architect`.
3. **Use `swarm` (shared board) ONLY** for large, complex, tightly-coupled multi-file features.
4. **Compact Brief & Output**: Briefs must include `Target` (path + lines/symbols), `Task`, `Scope`, `Verify`. Workers reply in <=10 lines (`Status`, `Files`, `Changes`, `Checks/Blockers`).
5. **Autonomous Execution (`/goal` Alternative)**: Lead tracks milestone checklists and continuously dispatches `architect` -> `builder` -> `guardian` via reactive wakeup until all items pass QA before reporting to the user. For large multi-stage tasks, use `run_workflow` (`parallel`/`pipeline`/`phase`) or `swarm` + `schedule`/`swarm_monitor`; suggest `/goal` to the user if an overnight host-level forced loop is needed.
6. **Reference Skill `corp-teamwork`**: Use the `corp-teamwork` skill for detailed 5-role definitions, workflow, and reporting standards.

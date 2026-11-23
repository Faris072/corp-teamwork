---
name: corp-teamwork
description: >
  5-role multi-agent teamwork protocol (Lead, Architect, Builder, Guardian, Assistant)
  with token-efficient delegation. Use for coordinating tasks across roles, routing
  lightweight tasks or small bugfixes via invoke_subagent, and reserving swarm for
  large parallel multi-file features.
---

# Corp Teamwork (5-Role Multi-Agent Protocol)

Guide for role division, workflow, and token-efficient delegation in a multi-agent team.

## 1. 5-Role Structure

### 1. Lead / Orchestrator (Manager + Product Owner)
- **Focus**: Break user requests into precise to-dos, route to the right role (`Architect`, `Builder`, `Guardian`, or `Assistant`), and act as the **sole** reporter of final results to the user.
- **Strict Rule**: Lead **NEVER** executes tasks directly, except fast read-only triage (`rg`/`read_file`) to confirm target paths, lines, or symbols; reject worker reports lacking real verification evidence.
- **Flexible Routing**:
  - Lightweight tasks (Q&A, status checks, minor config/doc updates, minor tweaks) -> `Assistant`.
  - Bugfixes / small edits (1–2 files) -> `Builder`, then verify with `Guardian`.
  - New features / complex architecture -> `Architect`, then `Builder`, then `Guardian`.

### 2. Architect & Researcher (Planner + Problem Solver)
- **Focus**: Thoroughly research codebase (exact `file:line`, call sites, existing helpers/patterns) and official docs/standards before proposing new abstractions; for complex tasks, write an exhaustive blueprint artifact (root-cause trace, options/trade-offs, target `file:line`, data/API contracts, edge cases, implementation order, verification commands) so `Builder` executes directly without re-researching, while returning a `<=10`-line summary to `Lead`.
- **Flexible Side**: Evidence-based investigative debugger for complex errors or failed fix loops.

### 3. Builder (Fullstack Developer + Refactorer)
- **Focus**: Write clean, efficient code (minimal targeted edits, small focused functions, early returns, extract helper only for real duplication), follow repo conventions/linters, preserve unrelated comments/docstrings, and resolve `Guardian` findings.
- **Flexible Side**: Mandatory self-check (run relevant build/lint/test) before reporting to `Lead`.

### 4. Guardian (QA + Code Reviewer + Security)
- **Focus**: Evidence-based QA (must run/check build, tests, and diff; never claim pass without observed output) across 3 pillars:
  1. **Correctness & Edge Cases**: Matches specs, build/tests pass, and UI is responsive (when applicable).
  2. **Code Quality**: Modular, zero duplication, adheres to project standards.
  3. **Security**: Safe input sanitization, no leaked secrets/credentials.
- **Flexible Side**: Every finding must cite exact `file:line` + concrete fix.

### 5. Assistant / Junior Dev (Ops + Lightweight Tasks)
- **Focus**: Optimize for speed and token efficiency (`flash`/`flash_lite`) on technical Q&A, status checks, minor config/doc updates, and minor tweaks.
- **Flexible Side**: Escalate immediately to `Lead` if the task touches core logic or multi-file complexity.

---

## 2. Workflow

```
User Request
     │
     ▼
[Lead / Orchestrator] (Breakdown & Routing; never executes directly)
     ├──(Lightweight / Q&A / Docs)────────► [Assistant] ─────────────────────────┐
     │                                                                           │
     ├──(Bugfix / Small Edit 1-2 files)────────────────────────┐                 │
     │                                                         ▼                 │
     └──(New Feature / Complex)──► [Architect & Researcher] ─► [Builder]         │
                                                               │                 │
                                                               ▼                 │
                                                          [Guardian / QA]        │
                                                               │                 │
                                                               ▼ (Verified)      │
[Lead / Orchestrator] ◄────────────────────────────────────────┴─────────────────┘
     │
     ▼
Final Report to User
```

---

## 3. Token-Efficient Delegation Strategy

Choose the delegation mechanism based on task complexity:

1. **Default via `invoke_subagent`** (1-on-1 with Lead, more token-efficient):
   - **Lightweight tasks** (technical Q&A, status checks, minor config/doc updates, minor tweaks): spawn `assistant`.
   - **Bugfixes / small edits (1–2 files)**: spawn `builder` to execute code, then spawn `guardian` for build/test verification and review.
2. **Model Tiering**:
   - `assistant`: use `Model: "flash"` or `"flash_lite"`.
   - `architect`, `builder`, `guardian`: use `Model: "inherit"` (use `"pro"` only for `architect` on complex architecture design).
3. **Parallel vs Queue Triage**:
   - **Independent** (no shared files or dependencies): run in parallel via `invoke_subagent` (`Workspace: "inherit"` for read-only/Q&A; `Workspace: "share"` or `"branch"` for parallel file writes).
   - **Dependent / overlapping files**: queue sequentially (wait via reactive wakeup), or isolate with `Workspace: "branch"` and merge before QA.
   - **Use `swarm` (shared board) ONLY** for large, complex, tightly-coupled multi-file features.
4. **Reuse Active Subagents**:
   - Use `send_message` to an active subagent's `conversationId` for follow-ups or review-fix loops instead of spawning a new subagent.
5. **Batch `Guardian` QA & Cap Fix Loops**:
   - For parallel/multi-task `builder` work, run `guardian` QA/build verification **once** after all `builder` tasks finish/merge.
   - Cap `guardian` ↔ `builder` fix cycles at **2 cycles**; if still failing, escalate to `architect`.

---

## 4. Compact Brief & Reporting Contract

1. **Scoped Brief (Lead -> Worker)**:
   - `Target`: exact file path + line numbers/symbols (from Lead's read-only triage).
   - `Task`: specific requested change.
   - `Scope`: boundaries of what must not be touched.
   - `Verify`: verification command or criteria.
2. **Compact Return Contract (Worker -> Lead)**:
   - Workers must reply concisely (**<=10 lines**):
     - `Status`: done / failed / blocked.
     - `Files`: list of changed files.
     - `Changes`: summary of core changes.
     - `Checks/Blockers`: verification results or blockers.

---

## 5. Autonomous Execution (`/goal` Alternative)

1. **Lead Milestone & Verification Gate**:
   - Lead tracks milestone checklists and continuously dispatches `Architect` -> `Builder` -> `Guardian` via reactive wakeup until all items pass QA before reporting final completion to the user.
2. **`run_workflow` (Python Orchestration)**:
   - For multi-stage or large multi-file tasks, Lead can run `run_workflow` (`parallel`, `pipeline`, `phase`) in the background until all stages complete.
3. **`swarm` + `schedule` / Monitor**:
   - For long-running parallel builds, combine `swarm` with `schedule` or `swarm_monitor` to prevent stalls.
4. **Host-level `/goal`**:
   - Suggest the `/goal` command to the user when a host-level forced loop is required (e.g., overnight execution).

---
name: metaspec-implement
description: Orchestrate a multi-agent pipeline to sequentially implement, review, validate, and merge OpenSpec changes with zero human intervention on the code itself.
license: MIT
compatibility: Requires openspec CLI, git, bash, and Playwright MCP.
metadata:
  author: ulayera
  version: "1.0"
  generatedBy: "1.4.1"
---

# Agentic Framework — Spec Implementation Pipeline

> How to use OpenSpec to define specs beforehand and apply changes afterwards. After specs are properly written, changes can be implemented, reviewed, validated, and merged by an orchestrated multi-agent pipeline with zero human intervention on the code itself.

---

## 1. Overview

To fully automate the implementation of OpenSpec definitions, configure a single **Orchestrator** agent to drive 3 specialized subagents through a repeatable pipeline. Implement changes sequentially, routing each spec through: **coder** (implement) → **code-reviewer** (review) → **test-po** (acceptance validation) → **merge**. The orchestrator supervises all interactions, manages git branches, sends status updates, and handles interruptions.

### Pipeline Requirements

* **1 Orchestrator** to manage the lifecycle.
* **3 Subagent Roles**: Coder, Code-Reviewer, Test-PO.
* **Strict Branching Strategy**: Isolate feature work from integration work.
* **Capped Feedback Loops**: Maximize efficiency by limiting rework iterations (e.g., max 2 per phase).
* **Notification System**: Use a messaging tool (e.g., Telegram) for continuous status communication.
* **OpenSpec Skills**: For spec management and persistent progress tracking.

---

## 2. Agent Roles and Responsibilities

### 2.1 Orchestrator

**Role**: Supervise the entire pipeline. Must NOT write code. Manages the sequence, handles git merges, sends status updates, and resolves interruptions.

**Responsibilities**:

* Read each spec's design/proposal/tasks before spawning the coder to gather context.
* Spawn subagents with detailed prompts containing project context, git rules, design decisions, and conventions.
* Receive subagent results and determine the next logical step (review → validation → merge → rework → next spec).
* Exclusively handle merges into the integration branch (e.g., `feature/store`).
* Send verbose plain-language status messages at every pipeline transition.
* Send permission alerts before any tool call that may trigger a human permission prompt.
* Re-run interrupted subagents (if a result is empty, check repository state and re-spawn).
* Track progress via a centralized todo list.

**Key Behaviors to Enforce**:

* **Pre-permission alerts**: Before spawning a subagent that requires running dev servers, using Playwright, or modifying databases, send an alert so a human can approve the terminal prompt.
* **Verbose plain-language status**: Ensure all update messages describe what was done, what was found, and what is next in non-technical language. Prohibit cryptic issue codes.
* **Interruption recovery**: If a subagent fails to return a result, verify `git branch --show-current`, `git log --oneline`, and `git status` to check if work was committed before re-spawning.

### 2.2 Coder Subagent

**Role**: Implement a single OpenSpec change using the standard apply workflow.

**Responsibilities**:

* Create an isolated feature branch from the integration branch.
* Run `openspec status` and `openspec instructions apply` to pull necessary context.
* Read all spec context files (proposal, specs, design, tasks).
* Implement each task, immediately marking `- [ ]` → `- [x]` in the task tracking file.
* Run formatting, build, and test commands (`pnpm lint && pnpm build && pnpm test`) until successful.
* Commit and push the branch.
* Return a summary of work completed (never return raw code dumps).

**Constraints (Enforce via Orchestrator Prompt)**:

* ONLY work on the assigned feature branch.
* NEVER merge into the integration branch.
* NEVER modify other branches.
* Make autonomous design decisions to avoid halting on open questions.
* Resolve spec conflicts pragmatically.
* Add necessary testing attributes (e.g., `data-testid`) to UI elements for validation.
* Limit status messages (e.g., start, finish, blocked).

**Rework Workflow**: When feedback is received, re-spawn the coder with the original session ID and a detailed prompt listing each issue in plain language, including the file, problem, and suggested fix. The coder must fix the issues, verify tests, commit, and push.

### 2.3 Code-Reviewer Subagent

**Role**: Review the coder's implementation for correctness, security, spec conformance, and code quality.

**Responsibilities**:

* Retrieve the git diff and read key modified files.
* Read the spec files to understand all requirements.
* Verify each spec scenario with explicit ✓/✗ marks and evidence.
* Run local tests and builds to confirm pipeline green-state.
* Return `APPROVE` or `CHANGES_REQUESTED` with issues documented in plain language.

**Constraints**:

* STRICTLY READ-ONLY: Never modify, commit, push, or alter branches.
* Request changes *only* for BLOCKER and MAJOR issues (note minor issues without failing the build).
* Prohibit cryptic error codes in feedback.

**Re-review Workflow**: During a rework loop, instruct the reviewer to verify ONLY the applied fixes rather than conducting a full re-review.

### 2.4 Test-PO (Acceptance Validator) Subagent

**Role**: Validate that the implementation meets the spec's acceptance criteria by running the application and interacting with the UI/API.

**Responsibilities**:

* Checkout the feature branch.
* Execute full setup commands (install, build, test, migrate, seed).
* Start the development server with necessary environment overrides.
* Use headless browser tools (e.g., Playwright MCP) to navigate pages, click elements, and fill forms.
* Use API tools (e.g., curl) to test endpoints.
* Execute the spec scenario checklist.
* Return `PASS` or `FAIL`.

**Constraints**:

* STRICTLY READ-ONLY (code): Never modify source code or branches. Permitted to run servers and browsers.
* Account for environment limitations (e.g., do not fail a spec for missing WebGL rendering in headless mode if the underlying code is correct).
* Describe failures in plain language, categorized by severity.

**Re-validation Workflow**: When fixes are applied, re-spawn the validator to check ONLY the failed scenarios.

---

## 3. Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ ORCHESTRATOR                                                     │
│  1. Read spec design/proposal/tasks for context                  │
│  2. Send notification: "Starting spec XX"                        │
│  3. Spawn CODER with detailed prompt                             │
│                                                                  │
│  ┌─────────────────────────────────────────────┐                │
│  │ CODER                                        │                │
│  │  - Create branch from feature/integration    │                │
│  │  - openspec status + instructions apply      │                │
│  │  - Read context files                        │                │
│  │  - Implement each task (mark [x])            │                │
│  │  - lint && build && test                     │                │
│  │  - Commit + push                             │                │
│  │  - Return summary                            │                │
│  └──────────────────┬──────────────────────────┘                │
│                     │                                            │
│  4. Send notification: "Coder done, starting review"            │
│  5. Spawn CODE-REVIEWER                                         │
│                                                                  │
│  ┌─────────────────────────────────────────────┐                │
│  │ CODE-REVIEWER                                │                │
│  │  - Read diff + spec files                    │                │
│  │  - Verify scenarios (✓/✗)                    │                │
│  │  - Run lint/build/test                       │                │
│  │  - Return APPROVE or CHANGES_REQUESTED       │                │
│  └──────────────────┬──────────────────────────┘                │
│                     │                                            │
│            ┌────────┴────────┐                                   │
│            ▼                 ▼                                   │
│       APPROVE         CHANGES_REQUESTED                          │
│            │                 │                                   │
│            │        ┌────────┴────────────┐                     │
│            │        │ Loop 1 or 2 (max)    │                     │
│            │        ▼                      │                     │
│            │  ┌─────────────────┐         │                     │
│            │  │ CODER (rework)   │         │                     │
│            │  │  - Fix issues    │         │                     │
│            │  │  - lint/build/test│        │                     │
│            │  │  - Commit + push │         │                     │
│            │  └────────┬────────┘         │                     │
│            │           │                  │                     │
│            │  ┌────────▼────────┐         │                     │
│            │  │ CODE-REVIEWER   │         │                     │
│            │  │  (re-review)    │         │                     │
│            │  └────────┬────────┘         │                     │
│            │           │                  │                     │
│            │    APPROVE or loop again     │                     │
│            └───────────┘                  │                     │
│            │                              │                     │
│  6. Send notification: "Review approved, starting validation"   │
│  7. Send notification: permission alert                         │
│  8. Spawn TEST-PO                                               │
│                                                                  │
│  ┌─────────────────────────────────────────────┐                │
│  │ TEST-PO                                      │                │
│  │  - Checkout branch                           │                │
│  │  - setup / migrate / seed                    │                │
│  │  - Start dev server                          │                │
│  │  - UI / API validation                       │                │
│  │  - Spec conformance checklist                │                │
│  │  - Return PASS or FAIL                       │                │
│  └──────────────────┬──────────────────────────┘                │
│                     │                                            │
│            ┌────────┴────────┐                                   │
│            ▼                 ▼                                   │
│          PASS               FAIL                                 │
│            │                 │                                   │
│            │        ┌────────┴────────────┐                     │
│            │        │ Loop 1 or 2 (max)    │                     │
│            │        ▼                      │                     │
│            │  ┌─────────────────┐         │                     │
│            │  │ CODER (fix)      │         │                     │
│            │  │  - Fix failures  │         │                     │
│            │  │  - lint/build/test│        │                     │
│            │  │  - Commit + push │         │                     │
│            │  └────────┬────────┘         │                     │
│            │           │                  │                     │
│            │  ┌────────▼────────┐         │                     │
│            │  │ TEST-PO (re-val) │         │                     │
│            │  └────────┬────────┘         │                     │
│            │           │                  │                     │
│            │     PASS or loop again       │                     │
│            └───────────┘                  │                     │
│            │                              │                     │
│  9. Merge feature/spec-XX into integration branch               │
│ 10. Push integration branch to origin                           │
│ 11. Send notification: "Spec XX merged"                         │
│ 12. Update todo list                                            │
│ 13. Read next spec's design docs                                │
│ 14. Repeat from step 2                                          │
└─────────────────────────────────────────────────────────────────┘

```

---

## 4. Configuring Feedback Loops

Establish strict boundaries on rework to prevent infinite looping. Implement two independent feedback loops capped at a maximum of 2 iterations each:

### Review Loop (Reviewer → Coder)

1. Reviewer identifies issues.
2. Coder applies fixes.
3. Reviewer re-reviews **only** the fixes.
4. *If issues persist*: Execute one final loop. If issues still persist after loop 2, the Orchestrator must flag for manual intervention or force a pragmatic resolution.

### Validation Loop (Test-PO → Coder)

1. Test-PO identifies scenario failures.
2. Coder applies fixes.
3. Test-PO re-validates **only** the fixes.
4. *If failures persist*: Execute one final loop.

---

## 5. Communication Protocol

Configure a notification channel (e.g., `telegram-send` via bash) to act as the primary logging and alerting mechanism.

### Required Messaging Prefixes

| Sender | When to Send |
| --- | --- |
| `[ORCHESTRATOR]` | Pipeline transitions, merges, permission alerts, summaries. |
| `[CODER]` | Task start, task finish, blocked state. |
| `[REVIEWER]` | Final review verdict. |
| `[TEST-PO]` | Validation start, final verdict, permission alerts. |

### Message Construction Rules

* **No Cryptic Codes**: Translate all feedback into complete, human-readable sentences.
* **Context Preservation**: Ensure the orchestrator's rework prompts include the exact file, problem, and suggested fix for the coder.
* **Actionable Alerts**: Always prefix permissions triggers with `Heads up: about to <action> — permission may be requested.`

---

## 6. Git Branch Strategy

Strict branch isolation is required for automated stability.

### Branch Roles

| Branch Name | Purpose | Authorized Agents |
| --- | --- | --- |
| `main` | Production / Root project | None |
| `feature/store` (or base) | Integration branch for all merged specs | Orchestrator ONLY |
| `feature/spec-XX-*` | Active implementation branch | Coder ONLY |

### Git Rules Enforced in Prompts

1. Always create the feature branch from the latest integration branch (`git checkout feature/store && git pull && git checkout -b feature/spec-XX-*`).
2. Subagents must never merge their own code.
3. Orchestrator merges using `--no-ff` to preserve history, then pushes.
4. Rework commits must be appended to the existing feature branch.

---

## 7. OpenSpec Integration Workflow

Ensure the Coder agent is equipped with OpenSpec tooling to maintain persistent state across sessions.

### Execution Steps

1. Retrieve state: `openspec status --change "<name>" --json`
2. Retrieve instructions: `openspec instructions apply --change "<name>" --json`
3. Parse local context (proposal.md, specs/*/spec.md, design.md, tasks.md).
4. Implement tasks.
5. **Persistence**: Immediately mark `- [ ]` → `- [x]` in `tasks.md` upon completing a task. This ensures that if the agent is interrupted, a newly spawned session can read the updated markdown file to resume progress.
6. Document any pragmatic conflict resolutions directly in `design.md`.

---

## 8. Subagent Prompt Templates

Structure the Orchestrator's spawning prompts to guarantee strict compliance.

### Coder Prompt

```text
== PROJECT CONTEXT ==
[Insert working dir, stack, scripts, existing infra]

== IMPORTANT: SPEC CONFLICT ==
[Clarify stack rules, e.g., "Use current stack. Do not switch stacks based on outdated specs."]

== GIT RULES (CRITICAL) ==
- Create branch from [Integration Branch]
- ONLY work on your branch
- NEVER merge
- NEVER touch other branches

== OPENSPEC APPLY WORKFLOW ==
openspec status → instructions apply → read context → implement → mark [x] → lint/build/test → commit → push

== DESIGN DECISIONS ==
[Provide explicit guidance for ambiguous areas]

== WHAT TO RETURN ==
Summary, git log, test results, push confirmation. (DO NOT return code dumps).

```

### Reviewer Prompt

```text
== CONTEXT ==
[Insert branch, spec files, and coder's summary]

== YOUR JOB ==
Get diff, read spec files, verify scenarios. Check security, correctness, and build health.

== WHAT TO RETURN ==
- Verdict (APPROVE/CHANGES_REQUESTED)
- Issues in PLAIN LANGUAGE (No cryptic codes)
- Conformance checklist
- Request changes ONLY for Blockers or Majors.

```

### Test-PO Prompt

```text
== CONTEXT ==
[Insert branch, test-id inventory, seed scripts]

== IMPORTANT: ALERTS ==
Send alert before triggering browser or server commands.

== YOUR JOB ==
Setup env → Start dev server → Run Playwright UI validation → Test APIs → Complete checklist.

== WHAT TO RETURN ==
- Verdict (PASS/FAIL)
- Failures in PLAIN LANGUAGE

```

### Rework Prompt (For Coder)

```text
You are resuming work on branch [Branch]. Reviewer/Test-PO found issues to fix. (Loop [X] of 2).

== ISSUES TO FIX ==
### BLOCKER 1: [Plain language description]
File: [file:line]
Problem: [Description]
Fix: [Action required]

== AFTER FIXES ==
lint/build/test → Commit → Push → Return summary of fixes.

```

---

## 9. Handling Interruptions

Build resilience into the Orchestrator to handle empty returns or timeout cancellations:

1. **Check Repository State**: Run `git branch --show-current`, `git log --oneline -3`, and `git status --short`.
2. **Evaluate Work**:
* *If a new commit exists*: Work was completed but the response was dropped. Verify build/tests, push, and proceed to the next pipeline phase.
* *If no new commit exists*: The subagent was interrupted before completion. Re-spawn the agent with a fresh context prompt (do not resume the broken session ID).


3. **Notify**: Broadcast an interruption recovery alert to the messaging channel.

---

## 10. Context & Todo List Management

* **Todo Lists**: Maintain a centralized orchestration list (`[ ] Spec 01: pending`, `[x] Spec 02: completed`). Transition states strictly sequentially to ensure dependencies are respected.
* **Pre-flight Context Loading**: The Orchestrator must parse the design/proposal markdown files *before* spawning the coder. Consolidate known constraints, stack decisions, and existing infrastructure into the initial prompt so the coder does not waste tokens discovering them.

---

## 11. Best Practices & Troubleshooting

* **Provide Explicit Context**: Subagents perform significantly better when infrastructure details, stack decisions, and conflicting documentation rules are resolved in the initial prompt.
* **Enforce Plain Language**: Translating error codes into human-readable sentences drastically reduces hallucinated fixes during rework loops.
* **Manage Stale Environments**: Ensure environment variables (e.g., `.env.local`) and test databases are wiped or explicitly overridden in the Test-PO prompts to prevent cross-contamination between validation runs.
* **Share Test Expectations**: Ensure the Reviewer subagent's test suite aligns with the Test-PO's browser expectations (e.g., testing invalid admin transitions at both the code review and UI levels).

---

## 12. Standard Tools Array

Ensure the underlying environment has access to the following capabilities:

| Tool Category | Purpose |
| --- | --- |
| `bash` / terminal | git, package managers, custom notifications (e.g., curl/telegram). |
| File Operations | `read`, `edit`, `write`, `glob`, `grep` for navigating and modifying codebases. |
| Agent Orchestration | Task/spawning functions to initialize and supervise subagents. |
| OpenSpec CLI | Reading status, applying instructions, persisting markdown updates. |
| Testing Tools | Browser manipulation frameworks (e.g., Playwright MCP) for UI acceptance. |
| Tracking Tools | `todowrite` or similar file-based checklist managers for orchestration state. |

---

## 13. Execution Checklist

To implement this pipeline for a new project:

1. **Prepare OpenSpec**: Verify `openspec` CLI is installed, changes are mapped in `openspec/changes/`, and task lists are pre-populated.
2. **Verify Tooling**: Ensure file, terminal, and browser capabilities are active.
3. **Configure Notifications**: Validate the external messaging setup.
4. **Execute Sequence**:
* Run specs in dependency order.
* For each spec: Extract context → Spawn Coder → Spawn Reviewer → Handle Feedback Loops → Spawn Test-PO → Handle Validation Loops → Merge & Push.


5. **Finalize**: Generate a deployment handoff summary upon clearing the spec queue.
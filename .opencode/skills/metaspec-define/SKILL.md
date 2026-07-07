---
name: metaspec-define
description: Orchestrate a multi-agent pipeline to extract requirements, design architecture, and define numbered OpenSpec proposals with zero code generation.
license: MIT
compatibility: Requires openspec CLI.
metadata:
  author: ulayera
  version: "1.0"
  generatedBy: "1.4.1"
---

# Agentic Framework — Spec Definition Pipeline

> How to orchestrate a multi-agent pipeline to extract requirements, design architecture, and define numbered OpenSpec proposals with zero code generation.

---

## 1. Overview

To guarantee successful automated implementation, specifications must be defined comprehensively before any code is written. This pipeline configures an **Orchestrator** agent to act as a systems analyst, interrogating the user for requirements, and then driving specialized subagents to draft, sequence, and QA the specifications.

### Pipeline Requirements

* **Zero-Code Policy**: This phase is strictly for defining architecture and requirements. No source code is generated.
* **Sequential Numbering**: Specs must be numbered in the exact order they need to be implemented (e.g., `01-system-design`, `02-auth`, etc.) to map dependency chains correctly.
* **Architectural Foundation First**: The first spec must always define the system design, reusable components, and core architecture.
* **3 Agent Roles**: Orchestrator, Proposer, and Expert PO/Architect.

---

## 2. Agent Roles and Responsibilities

### 2.1 Orchestrator (Main Agent)

**Role**: Supervise the requirement gathering and spec drafting lifecycle. Acts as the primary interface with the human user.

**Responsibilities**:

* Interview the user to extract a general overview of the project.
* Ask targeted questions to aggressively fill any undefined gaps in business logic, UI/UX requirements, or data structures.
* Define the optimal sequence of features, ensuring foundational elements are built before dependent features.
* Spawn **Proposer** subagents with highly specific context to write individual specs.
* Spawn **Expert PO / Architect** subagents to QA the drafted specs.
* Iterate and "one-shot" test the application design conceptually until the spec suite is comprehensive.

### 2.2 Proposer Subagent

**Role**: Translate orchestrator context into formal OpenSpec definitions.

**Responsibilities**:

* Execute OpenSpec commands (e.g., `openspec propose` or `/opsx` equivalent) to initialize spec proposals.
* Structure specs into logical tasks, defining acceptance criteria, UI states, and data models.
* Strictly adhere to the numbering system provided by the Orchestrator.
* **Constraint**: Never write application code; only write markdown/JSON proposals.

### 2.3 Expert PO / Architect Subagent

**Role**: Quality Assurance for the generated specifications.

**Responsibilities**:

* Review the output of the Proposer subagents.
* Ensure all specs are "Ready for Development" (e.g., no ambiguous requirements, edge cases accounted for, dependencies clearly stated).
* Verify architectural consistency across specs (e.g., making sure Spec 04 doesn't violate database rules established in Spec 01).
* Return a PASS/FAIL verdict with plain-language modification requests if specs are lacking.

---

## 3. Pipeline Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ ORCHESTRATOR                                                     │
│  1. Receive initial prompt / project overview from user          │
│  2. Interrogate user (Q&A loop to fill all requirement gaps)     │
│  3. Define sequence map (e.g., 01-core, 02-auth, 03-featureX)    │
│                                                                  │
│  ┌─────────────────────────────────────────────┐                │
│  │ PROPOSER (Foundation Spec)                   │                │
│  │  - Propose Spec 01 (System Design & UI)      │                │
│  │  - Define core components, tech stack rules  │                │
│  └──────────────────┬──────────────────────────┘                │
│                     │                                            │
│  ┌──────────────────▼──────────────────────────┐                │
│  │ EXPERT PO / ARCHITECT (QA)                   │                │
│  │  - Review Spec 01                            │                │
│  │  - Ensure it provides a solid base for rest  │                │
│  └──────────────────┬──────────────────────────┘                │
│                     │                                            │
│  4. For each remaining feature in sequence map:                  │
│                                                                  │
│     ┌─────────────────────────────────────────────┐             │
│     │ PROPOSER (Feature Spec)                      │             │
│     │  - Read Spec 01 + specific feature context   │             │
│     │  - openspec propose [numbered-feature]       │             │
│     └─────────────────┬───────────────────────────┘             │
│                       │                                         │
│     ┌─────────────────▼───────────────────────────┐             │
│     │ EXPERT PO / ARCHITECT (QA)                   │             │
│     │  - Review against sequence map               │             │
│     │  - Flag missing edge cases / loose ends      │             │
│     └─────────────────┬───────────────────────────┘             │
│                       │                                         │
│            ┌──────────┴──────────┐                              │
│            ▼                     ▼                              │
│           PASS                  FAIL                            │
│            │                     │                              │
│            │             [Rework Loop Triggered]                │
│            │                                                    │
│  5. Finalize suite of numbered specs ready for implementation    │
└─────────────────────────────────────────────────────────────────┘

```

---

## 4. Guardrails & Orchestration Rules

### 4.1 Aggressive Gap Filling

Before spawning any subagents, the Orchestrator must actively identify undefined technical or product areas. Do not assume behavior. If a user defines a feature, the Orchestrator must ask about:

* Empty states, error states, and loading states.
* Data retention, pagination, and syncing behaviors.
* Internationalization (i18n), accessibility, and theming rules.
* Edge cases regarding permissions or multi-tenant overlaps.

### 4.2 Foundation-First Rule

The very first spec (`01-system-design` or similar) must be dedicated entirely to the application's foundation. It must instruct the future Coder agents to build reusable UI components, set up the database schema, configure routing, establish theming (light/dark mode), and setup the component library (e.g., Storybook) before any business logic is touched.

### 4.3 Contextual Spawning

When the Orchestrator spawns a Proposer, it must inject the overarching project rules into the prompt to ensure consistency.

**Proposer Prompt Structure:**

```text
== PROJECT RULES ==
[Insert global rules: e.g., Focus on lightweight performance, fixed i18n text only, specific tech stack]

== YOUR TASK ==
Propose spec number [XX].
Feature: [Description]
Requirements: [Extracted from Orchestrator Q&A]

== CONSTRAINTS ==
- Run `openspec propose`
- Write comprehensive markdown/tasks.
- DO NOT generate code.

```

### 4.4 QA Enforcement

The Expert PO/Architect subagent acts as the gatekeeper. It must be prompted to aggressively look for implementation blockers. If a spec requires a background job but doesn't define the trigger, the Architect must fail the spec and send it back to the Proposer.
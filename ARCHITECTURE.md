# Axiom — Software Architecture Document

**Version:** 1.0
**Date:** 2026-03-15
**Status:** Approved

---

## Table of Contents

1. [Overview](#1-overview)
2. [Design Principles](#2-design-principles)
3. [System Architecture](#3-system-architecture)
4. [Core Flow](#4-core-flow)
5. [Software Requirements Specification (SRS) Format](#5-software-requirements-specification-srs-format)
6. [Orchestrator](#6-orchestrator)
7. [Sub-Orchestrators](#7-sub-orchestrators)
8. [Workers](#8-workers)
9. [Reviewers](#9-reviewers)
10. [Docker Sandbox Architecture](#10-docker-sandbox-architecture)
11. [File Router & Approval Pipeline](#11-file-router--approval-pipeline)
12. [Task System](#12-task-system)
13. [Model Registry](#13-model-registry)
14. [BitNet Local Inference](#14-bitnet-local-inference)
15. [Communication Model](#15-communication-model)
16. [Budget & Cost Management](#16-budget--cost-management)
17. [State Management & Crash Recovery](#17-state-management--crash-recovery)
18. [Git Integration](#18-git-integration)
19. [Claw Integration](#19-claw-integration)
20. [Skill System](#20-skill-system)
21. [GUI Dashboard](#21-gui-dashboard)
22. [CLI Reference](#22-cli-reference)
23. [Project Directory Structure](#23-project-directory-structure)
24. [Security Model](#24-security-model)
25. [Error Handling & Escalation](#25-error-handling--escalation)

---

## 1. Overview

### 1.1 Purpose

Axiom is a local, software-development-focused AI agent orchestration platform. It coordinates multiple AI agents — running in isolated Docker containers — to autonomously build software from a single user prompt.

### 1.2 Core Philosophy

Axiom is built on two foundational insights:

1. **The Misinterpretation Loop:** When users correct AI agents mid-execution, agents blend contradictory instructions rather than cleanly replacing them. Axiom eliminates this by enforcing a single-prompt → SRS → approval → autonomous execution flow. Once the SRS is accepted, scope is immutable.

2. **Context Overload & Hallucination:** As AI agents accumulate context about a project, they begin hallucinating references to nonexistent code, APIs, and files. Axiom prevents this by isolating workers with minimal context — each worker receives only what it needs to complete its specific task, nothing more.

### 1.3 Scope

Axiom is software-development-specific. While its primitives (tasks, agents, containers) are generic, the system is optimized for code generation workflows: SRS generation, task decomposition, code writing, automated validation, code review, and git integration.

### 1.4 Technology Stack

| Component | Technology |
|---|---|
| Core engine | Go |
| State store | SQLite |
| Worker isolation | Docker |
| Local inference | BitNet + Falcon3 |
| Cloud inference | OpenRouter API |
| GUI framework | Wails v2 (Go + React) |
| Claw integration | REST API + WebSocket + Cloudflare Tunnel |

---

## 2. Design Principles

### 2.1 Immutable Scope

Once a user accepts the SRS, the project scope SHALL NOT be modified. The user MAY pause or cancel execution, but SHALL NOT alter requirements mid-run. This is the core mechanism that prevents the misinterpretation loop.

### 2.2 Minimal Context Workers

Workers SHALL receive the minimum context required to complete their assigned task. Workers SHALL NOT have access to the project filesystem, other workers' output, the full SRS, or any information beyond their TaskSpec. The orchestrator SHALL determine the appropriate context level for each task.

### 2.3 100% Local Execution

All computation, state, and data SHALL remain on the user's machine. Axiom SHALL NOT transmit project data to any service other than the user-configured AI model providers (OpenRouter, BitNet). Axiom SHALL collect zero telemetry.

### 2.4 Disposable Agents

Every worker and reviewer SHALL be destroyed after completing its task. No agent persists between tasks. This ensures clean context boundaries and prevents state leakage between tasks.

### 2.5 Budget-Aware Orchestration

The orchestrator SHALL respect the user-defined budget ceiling at all times. Model selection, concurrency level, and retry strategies SHALL be informed by remaining budget. The system SHALL NOT exceed the budget without explicit user authorization.

---

## 3. System Architecture

### 3.1 High-Level Diagram

```
┌──────────────────────────────────────────────────────────┐
│                         USER                              │
│              (CLI / GUI / Claw Chat)                      │
└────────────────────────┬─────────────────────────────────┘
                         │ prompt
                         ▼
┌──────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                           │
│          (Claw / Claude Code / Codex / OpenCode)          │
│                                                           │
│  • Generates SRS from user prompt                         │
│  • Receives user approval on SRS                          │
│  • Decomposes SRS into hierarchical task tree              │
│  • Routes tasks to workers by complexity tier              │
│  • Spawns sub-orchestrators for subtrees                  │
│  • Performs final validation of worker output              │
│  • Commits approved changes to project branch             │
│  • Manages budget allocation across tasks                 │
└────┬──────────────┬──────────────┬───────────────────────┘
     │              │              │
     ▼              ▼              ▼
┌──────────┐ ┌──────────┐ ┌───────────────┐
│ Worker A │ │ Worker B │ │Sub-Orchestr.  │
│ (Docker) │ │ (Docker) │ │   (Docker)    │
│ OpenRtr  │ │ BitNet   │ │               │
│ Sonnet   │ │ Falcon3  │ │  ┌────┐┌────┐ │
│          │ │          │ │  │ W  ││ W  │ │
└────┬─────┘ └────┬─────┘ │  └────┘└────┘ │
     │            │        └──────┬────────┘
     ▼            ▼               ▼
┌──────────────────────────────────────────────────────────┐
│              FILE ROUTER / APPROVAL PIPELINE              │
│  • Receives staged output from worker containers          │
│  • Runs automated validation (compile, lint, test)        │
│  • Routes to reviewer container for code review           │
│  • Routes to orchestrator for final validation            │
│  • Writes approved files to project filesystem            │
└────────────────────────┬─────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────┐
│                  PROJECT FILESYSTEM                       │
│  .axiom/          (SQLite DB, config, SRS)                │
│  src/             (project source code)                   │
│  ...                                                      │
└──────────────────────────────────────────────────────────┘

HOST SERVICES (outside containers):
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ BitNet Server  │  │ Credential     │  │ Axiom API      │
│ :3002          │  │ Proxy :3001    │  │ Server :3000   │
│ Falcon3 1-bit  │  │ API keys       │  │ (for Claws)    │
└────────────────┘  └────────────────┘  └────────────────┘
```

### 3.2 Component Summary

| Component | Responsibility |
|---|---|
| **Axiom Core Engine** | CLI commands, container lifecycle, file routing, API server, task management, cost tracking |
| **Orchestrator** | SRS generation, task decomposition, model routing, final validation, git commits |
| **Sub-Orchestrator** | Manages a subtree of tasks delegated by the orchestrator |
| **Worker** | Executes a single TaskSpec inside a Docker container |
| **Reviewer** | Evaluates worker output against the TaskSpec inside a Docker container |
| **File Router** | Brokers files from container staging to project filesystem via approval pipeline |
| **Model Registry** | Unified catalog of available models (OpenRouter + BitNet) with capabilities and pricing |
| **BitNet Server** | Host-side local inference server for trivial tasks |
| **Credential Proxy** | Host-side proxy that injects API keys into container requests |
| **Axiom API Server** | REST + WebSocket server for Claw-based orchestrators |
| **GUI Dashboard** | Wails desktop app for project visualization and control |

---

## 4. Core Flow

### 4.1 Project Lifecycle

The Axiom project lifecycle SHALL follow this sequence:

```
1. INITIALIZATION
   User runs `axiom init` to create project structure.
   User configures settings (budget, concurrency, model preferences).

2. PROMPT SUBMISSION
   User provides a natural language prompt describing the desired software.
   Prompt is submitted via CLI (`axiom run`), GUI, or Claw conversation.

3. SRS GENERATION
   Orchestrator analyzes the prompt and generates a structured SRS document.
   SRS follows the mandated format (see Section 5).

4. SRS APPROVAL GATE
   SRS is presented to the user for review and approval.
   User MUST explicitly approve or reject the SRS.
   If rejected, orchestrator revises based on feedback and resubmits.
   This loop continues until the user approves.

5. SCOPE LOCK
   Upon approval, the SRS becomes IMMUTABLE.
   The SRS file is written to `.axiom/srs.md` and SHALL NOT be modified.
   No changes to scope, requirements, or acceptance criteria are permitted.

6. TASK DECOMPOSITION
   Orchestrator decomposes the SRS into a hierarchical task tree.
   Each leaf task is a single, atomic unit of work.
   Tasks are assigned dependency relationships.
   Task tree is persisted to SQLite.

7. EXECUTION LOOP
   Orchestrator identifies tasks with no unmet dependencies.
   For each available task:
     a. Orchestrator builds a TaskSpec with minimal context.
     b. Orchestrator selects worker model tier based on task complexity and budget.
     c. Worker container is spawned with TaskSpec via IPC.
     d. Worker executes and outputs to container staging area.
     e. Automated validation runs (compile, lint, tests).
     f. If automated checks fail: return to worker (max 3 retries).
     g. If retries exhausted: escalate to next model tier (max 2 escalations).
     h. If escalation exhausted: mark task BLOCKED, notify orchestrator.
     i. If automated checks pass: spawn reviewer container with ReviewSpec.
     j. If reviewer rejects: return to worker with feedback.
     k. If reviewer approves: pass to orchestrator for final validation.
     l. If orchestrator rejects: return to worker with feedback.
     m. If orchestrator approves: kill worker + reviewer containers.
     n. Write approved files to project filesystem.
     o. Run project-wide integration check (full build + test suite).
     p. If integration check fails: revert commit, reroute task.
     q. If integration check passes: commit to project branch.
     r. Update task status to DONE.
     s. Unblock dependent tasks.

8. COMPLETION
   All tasks in the task tree reach DONE status.
   Orchestrator generates final status report.
   User reviews the project branch and merges when satisfied.
```

### 4.2 Concurrency

The system SHALL support up to 10 concurrent worker containers by default. This limit SHALL be user-configurable via `.axiom/config.toml`. BitNet tasks running on the local inference server SHALL NOT count against the container concurrency limit.

The orchestrator SHALL manage the work queue and parallelize independent tasks within the concurrency limit.

### 4.3 User Intervention

During execution, the user MAY issue the following commands:

| Command | Behavior |
|---|---|
| `axiom pause` | Stop spawning new workers. Allow running workers and reviewers to complete. Persist state. |
| `axiom cancel` | Kill all active containers. Revert uncommitted changes. Mark project as CANCELLED. |
| `axiom status` | Display task tree progress, active workers, budget consumed, projected total cost. |

The user SHALL NOT modify the SRS or task tree during execution. If the user determines the SRS is incorrect, they MUST cancel and start a new project with a revised prompt.

---

## 5. Software Requirements Specification (SRS) Format

### 5.1 Structure

The SRS SHALL be a structured markdown document with the following required sections in this order:

```markdown
# SRS: <Project Name>

## 1. Architecture

### 1.1 System Overview
<High-level description of the system architecture.>

### 1.2 Component Breakdown
<List of major components/modules and their responsibilities.>

### 1.3 Technology Decisions
<Languages, frameworks, libraries, and rationale for each choice.>

### 1.4 Data Model
<Database schemas, file formats, API shapes, and data flow.>

### 1.5 Directory Structure
<Target project directory layout.>

## 2. Requirements & Constraints

### 2.1 Functional Requirements
<Numbered list of functional requirements using SHALL/MUST terminology.>
- FR-001: The system SHALL ...
- FR-002: The system MUST ...

### 2.2 Non-Functional Requirements
<Performance, scalability, security, compatibility requirements.>
- NFR-001: The system SHALL respond within ...

### 2.3 Constraints
<Technical constraints, platform requirements, dependency restrictions.>

### 2.4 Assumptions
<Assumptions made during SRS generation.>

## 3. Acceptance Criteria

### 3.1 Per-Component Criteria
<For each component defined in 1.2, specific testable acceptance criteria.>

#### Component: <Name>
- [ ] AC-001: <Testable criterion>
- [ ] AC-002: <Testable criterion>

### 3.2 Integration Criteria
<System-wide criteria that validate components working together.>
- [ ] IC-001: <Integration criterion>

### 3.3 Completion Definition
<What "done" means for this project.>
```

### 5.2 Immutability

Once approved by the user, the SRS SHALL be written to `.axiom/srs.md` with read-only file permissions. The Axiom engine SHALL reject any attempt to modify this file after acceptance. The SRS SHA-256 hash SHALL be stored in the SQLite database for integrity verification.

### 5.3 Traceability

Every task in the task tree SHALL reference one or more SRS requirements (e.g., `FR-001`, `AC-003`). This provides traceability from user requirements through task decomposition to committed code.

---

## 6. Orchestrator

### 6.1 Role

The orchestrator is the single top-level agent responsible for the entire project lifecycle. It generates the SRS, decomposes tasks, selects models, validates output, and commits code. The orchestrator is the ONLY entity (other than the user) with access to the project filesystem.

### 6.2 Supported Runtimes

| Runtime | Description | Recommended |
|---|---|---|
| **Claw** (OpenClaw / NanoClaw / Any Claw-Based Agent) | AI assistant with conversational context. Learns Axiom via skill injection. | **Yes — Primary recommendation** |
| **Claude Code** | Anthropic's CLI coding agent. Learns Axiom via CLAUDE.md + hooks. | Supported but not recommended (tendency to self-execute rather than delegate) |
| **Codex** | OpenAI's CLI coding agent. Learns Axiom via system prompt. | Supported |
| **OpenCode** | Open-source coding agent. Learns Axiom via system prompt. | Supported |

### 6.3 Claw Advantage

When a Claw-based orchestrator is used, it provides two key benefits:

1. **Model Flexibility:** The user's preferred model runs as the orchestrator without Axiom managing API keys beyond OpenRouter. This is especially valuable for users wanting Claude-class orchestration without using Claude Code.

2. **Conversational Context:** The Claw has prior context from the user's conversation, enabling higher-quality SRS generation even from vague prompts. The Claw understands what the user likely wants based on prior interactions.

### 6.4 SRS Approval Delegation

When the orchestrator is a Claw, the user MAY configure Axiom to delegate SRS approval to the Claw as a proxy. This setting SHALL be configured before project start in `.axiom/config.toml`:

```toml
[orchestrator]
srs_approval_delegate = "claw"  # "user" (default) | "claw"
```

When set to `"claw"`, the Claw SHALL review and approve the SRS on the user's behalf. The user MAY override this setting at any time.

### 6.5 Orchestrator Responsibilities

The orchestrator SHALL:

- Generate the SRS from the user's prompt.
- Present the SRS for approval (to user or delegated Claw).
- Decompose the approved SRS into a hierarchical task tree.
- Define project parameters: budget allocation, model routing table, concurrency level, review requirements.
- Build TaskSpecs for each leaf task with minimal required context.
- Select the appropriate model tier for each worker and reviewer.
- Spawn sub-orchestrators for complex subtrees when appropriate.
- Perform final validation of worker output against SRS requirements.
- Write approved files to the project filesystem.
- Execute git commits with task references.
- Track budget consumption and adjust strategy to stay within limits.
- Report project status and completion.

---

## 7. Sub-Orchestrators

### 7.1 Role

A sub-orchestrator manages a delegated subtree of the task tree. The main orchestrator creates a sub-orchestrator when a portion of the project is complex enough to warrant independent management (e.g., "backend API" or "authentication system").

### 7.2 Runtime

Sub-orchestrators SHALL run exclusively via OpenRouter. The main orchestrator SHALL select the model based on subtree complexity and available budget.

### 7.3 Capabilities

A sub-orchestrator SHALL:

- Receive a subset of SRS requirements relevant to its subtree.
- Further decompose its subtree into granular tasks.
- Build TaskSpecs for its workers.
- Select model tiers for its workers and reviewers.
- Spawn and manage worker and reviewer containers within its subtree.
- Validate worker output against its subset of requirements.
- Report results back to the main orchestrator.
- Spawn its own review team if the subtree warrants it.
- Grant lateral communication permissions between its workers when necessary.

A sub-orchestrator SHALL NOT:

- Modify the SRS.
- Access tasks outside its delegated subtree.
- Exceed its allocated budget partition.
- Communicate with workers belonging to other sub-orchestrators (unless explicitly permitted by the main orchestrator).

### 7.4 Isolation

Sub-orchestrators SHALL run inside Docker containers. They SHALL receive only the SRS sections and context relevant to their subtree. They SHALL NOT have access to the full project filesystem — file approval flows up to the main orchestrator.

---

## 8. Workers

### 8.1 Role

A worker is a disposable, single-task agent that executes one TaskSpec inside a Docker container. Workers are the lowest-level agents in the hierarchy.

### 8.2 Runtime

Workers SHALL run exclusively via OpenRouter or BitNet local inference. The orchestrator (or sub-orchestrator) SHALL select the model based on task complexity:

| Tier | Models | Use Cases |
|---|---|---|
| **Local** | BitNet/Falcon3 | Tool calls, variable renames, config changes, import additions, boilerplate generation |
| **Cheap** | Haiku, Flash, small open-source | Simple functions, unit tests, straightforward implementations |
| **Standard** | Sonnet, GPT-4o, mid-tier open-source | Most coding tasks, moderate complexity, refactoring |
| **Premium** | Opus, o1, large open-source | Complex algorithms, architectural patterns, critical-path code |

### 8.3 TaskSpec Format

Every worker SHALL receive a self-contained TaskSpec. The TaskSpec SHALL include all information the worker needs and nothing more.

```markdown
# TaskSpec: <task-id>

## Objective
<Clear, single-sentence description of what to produce.>

## Context Files
<Only the specific files and/or code snippets the worker needs.
 Files are included inline — the worker has no filesystem access
 beyond its container.>

### File: src/handlers/auth.go (lines 1-45)
```go
<relevant code snippet>
`` `

## Interface Contract
<Function signatures, type definitions, API shapes, and data structures
 that the worker's output MUST conform to.>

## Constraints
- Language: <language and version>
- Style: <coding style requirements>
- Dependencies: <allowed dependencies>
- Max file length: <line limit>

## Acceptance Criteria
- [ ] <Specific, testable criterion>
- [ ] <Specific, testable criterion>
- [ ] <Specific, testable criterion>

## Output Format
Write all output files to /workspace/staging/
Maintain the target directory structure within staging.
```

### 8.4 Worker Lifecycle

1. Docker container is spawned with the TaskSpec delivered via IPC.
2. Worker model processes the TaskSpec and generates code.
3. Worker writes output files to `/workspace/staging/` inside the container.
4. Worker signals completion.
5. Axiom extracts staged files and routes them through the approval pipeline.
6. If the worker receives revision feedback, it revises and resubmits.
7. Upon final approval (or max retry exhaustion), the container is destroyed.

### 8.5 Worker Restrictions

Workers SHALL NOT:

- Access the project filesystem.
- Know the project name, repository, or broader architecture.
- Communicate with other workers (unless explicitly granted by the orchestrator).
- Access the internet (all model calls go through the host credential proxy).
- Persist any state between tasks (containers are destroyed).
- Modify their own runtime environment.

---

## 9. Reviewers

### 9.1 Role

A reviewer is a disposable, single-task agent that evaluates a worker's output against the original TaskSpec. Reviewers run inside Docker containers and are destroyed after completing their review.

### 9.2 Runtime

Reviewers SHALL run via OpenRouter or BitNet local inference. The model tier for a reviewer SHALL be appropriate for the complexity of the task being reviewed:

| Task Tier | Reviewer Model Tier |
|---|---|
| Local (trivial) | Local (BitNet/Falcon3) |
| Cheap (simple) | Cheap (Haiku/Flash) |
| Standard (moderate) | Standard (Sonnet/GPT-4o) |
| Premium (complex) | Premium (Opus/o1) |

### 9.3 Mandate

Every task SHALL have a reviewer. No worker output SHALL be promoted to the project filesystem without passing reviewer approval.

### 9.4 ReviewSpec Format

```markdown
# ReviewSpec: <task-id>

## Original TaskSpec
<The complete TaskSpec that was given to the worker.>

## Worker Output
<The complete set of files produced by the worker.>

## Automated Check Results
✅ Compilation: PASS
✅ Linting: PASS (0 errors, 2 warnings)
✅ Unit Tests: PASS (12/12)
⚠️ Warnings:
  - line 45: unused variable `tmp` (non-blocking)

## Review Instructions
Evaluate the worker's output against the original TaskSpec.
For each acceptance criterion, determine if it is satisfied.

Respond in the following format:

### Verdict: APPROVE | REJECT

### Criterion Evaluation
- [ ] AC-001: PASS | FAIL — <explanation>
- [ ] AC-002: PASS | FAIL — <explanation>

### Feedback (if REJECT)
<Specific, actionable feedback for the worker to address.
 Reference exact line numbers and code sections.>
```

### 9.5 Reviewer Restrictions

Reviewers SHALL NOT:

- Modify the worker's code (reviewers evaluate, they do not fix).
- Access the project filesystem.
- Communicate with the worker directly (feedback is routed through Axiom).
- Know the broader project context beyond the ReviewSpec.

---

## 10. Docker Sandbox Architecture

### 10.1 Design Origin

The Docker sandbox architecture is adapted from the NanoClaw container isolation system. It provides OS-level isolation for all workers and reviewers.

### 10.2 Container Image

Axiom SHALL provide a base container image (`axiom-worker:latest`) built from the following specification:

```dockerfile
FROM node:22-slim

# Build tools for compilation checks
RUN apt-get update && apt-get install -y \
    build-essential git curl wget \
    && rm -rf /var/lib/apt/lists/*

# Language runtimes (extensible by user)
# Go, Python, Node.js available by default

WORKDIR /workspace
RUN mkdir -p /workspace/staging /workspace/ipc/input /workspace/ipc/output

USER node
ENTRYPOINT ["/app/entrypoint.sh"]
```

The image SHALL be extensible — users MAY create custom images with additional language runtimes or tools as needed for their projects.

### 10.3 Volume Mounts

| Host Path | Container Path | Mode | Purpose |
|---|---|---|---|
| `.axiom/containers/specs/<task-id>/` | `/workspace/spec/` | read-only | TaskSpec or ReviewSpec input |
| `.axiom/containers/staging/<task-id>/` | `/workspace/staging/` | read-write | Worker output staging area |
| `.axiom/containers/ipc/<task-id>/` | `/workspace/ipc/` | read-write | IPC communication channel |

The project filesystem SHALL NOT be mounted into worker or reviewer containers under any circumstances.

### 10.4 Network Isolation

Worker containers SHALL have network access restricted to:

- `host.docker.internal:3001` — Credential proxy (for OpenRouter API calls)
- `host.docker.internal:3002` — BitNet local inference server

All other network access SHALL be blocked. Workers SHALL NOT be able to reach the internet directly.

### 10.5 Credential Proxy

The credential proxy SHALL run on the host at port 3001. It SHALL:

- Accept API requests from containers with placeholder credentials.
- Inject the user's actual OpenRouter API key before forwarding to the upstream provider.
- Return responses to the container.
- Never expose the actual API key to the container environment.

### 10.6 Container Lifecycle Management

The Axiom engine SHALL:

- Spawn containers using `docker run --rm` for automatic cleanup.
- Name containers with the pattern `axiom-<task-id>-<timestamp>` for tracking.
- Enforce a hard timeout (default: 30 minutes, configurable) per container.
- Run orphan cleanup on startup, killing any `axiom-*` containers from prior crashed sessions.
- Track active container count and enforce the concurrency limit.
- Destroy containers immediately upon task approval or final retry exhaustion.

### 10.7 Security Layers

| Layer | Mechanism |
|---|---|
| OS isolation | Docker container (separate filesystem, network, process namespace) |
| Non-root execution | Containers run as `node` user |
| No project access | Project filesystem is never mounted |
| Credential isolation | API keys injected by host-side proxy, never in container env |
| Network restriction | Only credential proxy and BitNet server reachable |
| Staging boundary | Workers can only write to `/workspace/staging/` |
| IPC boundary | Communication only through filesystem IPC |
| Timeout enforcement | Hard kill after configurable timeout |
| Orphan cleanup | Stale containers destroyed on engine startup |

---

## 11. File Router & Approval Pipeline

### 11.1 Overview

The file router is the mechanism by which worker output moves from container staging to the project filesystem. No file SHALL reach the project filesystem without passing through the full approval pipeline.

### 11.2 Pipeline Stages

```
Stage 1: EXTRACTION
  Axiom extracts files from /workspace/staging/<task-id>/
  Files are copied to a host-side staging directory.

Stage 2: AUTOMATED VALIDATION
  Axiom runs automated checks against the staged files:
    - Compilation (language-appropriate: go build, tsc, gcc, etc.)
    - Linting (language-appropriate: golangci-lint, eslint, ruff, etc.)
    - Unit tests (if test files are included in the TaskSpec)
    - Interface contract validation (do exports match the spec?)

  If ANY check fails:
    → Errors are packaged as feedback.
    → Feedback is delivered to the worker via IPC.
    → Worker revises and resubmits.
    → Max 3 retries before escalation.

Stage 3: REVIEWER EVALUATION
  A reviewer container is spawned with a ReviewSpec containing:
    - The original TaskSpec
    - The worker's output
    - Automated check results

  Reviewer evaluates and returns APPROVE or REJECT with feedback.

  If REJECT:
    → Feedback is delivered to the worker via IPC.
    → Worker revises and resubmits.
    → Reviewer is destroyed and a new one spawned for the revision.

Stage 4: ORCHESTRATOR VALIDATION
  The orchestrator receives the approved output and validates it
  against the relevant SRS requirements.

  If orchestrator rejects:
    → Feedback is delivered to the worker via IPC.
    → Worker revises and resubmits.

Stage 5: COMMIT
  Approved files are written to the project filesystem.
  Project-wide integration check runs (full build + full test suite).

  If integration check fails:
    → Commit is reverted.
    → Task is rerouted (new worker, potentially different approach).

  If integration check passes:
    → Changes are committed to the project branch.
    → Task status is updated to DONE in SQLite.
    → Dependent tasks are unblocked.
```

### 11.3 Fast Track

For trivial tasks (BitNet tier), the orchestrator MAY configure a fast track that skips Stage 4 (orchestrator validation) when automated checks and reviewer both pass. This reduces latency for simple operations. The fast track SHALL be opt-in per task, decided by the orchestrator.

---

## 12. Task System

### 12.1 Task Tree

The orchestrator SHALL decompose the SRS into a hierarchical task tree. The tree SHALL be stored in SQLite with the following schema:

```sql
CREATE TABLE tasks (
    id          TEXT PRIMARY KEY,
    parent_id   TEXT REFERENCES tasks(id),
    title       TEXT NOT NULL,
    description TEXT,
    status      TEXT NOT NULL DEFAULT 'queued',
    tier        TEXT NOT NULL,            -- local | cheap | standard | premium
    srs_refs    TEXT,                     -- comma-separated SRS requirement IDs
    assigned_to TEXT,                     -- container ID when active
    model_used  TEXT,                     -- model ID used for execution
    cost        REAL DEFAULT 0,           -- cost in USD for this task
    retries     INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    started_at  DATETIME,
    completed_at DATETIME,
    FOREIGN KEY (parent_id) REFERENCES tasks(id)
);

CREATE TABLE task_dependencies (
    task_id    TEXT NOT NULL REFERENCES tasks(id),
    depends_on TEXT NOT NULL REFERENCES tasks(id),
    PRIMARY KEY (task_id, depends_on)
);
```

### 12.2 Task States

```
queued → in_progress → in_review → done
              │            │
              ▼            ▼
           failed       failed
           blocked      blocked
              │            │
              ▼            ▼
           queued        queued    (via retry or escalation)
```

| State | Description |
|---|---|
| `queued` | Task is waiting for dependencies to resolve and a worker to be assigned. |
| `in_progress` | A worker container is actively executing this task. |
| `in_review` | Worker output has passed automated checks and is being reviewed. |
| `done` | Task has passed all validation and output is committed. |
| `failed` | Task has failed automated checks or review. Awaiting retry. |
| `blocked` | Task has exhausted all retries and escalations. Requires orchestrator intervention. |

### 12.3 Dependency Enforcement

A task SHALL NOT transition from `queued` to `in_progress` unless ALL tasks in its dependency set have status `done`.

Axiom SHALL detect circular dependencies at task creation time and reject them with an error.

### 12.4 Task Decomposition Principles

The orchestrator SHALL decompose tasks according to these principles:

1. **Atomic:** Each leaf task SHALL produce a single, testable unit of work.
2. **Independent:** Tasks SHOULD minimize dependencies on other tasks.
3. **Small:** Tasks SHALL be broken down to the smallest completable unit. This maximizes the use of cheap/local models.
4. **Traceable:** Every task SHALL reference one or more SRS requirements.

---

## 13. Model Registry

### 13.1 Purpose

The model registry provides orchestrators and sub-orchestrators with current information about all available models, including capabilities, pricing, and recommended use cases.

### 13.2 Sources

The registry SHALL aggregate models from two sources:

1. **OpenRouter:** Fetched dynamically via the `/api/v1/models` endpoint.
2. **BitNet Local:** Scanned from the local BitNet server's loaded models.

### 13.3 Registry Schema

Each model entry SHALL contain:

```json
{
    "id": "anthropic/claude-4-sonnet",
    "source": "openrouter",
    "tier": "standard",
    "context_window": 200000,
    "max_output": 64000,
    "pricing": {
        "prompt_per_million": 3.00,
        "completion_per_million": 15.00
    },
    "strengths": ["code-generation", "reasoning", "refactoring", "debugging"],
    "weaknesses": ["slow-for-trivial-tasks", "expensive-for-boilerplate"],
    "supports_tools": true,
    "supports_vision": false,
    "recommended_for": ["standard-coding", "code-review", "moderate-refactoring"],
    "not_recommended_for": ["simple-renames", "config-changes"],
    "last_updated": "2026-03-15T00:00:00Z"
}
```

### 13.4 Capability Index

The `strengths`, `weaknesses`, `recommended_for`, and `not_recommended_for` fields SHALL be maintained in a curated `models.json` file shipped with Axiom. This file SHALL be updatable via:

```bash
axiom models refresh
```

This command SHALL:

1. Fetch the latest model list and pricing from OpenRouter API.
2. Download the latest `models.json` capability index from the Axiom repository.
3. Scan locally available BitNet models.
4. Merge all sources into the local SQLite registry.

### 13.5 Offline Operation

If network access is unavailable during refresh, Axiom SHALL fall back to the last cached registry data in SQLite. A warning SHALL be displayed indicating the data may be stale.

### 13.6 CLI Commands

| Command | Description |
|---|---|
| `axiom models refresh` | Update registry from all sources |
| `axiom models list` | Display all registered models |
| `axiom models list --tier <tier>` | Filter by tier (local, cheap, standard, premium) |
| `axiom models list --source <source>` | Filter by source (openrouter, local) |
| `axiom models info <model-id>` | Show detailed info for a specific model |

---

## 14. BitNet Local Inference

### 14.1 Purpose

BitNet provides free, zero-latency local inference for trivial tasks. By breaking work into the smallest possible units, many tasks can be handled by small, 1-bit quantized models running on the user's hardware.

### 14.2 Implementation

Axiom SHALL include BitNet integration using the Microsoft BitNet framework with Falcon3 series models. The BitNet server SHALL run as a host-side service, exposing an OpenAI-compatible API at `http://localhost:3002`.

### 14.3 Architecture

```
Worker Container
  → API call to http://host.docker.internal:3002/v1/chat/completions
  → BitNet server processes locally
  → Response returned to container
```

Workers SHALL NOT distinguish between BitNet and OpenRouter responses. The orchestrator's model routing determines which endpoint a worker uses. From the worker's perspective, it is simply calling a model API.

### 14.4 Model Selection

Axiom SHALL ship with support for the Falcon3 series of 1-bit quantized models. These models are selected for:

- Small memory footprint (runnable on most consumer hardware)
- Adequate capability for trivial code tasks (tool calls, renames, simple transforms)
- Zero API cost

### 14.5 User Control

BitNet local inference SHALL be enabled by default. Users MAY disable it:

```toml
# .axiom/config.toml
[bitnet]
enabled = false
```

### 14.6 CLI Commands

| Command | Description |
|---|---|
| `axiom bitnet start` | Start the local inference server |
| `axiom bitnet stop` | Stop the local inference server |
| `axiom bitnet status` | Show server status, loaded model, memory usage |
| `axiom bitnet models` | List available local models |

### 14.7 First-Run Setup

On first invocation of `axiom bitnet start`, if no model weights are present, Axiom SHALL download the Falcon3 1-bit weights automatically. The user SHALL be informed of the download size and asked to confirm.

---

## 15. Communication Model

### 15.1 Default: Strictly Hierarchical

By default, all communication SHALL be strictly hierarchical:

```
User ↔ Orchestrator ↔ Sub-Orchestrators ↔ Workers/Reviewers
```

Workers SHALL NOT communicate with other workers. Reviewers SHALL NOT communicate with workers. All information flows up and down the hierarchy through Axiom's IPC broker.

### 15.2 Orchestrator-Granted Lateral Channels

An orchestrator (or sub-orchestrator) MAY grant explicit lateral communication permission between specific workers. This is intended for cases where two workers need to coordinate (e.g., frontend and backend workers agreeing on an API contract).

Lateral communication permissions SHALL be:

- **Explicit:** The orchestrator MUST specify which workers may communicate and the scope of communication.
- **Scoped:** Permissions MAY be granted between specific workers, all workers at the same level, or individual agents at other levels.
- **Brokered:** All lateral messages SHALL pass through the host-side Axiom engine. Workers do not communicate directly.
- **Logged:** All lateral messages SHALL be recorded in the SQLite database for audit.

### 15.3 IPC Protocol

Communication between the host and containers SHALL use filesystem-based IPC:

| Direction | Mechanism |
|---|---|
| Host → Container | JSON files written to `/workspace/ipc/input/` |
| Container → Host | JSON files written to `/workspace/ipc/output/` |

The container SHALL poll `/workspace/ipc/input/` at 500ms intervals for new messages. The host SHALL poll container IPC output directories for responses.

### 15.4 Message Types

| Type | Direction | Purpose |
|---|---|---|
| `task_spec` | Host → Worker | Deliver TaskSpec for execution |
| `review_spec` | Host → Reviewer | Deliver ReviewSpec for evaluation |
| `revision_request` | Host → Worker | Return feedback for revision |
| `task_output` | Worker → Host | Submit completed work |
| `review_result` | Reviewer → Host | Submit review verdict |
| `lateral_message` | Worker → Host → Worker | Brokered lateral communication |
| `shutdown` | Host → Container | Request graceful container shutdown |

---

## 16. Budget & Cost Management

### 16.1 Budget Configuration

Before starting a project, the user SHALL set a maximum budget:

```toml
# .axiom/config.toml
[budget]
max_usd = 10.00          # Maximum spend for this project
warn_at_percent = 80     # Warn when 80% of budget consumed
```

The user MAY also set the budget via CLI:

```bash
axiom run --budget 10.00 "Build me a REST API for user management"
```

### 16.2 Cost Tracking

Axiom SHALL track costs at the following granularity:

| Level | Tracked Data |
|---|---|
| Per-request | Model ID, input tokens, output tokens, cost |
| Per-task | Total cost across all worker + reviewer requests |
| Per-agent-type | Aggregate cost for workers vs reviewers vs sub-orchestrators |
| Per-model | Aggregate cost per model used |
| Per-project | Cumulative total cost |

Cost data SHALL be stored in SQLite:

```sql
CREATE TABLE cost_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id     TEXT REFERENCES tasks(id),
    agent_type  TEXT NOT NULL,          -- worker | reviewer | sub_orchestrator
    model_id    TEXT NOT NULL,
    input_tokens  INTEGER,
    output_tokens INTEGER,
    cost_usd    REAL NOT NULL,
    timestamp   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 16.3 Budget-Aware Orchestration

The orchestrator SHALL use the budget to inform decisions:

1. **Model selection:** When budget is limited, prefer cheaper models and BitNet for more tasks.
2. **Concurrency:** Reduce parallel workers to slow spend rate if budget is tight.
3. **Retry limits:** Reduce max retries when budget is constrained.
4. **Budget reservation:** The orchestrator SHALL reserve at least 20% of the remaining budget for the review pipeline. Spending 100% on workers with nothing left for reviewers SHALL be prevented.
5. **Budget exhaustion:** When the budget is fully consumed, Axiom SHALL pause execution and prompt the user to either increase the budget or cancel.

### 16.4 Cost Display

Cost information SHALL be available via:

- `axiom status` — shows current spend, budget remaining, projected total
- GUI dashboard — real-time cost tracking with per-task breakdown
- Completion report — final cost summary by category

---

## 17. State Management & Crash Recovery

### 17.1 State Store

All project state SHALL be persisted in SQLite at `.axiom/axiom.db`. This includes:

- Task tree (tasks, dependencies, statuses)
- Active container metadata (container ID, task ID, started_at)
- Cost log
- Model registry cache
- Project configuration
- SRS hash for integrity verification
- Communication log

### 17.2 Stateless Orchestrator

The orchestrator SHALL be stateless with respect to Axiom project state. All state required for orchestration SHALL be readable from SQLite. This means:

- An orchestrator can crash and restart without losing progress.
- A different orchestrator instance can resume a project.
- The GUI can display accurate state regardless of orchestrator status.

### 17.3 Crash Recovery Procedure

On startup, the Axiom engine SHALL:

1. **Kill orphaned containers:** Find any `axiom-*` Docker containers from previous sessions and destroy them.
2. **Reconcile state:** Identify tasks with status `in_progress` or `in_review` that have no running container. Reset these to `queued`.
3. **Clean staging:** Remove any staged files that were not committed.
4. **Resume execution:** The orchestrator reads the task tree from SQLite and resumes from the current state.

### 17.4 Backup

Axiom SHALL NOT implement automated backup. The SQLite database is a single file that the user can back up with standard filesystem tools. The SRS is a markdown file in the project directory. Both are version-controlled alongside the project.

---

## 18. Git Integration

### 18.1 Branch Strategy

On project start, Axiom SHALL create a dedicated branch:

```
axiom/<project-slug>
```

All task commits SHALL be made to this branch. The user's current branch SHALL NOT be modified during execution.

### 18.2 Commit Protocol

After a task's output is approved and passes integration checks, Axiom SHALL commit with the following format:

```
[axiom] <task-title>

Task: <task-id>
SRS Refs: FR-001, AC-003
Model: anthropic/claude-4-sonnet (worker), anthropic/claude-4-haiku (reviewer)
Cost: $0.0234
```

### 18.3 Integration Checks

After each commit to the project branch, Axiom SHALL run a project-wide integration check:

1. Full build (language-appropriate).
2. Full test suite (if tests exist).
3. Linting (project-configured linters).

If the integration check fails:

1. The commit SHALL be reverted.
2. The task SHALL be rerouted — a new worker is spawned, potentially with a different approach or model.
3. The failure details SHALL be included in the new TaskSpec.

### 18.4 Project Completion

Upon project completion, the user SHALL:

1. Review the full diff of the `axiom/<project-slug>` branch against the base branch.
2. Merge the branch when satisfied.
3. Axiom SHALL NOT automatically merge or push to remote repositories.

---

## 19. Claw Integration

### 19.1 Overview

Axiom SHALL expose a local API server that Claw-based orchestrators can connect to. This enables users to orchestrate Axiom projects from their Claw conversation (Discord, Telegram, WhatsApp, etc.).

### 19.2 API Server

The Axiom API server SHALL run on the host at port 3000 (configurable). It SHALL expose:

**REST Endpoints:**

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/projects` | Create a new project |
| `POST` | `/api/v1/projects/:id/run` | Submit prompt and start SRS generation |
| `POST` | `/api/v1/projects/:id/srs/approve` | Approve the generated SRS |
| `POST` | `/api/v1/projects/:id/srs/reject` | Reject SRS with feedback |
| `GET` | `/api/v1/projects/:id/status` | Get project status, task tree, budget |
| `POST` | `/api/v1/projects/:id/pause` | Pause execution |
| `POST` | `/api/v1/projects/:id/cancel` | Cancel execution |
| `GET` | `/api/v1/projects/:id/tasks` | Get task tree with statuses |
| `GET` | `/api/v1/projects/:id/costs` | Get cost breakdown |
| `GET` | `/api/v1/models` | Get model registry |

**WebSocket:**

| Endpoint | Purpose |
|---|---|
| `ws://localhost:3000/ws/projects/:id` | Real-time project events (task completions, reviews, errors, budget warnings) |

### 19.3 Authentication

The API server SHALL require authentication via a generated token:

```bash
axiom api token generate
# Outputs: axm_sk_<random-token>
```

All API requests SHALL include the token in the `Authorization` header:

```
Authorization: Bearer axm_sk_<random-token>
```

### 19.4 Tunnel for Remote Claws

For Claw instances running in Docker containers (NanoClaw) or on remote systems, Axiom SHALL support Cloudflare Tunnel:

```bash
axiom tunnel start
# Outputs: https://<random>.trycloudflare.com
# User provides this URL to their Claw configuration
```

For OpenClaw instances running on the same host, the Claw connects directly to `http://localhost:3000`.

### 19.5 Claw Skill

Axiom SHALL generate a Claw skill file that teaches the Claw how to act as an Axiom orchestrator. The skill SHALL:

- Explain the Axiom workflow (prompt → SRS → approval → execution).
- Provide API endpoint documentation.
- Include TaskSpec and ReviewSpec templates.
- Describe the model registry and tier system.
- Explain budget management.
- Provide example orchestration workflows.

The skill is generated via:

```bash
axiom skill generate --runtime claw --output axiom-skill.md
```

---

## 20. Skill System

### 20.1 Purpose

The skill system teaches orchestrator runtimes how to use Axiom. Each supported runtime has a different mechanism for receiving instructions, so Axiom generates runtime-specific skill files.

### 20.2 Supported Runtimes

| Runtime | Skill Mechanism | Generated Artifact |
|---|---|---|
| Claw | Skill file (markdown) | `axiom-skill.md` |
| Claude Code | CLAUDE.md injection + hooks | `.claude/CLAUDE.md` + hook config |
| Codex | System prompt / instructions file | `codex-instructions.md` |
| OpenCode | System prompt / instructions file | `opencode-instructions.md` |

### 20.3 Skill Content

All generated skills SHALL include:

1. Axiom workflow overview.
2. Available CLI commands or API endpoints.
3. TaskSpec format and construction guidelines.
4. ReviewSpec format.
5. Model registry usage and tier descriptions.
6. Budget management rules.
7. Task decomposition principles.
8. Communication model rules.
9. Error handling and escalation procedures.

### 20.4 Generation

```bash
axiom skill generate --runtime <claw|claude-code|codex|opencode>
```

Skills SHALL be regenerated when project configuration changes, to ensure the orchestrator always has current information.

---

## 21. GUI Dashboard

### 21.1 Technology

The GUI SHALL be a Wails v2 desktop application (Go backend + React frontend). It provides visual monitoring and control of Axiom projects.

### 21.2 Views

| View | Content |
|---|---|
| **Project Overview** | SRS summary, budget used/remaining, overall progress percentage, elapsed time |
| **Task Tree** | Hierarchical visualization with status indicators (queued/running/review/done/blocked). Expandable nodes showing TaskSpec details. |
| **Active Containers** | Live list of running worker and reviewer containers with model, task, duration, and resource usage |
| **Cost Dashboard** | Spend per task, per model, per agent type, cumulative, projected total. Budget gauge with warning threshold. |
| **File Diff Viewer** | Side-by-side diff view of worker output vs. existing code. Shows approval status. |
| **Log Stream** | Real-time log output from active containers and Axiom engine |
| **Timeline** | Chronological event view — task starts, completions, reviews, commits, errors |
| **Model Registry** | Browseable model catalog with capabilities, pricing, and usage statistics |

### 21.3 Controls

The GUI SHALL provide controls for:

- Starting a new project (prompt input + budget configuration).
- SRS approval/rejection.
- Pausing and canceling execution.
- Viewing and modifying budget.
- Starting/stopping BitNet server.
- Starting/stopping API tunnel.

### 21.4 Real-Time Updates

The GUI backend SHALL watch the SQLite database for changes and emit events to the React frontend via Wails event system. Updates SHALL be reflected in the UI within 500ms.

---

## 22. CLI Reference

### 22.1 Project Commands

| Command | Description |
|---|---|
| `axiom init` | Initialize a new Axiom project in the current directory |
| `axiom run "<prompt>"` | Start a new project: generate SRS, await approval, execute |
| `axiom run --budget <usd> "<prompt>"` | Start with a specific budget |
| `axiom status` | Show project status, task tree, active workers, budget |
| `axiom pause` | Pause execution (finish running tasks, stop spawning new ones) |
| `axiom resume` | Resume a paused project |
| `axiom cancel` | Cancel execution, kill containers, revert uncommitted changes |

### 22.2 Model Commands

| Command | Description |
|---|---|
| `axiom models refresh` | Update model registry from OpenRouter + local |
| `axiom models list` | List all registered models |
| `axiom models list --tier <tier>` | Filter by tier |
| `axiom models info <model-id>` | Show model details |

### 22.3 BitNet Commands

| Command | Description |
|---|---|
| `axiom bitnet start` | Start local inference server |
| `axiom bitnet stop` | Stop local inference server |
| `axiom bitnet status` | Show server status and resource usage |
| `axiom bitnet models` | List available local models |

### 22.4 API & Tunnel Commands

| Command | Description |
|---|---|
| `axiom api start` | Start the REST + WebSocket API server |
| `axiom api stop` | Stop the API server |
| `axiom api token generate` | Generate a new API authentication token |
| `axiom tunnel start` | Start Cloudflare Tunnel for remote Claw access |
| `axiom tunnel stop` | Stop the tunnel |

### 22.5 Skill Commands

| Command | Description |
|---|---|
| `axiom skill generate --runtime <runtime>` | Generate skill file for specified runtime |

### 22.6 Utility Commands

| Command | Description |
|---|---|
| `axiom version` | Show Axiom version |
| `axiom doctor` | Check system requirements (Docker, BitNet, network) |
| `axiom export` | Export project state as human-readable JSON |

---

## 23. Project Directory Structure

### 23.1 Axiom Project Structure

```
project-root/
├── .axiom/
│   ├── axiom.db                    # SQLite database (all state)
│   ├── config.toml                 # Project configuration
│   ├── srs.md                      # Accepted SRS (immutable after approval)
│   ├── task-tree.md                # Human-readable task decomposition
│   ├── models.json                 # Curated model capability index
│   ├── containers/
│   │   ├── specs/                  # Generated TaskSpecs and ReviewSpecs
│   │   │   └── <task-id>/
│   │   │       └── spec.md
│   │   ├── staging/                # Worker output staging areas
│   │   │   └── <task-id>/
│   │   │       └── <output files>
│   │   └── ipc/                    # IPC communication directories
│   │       └── <task-id>/
│   │           ├── input/
│   │           └── output/
│   └── logs/
│       └── axiom.log               # Engine log
├── src/                            # Project source code (managed by Axiom)
├── ...
```

### 23.2 Global Configuration

```
~/.axiom/
├── config.toml                     # Global settings (default budget, preferred models)
├── api-tokens/                     # Generated API tokens
├── bitnet/
│   └── models/                     # Downloaded BitNet model weights
└── registry.db                     # Cached model registry
```

---

## 24. Security Model

### 24.1 Threat Model

The primary threat is a misbehaving AI agent (particularly a cheap/local model) executing harmful operations on the user's system. Axiom mitigates this through defense-in-depth:

### 24.2 Security Layers

| Layer | Threat Mitigated | Mechanism |
|---|---|---|
| Docker isolation | Agent escapes to host filesystem | Separate filesystem/process/network namespace |
| No project mount | Agent corrupts project code | Project filesystem never mounted in containers |
| Staging + approval | Bad code reaches production | Multi-gate approval pipeline (automated + reviewer + orchestrator) |
| Credential proxy | API key theft | Keys never enter container environment |
| Network restriction | Data exfiltration | Only proxy and BitNet ports reachable |
| Non-root execution | Privilege escalation | Container runs as unprivileged user |
| Timeout enforcement | Resource exhaustion | Hard kill after configurable timeout |
| Budget ceiling | Cost runaway | Execution pauses when budget exhausted |
| Integration checks | Breaking changes | Full build + test after every commit |
| Branch isolation | Main branch corruption | All work on `axiom/<slug>` branch |

### 24.3 Trust Boundaries

```
TRUSTED (host):
  - Axiom engine
  - Orchestrator (when running on host)
  - Credential proxy
  - BitNet server
  - Project filesystem
  - SQLite database

UNTRUSTED (containers):
  - Workers
  - Reviewers
  - Sub-orchestrators
```

All data crossing the trust boundary (container → host) SHALL pass through the approval pipeline. No container output SHALL be written to the project filesystem without validation.

---

## 25. Error Handling & Escalation

### 25.1 Worker Failure Escalation

When a worker fails to produce acceptable output, the following escalation SHALL be applied:

```
Tier 1: RETRY (same model)
  - Worker receives error feedback.
  - Worker revises and resubmits.
  - Max 3 retries at the same model tier.

Tier 2: ESCALATE (better model)
  - Task is reassigned to the next higher model tier.
  - New worker container is spawned with the original TaskSpec + failure history.
  - Max 2 escalations (e.g., local → cheap → standard).

Tier 3: BLOCK (orchestrator intervention)
  - Task is marked BLOCKED.
  - Orchestrator is notified with full failure history.
  - Orchestrator MAY:
    a. Restructure the task (break into smaller subtasks).
    b. Provide additional context to the TaskSpec.
    c. Manually resolve the task.
    d. Flag the task for user attention.
```

### 25.2 Integration Failure

When a committed task breaks the project-wide integration check:

1. The commit SHALL be reverted.
2. The task SHALL be re-queued with the integration failure details included in the TaskSpec.
3. A new worker SHALL be spawned to resolve the conflict.
4. If the conflict persists after escalation, the orchestrator SHALL be notified to restructure the affected tasks.

### 25.3 Budget Exhaustion

When the project budget is fully consumed:

1. All active containers SHALL be allowed to complete their current work.
2. No new workers SHALL be spawned.
3. The user SHALL be notified with a cost summary and progress report.
4. The user MAY increase the budget to resume execution.
5. The user MAY cancel the project.

### 25.4 Orchestrator Failure

If the orchestrator crashes or disconnects:

1. Running containers SHALL continue until their timeout expires.
2. Axiom engine SHALL persist all state to SQLite.
3. On orchestrator reconnection, it SHALL resume from persisted state (see Section 17.3).

### 25.5 Docker Daemon Failure

If the Docker daemon becomes unavailable:

1. `axiom doctor` SHALL detect this condition.
2. Axiom SHALL NOT attempt to spawn new containers.
3. The user SHALL be notified to restart Docker.
4. Running containers may be lost — Axiom SHALL reconcile state on recovery (see Section 17.3).

---

## Appendix A: Configuration Reference

### `.axiom/config.toml`

```toml
[project]
name = "my-project"
slug = "my-project"

[budget]
max_usd = 10.00
warn_at_percent = 80

[concurrency]
max_workers = 10

[orchestrator]
runtime = "claw"                        # claw | claude-code | codex | opencode
srs_approval_delegate = "user"          # user | claw

[bitnet]
enabled = true
host = "localhost"
port = 3002

[docker]
image = "axiom-worker:latest"
timeout_minutes = 30
network_mode = "bridge"

[git]
auto_commit = true
branch_prefix = "axiom"

[api]
port = 3000
```

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| **SRS** | Software Requirements Specification — the immutable project definition document |
| **TaskSpec** | Self-contained task description delivered to a worker |
| **ReviewSpec** | Task description + worker output delivered to a reviewer |
| **Orchestrator** | Top-level agent responsible for project lifecycle |
| **Sub-Orchestrator** | Delegated agent managing a subtree of tasks |
| **Worker** | Disposable agent that executes a single TaskSpec |
| **Reviewer** | Disposable agent that evaluates worker output |
| **File Router** | System that brokers files from containers to project filesystem |
| **Model Registry** | Catalog of available AI models with capabilities and pricing |
| **BitNet** | Local 1-bit quantized inference engine for trivial tasks |
| **Claw** | AI assistant platform (OpenClaw/NanoClaw) used as recommended orchestrator |
| **Credential Proxy** | Host-side service that injects API keys into container requests |
| **IPC** | Inter-Process Communication via filesystem between host and containers |
| **Tier** | Model complexity classification (local, cheap, standard, premium) |
| **Gate** | Approval checkpoint in the file router pipeline |

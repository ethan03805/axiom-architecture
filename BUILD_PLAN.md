# Axiom — Build Plan

**Version:** 1.0
**Date:** 2026-03-16
**Architecture Reference:** ARCHITECTURE.md v2.2

This document describes how to build the Axiom platform from start to finish. It is organized as a sequence of phases, each containing discrete build steps. Each phase builds on the previous one. Within a phase, steps can be parallelized where noted.

---

## Table of Contents

1. [Phase 0: Project Scaffolding](#phase-0-project-scaffolding)
2. [Phase 1: Core Engine Foundation](#phase-1-core-engine-foundation)
3. [Phase 2: Container Lifecycle](#phase-2-container-lifecycle)
4. [Phase 3: IPC Protocol](#phase-3-ipc-protocol)
5. [Phase 4: Inference Broker](#phase-4-inference-broker)
6. [Phase 5: Task System & Concurrency](#phase-5-task-system--concurrency)
7. [Phase 6: File Router & Approval Pipeline](#phase-6-file-router--approval-pipeline)
8. [Phase 7: Validation Sandbox](#phase-7-validation-sandbox)
9. [Phase 8: Semantic Indexer](#phase-8-semantic-indexer)
10. [Phase 9: Merge Queue & Git Integration](#phase-9-merge-queue--git-integration)
11. [Phase 10: Orchestrator Runtime (Embedded)](#phase-10-orchestrator-runtime-embedded)
12. [Phase 11: SRS & ECO System](#phase-11-srs--eco-system)
13. [Phase 12: Budget & Cost Management](#phase-12-budget--cost-management)
14. [Phase 13: Model Registry](#phase-13-model-registry)
15. [Phase 14: BitNet Local Inference](#phase-14-bitnet-local-inference)
16. [Phase 15: CLI](#phase-15-cli)
17. [Phase 16: API Server & Claw Integration](#phase-16-api-server--claw-integration)
18. [Phase 17: Skill System](#phase-17-skill-system)
19. [Phase 18: Security Hardening](#phase-18-security-hardening)
20. [Phase 19: Observability & Crash Recovery](#phase-19-observability--crash-recovery)
21. [Phase 20: GUI Dashboard](#phase-20-gui-dashboard)
22. [Phase 21: Docker Images](#phase-21-docker-images)
23. [Phase 22: Integration Testing](#phase-22-integration-testing)
24. [Phase 23: End-to-End Testing](#phase-23-end-to-end-testing)

---

## Phase 0: Project Scaffolding

**Goal:** Initialize the Go project, establish directory structure, and set up development tooling.

### Steps

0.1. Initialize Go module:
  - `go mod init github.com/ethan03805/axiom`
  - Set Go version to 1.22+.

0.2. Create the project directory structure:

```
axiom/
├── cmd/
│   └── axiom/
│       └── main.go              # CLI entrypoint
├── internal/
│   ├── engine/                  # Trusted Engine core
│   │   ├── engine.go            # Engine struct, startup, shutdown
│   │   └── config.go            # Configuration loading (TOML)
│   ├── state/                   # SQLite state management
│   │   ├── db.go                # Database initialization, migrations
│   │   ├── tasks.go             # Task CRUD operations
│   │   ├── events.go            # Event logging
│   │   ├── costs.go             # Cost tracking
│   │   └── eco.go               # ECO records
│   ├── container/               # Docker container lifecycle
│   │   ├── manager.go           # Container spawning, destruction, tracking
│   │   ├── images.go            # Image selection logic
│   │   └── hardening.go         # Security flags, hardening policy
│   ├── ipc/                     # Filesystem-based IPC protocol
│   │   ├── protocol.go          # Message types, serialization
│   │   ├── watcher.go           # inotify-based file watching
│   │   └── handler.go           # Request dispatching
│   ├── broker/                  # Inference Broker
│   │   ├── broker.go            # Request routing, budget checks
│   │   ├── openrouter.go        # OpenRouter API client
│   │   ├── bitnet.go            # BitNet local client
│   │   └── streaming.go         # Chunked streaming support
│   ├── pipeline/                # File Router & Approval Pipeline
│   │   ├── router.go            # Manifest validation, path safety
│   │   ├── pipeline.go          # Stage orchestration
│   │   └── manifest.go          # Manifest parsing and validation
│   ├── merge/                   # Merge Queue
│   │   ├── queue.go             # Serialized merge processing
│   │   └── conflict.go          # Conflict detection, re-queue logic
│   ├── index/                   # Semantic Indexer
│   │   ├── indexer.go           # tree-sitter integration
│   │   ├── queries.go           # Typed query API
│   │   └── refresh.go           # Full and incremental refresh
│   ├── git/                     # Git operations
│   │   ├── manager.go           # Branch creation, commits, merges
│   │   └── snapshot.go          # Base snapshot management
│   ├── budget/                  # Budget enforcement
│   │   ├── enforcer.go          # Per-request budget checks
│   │   └── tracker.go           # Cost aggregation and reporting
│   ├── srs/                     # SRS and ECO management
│   │   ├── parser.go            # SRS format validation
│   │   ├── lock.go              # Immutability enforcement, SHA-256
│   │   └── eco.go               # ECO proposal, validation, application
│   ├── registry/                # Model Registry
│   │   ├── registry.go          # Model aggregation and lookup
│   │   ├── openrouter.go        # OpenRouter model fetching
│   │   └── performance.go       # Historical performance tracking
│   ├── security/                # Security subsystems
│   │   ├── secrets.go           # Secret scanning and redaction
│   │   ├── injection.go         # Prompt injection mitigation
│   │   └── validation.go        # Path canonicalization, file safety
│   ├── orchestrator/            # Embedded orchestrator runtime
│   │   ├── embedded.go          # Embedded mode container management
│   │   ├── bootstrap.go         # Bootstrap mode context
│   │   └── taskspec.go          # TaskSpec/ReviewSpec generation helpers
│   ├── skill/                   # Skill System
│   │   └── generator.go         # Runtime-specific skill file generation
│   ├── api/                     # REST + WebSocket API server
│   │   ├── server.go            # HTTP server, routing
│   │   ├── handlers.go          # Endpoint handlers
│   │   ├── websocket.go         # WebSocket event streaming
│   │   ├── auth.go              # Token authentication
│   │   └── ratelimit.go         # Per-token rate limiting
│   ├── tunnel/                  # Cloudflare Tunnel integration
│   │   └── tunnel.go            # Tunnel start/stop/status
│   ├── events/                  # Event emission system
│   │   ├── emitter.go           # Event bus
│   │   └── types.go             # Event type definitions
│   └── doctor/                  # System health checks
│       └── doctor.go            # Docker, BitNet, resource checks
├── gui/                         # Wails v2 frontend
│   ├── frontend/                # React app
│   │   ├── src/
│   │   │   ├── App.tsx
│   │   │   ├── components/
│   │   │   ├── views/
│   │   │   ├── hooks/
│   │   │   └── types/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── app.go                   # Wails app definition
│   └── bindings.go              # Go-to-React bindings
├── docker/                      # Dockerfile definitions
│   ├── Dockerfile.meeseeks-go
│   ├── Dockerfile.meeseeks-node
│   ├── Dockerfile.meeseeks-python
│   ├── Dockerfile.meeseeks-multi
│   └── axiom-seccomp.json       # Optional seccomp profile
├── schemas/                     # Embedded SQL migrations
│   └── 001_initial.sql
├── skills/                      # Skill templates
│   ├── claw.md.tmpl
│   ├── claude-code.md.tmpl
│   ├── codex.md.tmpl
│   └── opencode.md.tmpl
├── go.mod
├── go.sum
└── Makefile
```

0.3. Set up the Makefile with targets:
  - `build` — compile the `axiom` binary.
  - `test` — run all unit tests.
  - `lint` — run `golangci-lint`.
  - `docker-images` — build all Meeseeks Docker images.
  - `gui` — build the Wails frontend.
  - `all` — build everything.

0.4. Add core Go dependencies:
  - `github.com/mattn/go-sqlite3` (SQLite driver)
  - `github.com/BurntSushi/toml` (TOML config parsing)
  - `github.com/spf13/cobra` (CLI framework)
  - `github.com/docker/docker` (Docker SDK)
  - `github.com/gorilla/websocket` (WebSocket support)
  - `github.com/fsnotify/fsnotify` (inotify wrapper for IPC)
  - `github.com/smacker/go-tree-sitter` (tree-sitter bindings)

0.5. Create the initial SQLite migration file (`schemas/001_initial.sql`) containing all tables from Architecture Section 15.2:
  - `tasks` (with `eco_ref` column)
  - `task_srs_refs`
  - `task_dependencies`
  - `task_target_files`
  - `task_locks` (normalized: `resource_type`, `resource_key` primary key)
  - `task_attempts`
  - `validation_runs`
  - `review_runs`
  - `task_artifacts`
  - `container_sessions`
  - `events`
  - `cost_log`
  - `eco_log`

0.6. Implement configuration loading (`internal/engine/config.go`):
  - Parse `.axiom/config.toml` with all sections from Appendix A.
  - Parse `~/.axiom/config.toml` for global defaults.
  - Merge project config over global config.
  - Validate required fields and provide sensible defaults.

**Acceptance Criteria:**
- `go build ./cmd/axiom` succeeds.
- `go test ./...` passes (even if tests are minimal stubs).
- SQLite database initializes with all tables from the schema.
- Config loading reads and merges TOML files correctly.

---

## Phase 1: Core Engine Foundation

**Goal:** Build the engine struct, lifecycle management, and event system.

**Depends on:** Phase 0

### Steps

1.1. Implement the Engine struct (`internal/engine/engine.go`):
  - Holds references to all subsystems (state, container manager, broker, etc.).
  - `New(config)` constructor initializes all subsystems.
  - `Start()` method begins the engine event loop.
  - `Shutdown()` method performs graceful shutdown (kill containers, flush state, close DB).

1.2. Implement the state layer (`internal/state/`):
  - `db.go`: Open SQLite in WAL mode, run migrations, configure `PRAGMA busy_timeout=5000`, set `MaxOpenConns(10)`.
  - `tasks.go`: Full CRUD for tasks table — Create, Get, List, Update status, bulk create (`create_task_batch`). Query tasks by status, by parent, by dependency readiness.
  - `events.go`: Insert events with type, task_id, agent_type, agent_id, JSON details. Query events with filtering.
  - `costs.go`: Insert cost records, query aggregations (per-task, per-model, per-agent-type, per-project total).
  - `eco.go`: Insert ECO records, update status, query by status.

1.3. Implement the event emission system (`internal/events/`):
  - `emitter.go`: In-process event bus with subscriber registration. Supports multiple subscribers (GUI, API WebSocket, internal consumers).
  - `types.go`: Define all event types as constants: `task_created`, `task_started`, `task_completed`, `task_failed`, `task_blocked`, `container_spawned`, `container_destroyed`, `review_started`, `review_completed`, `merge_started`, `merge_completed`, `budget_warning`, `budget_exhausted`, `eco_proposed`, `eco_approved`, `eco_rejected`, `srs_submitted`, `srs_approved`, `scope_expansion_requested`, `scope_expansion_approved`, `scope_expansion_denied`, `context_invalidation_warning`, `provider_unavailable`.

1.4. Implement task state machine enforcement:
  - Valid transitions: `queued` -> `in_progress` -> `in_review` -> `done`.
  - Side transitions: `in_progress` -> `failed` -> `queued` (retry), `in_progress` -> `blocked`, `in_progress` -> `waiting_on_lock` -> `queued`.
  - `cancelled_eco` can be set from any active state.
  - Reject invalid transitions with descriptive errors.
  - Dependency enforcement: task cannot leave `queued` unless all dependencies are `done`.
  - Circular dependency detection at task creation time.

**Acceptance Criteria:**
- Engine starts, initializes SQLite with all tables, and shuts down cleanly.
- Task CRUD operations work with proper state machine enforcement.
- Events are emitted and received by subscribers.
- Invalid state transitions are rejected.
- Circular dependencies are detected and rejected.

---

## Phase 2: Container Lifecycle

**Goal:** Spawn, manage, and destroy Docker containers with full hardening.

**Depends on:** Phase 1

### Steps

2.1. Implement the container manager (`internal/container/manager.go`):
  - `SpawnMeeseeks(taskID, image, spec)` — creates a container with TaskSpec mounted.
  - `SpawnReviewer(taskID, image, reviewSpec)` — creates a container with ReviewSpec.
  - `SpawnSubOrchestrator(taskID, image, config)` — creates a sub-orchestrator container.
  - `SpawnValidator(taskID, image, projectSnapshot, meeseeksOutput)` — creates a validation sandbox.
  - `Destroy(containerID)` — forcefully removes a container.
  - `ListActive()` — returns all running `axiom-*` containers.
  - Track active container count and enforce the concurrency limit (`max_meeseeks` from config).

2.2. Implement container naming:
  - Pattern: `axiom-<task-id>-<timestamp>`.
  - Log all lifecycle events to the `container_sessions` table.

2.3. Implement volume mount setup:
  - Create host directories: `.axiom/containers/specs/<task-id>/`, `.axiom/containers/staging/<task-id>/`, `.axiom/containers/ipc/<task-id>/input/`, `.axiom/containers/ipc/<task-id>/output/`.
  - Mount specs as read-only, staging and IPC as read-write.
  - Never mount the project filesystem.

2.4. Implement the hardening policy (`internal/container/hardening.go`):
  - Apply all flags from Architecture Section 12.6.1:
    - `--read-only`
    - `--cap-drop=ALL`
    - `--security-opt=no-new-privileges`
    - `--pids-limit=256`
    - `--tmpfs /tmp:rw,noexec,size=256m`
    - `--network=none`
    - `--user <uid>:<gid>`
    - `--cpus` and `--memory` from config
  - Optional seccomp profile application.
  - Build these into a `ContainerConfig` struct that is applied uniformly.

2.5. Implement timeout enforcement:
  - Per-container hard timeout (default 30 minutes, configurable).
  - Background goroutine monitors container age and kills expired containers.
  - Log timeout kills as events.

2.6. Implement orphan cleanup:
  - On engine startup, find all `axiom-*` containers and destroy them.
  - Reconcile `container_sessions` table: mark orphaned sessions as `exit_reason = "orphaned"`.

2.7. Implement concurrency control:
  - Track active Meeseeks count.
  - When at limit, queue spawn requests and process when a slot frees up.
  - Separate tracking for BitNet tasks (they use local resources, not container slots).

**Acceptance Criteria:**
- Containers spawn with correct volume mounts, hardening flags, and resource limits.
- Container naming follows the pattern.
- Containers are destroyed on task completion and on timeout.
- Orphan cleanup runs on startup.
- Concurrency limit is respected.
- All lifecycle events are logged to `container_sessions` and `events`.

---

## Phase 3: IPC Protocol

**Goal:** Implement filesystem-based IPC with inotify event notification.

**Depends on:** Phase 2

### Steps

3.1. Define message types (`internal/ipc/protocol.go`):
  - Go structs for every message type from Architecture Section 20.4:
    - `task_spec`, `review_spec`, `revision_request`, `task_output`, `review_result`
    - `inference_request`, `inference_response`
    - `lateral_message`
    - `action_request`, `action_response`
    - `request_scope_expansion`, `scope_expansion_response`
    - `context_invalidation_warning`
    - `shutdown`
  - JSON serialization/deserialization for each.
  - Include scope expansion example formats from Architecture Section 10.7.

3.2. Implement file-based message delivery:
  - Engine -> Container: Write JSON files to `.axiom/containers/ipc/<task-id>/input/`.
  - Container -> Engine: Read JSON files from `.axiom/containers/ipc/<task-id>/output/`.
  - File naming: `<message-type>-<sequence-number>.json`.

3.3. Implement the inotify watcher (`internal/ipc/watcher.go`):
  - Watch all active container IPC output directories using `fsnotify`.
  - On new file: read, parse, dispatch to handler.
  - Fallback: if `fsnotify` fails, poll at 1-second intervals.

3.4. Implement the request handler (`internal/ipc/handler.go`):
  - Dispatch incoming messages by type to the appropriate engine subsystem.
  - `inference_request` -> Inference Broker.
  - `task_output` -> File Router / Approval Pipeline.
  - `review_result` -> Pipeline stage advancement.
  - `action_request` -> Action dispatcher (spawn, query, etc.).
  - `request_scope_expansion` -> Scope expansion handler.
  - Return responses by writing to the container's input directory.

3.5. Implement streaming support for inference responses:
  - Chunked response files: `response-001.json`, `response-002.json`, etc.
  - Engine writes chunks as they arrive from the model provider.
  - Container uses inotify to process chunks incrementally.

**Acceptance Criteria:**
- Messages round-trip between engine and a test container via filesystem IPC.
- inotify correctly detects new IPC files within 100ms.
- All message types serialize/deserialize correctly.
- Streaming inference responses deliver chunks incrementally.
- Fallback polling works when inotify is unavailable.

---

## Phase 4: Inference Broker

**Goal:** Mediate all model API calls with budget enforcement, model allowlists, and audit.

**Depends on:** Phase 3

### Steps

4.1. Implement the broker core (`internal/broker/broker.go`):
  - Receive `inference_request` messages from IPC handler.
  - Validate: model allowlist (is the requested model allowed for this task's tier?).
  - Validate: token budget (max_tokens x model pricing fits within remaining task budget).
  - Validate: rate limit (per-task request count, default max 50).
  - If all checks pass: route to the appropriate provider.
  - If any check fails: return error response via IPC.
  - Log every request and response in `cost_log` table.

4.2. Implement the OpenRouter client (`internal/broker/openrouter.go`):
  - HTTP client for the OpenRouter `/api/v1/chat/completions` endpoint.
  - Support streaming (SSE) and non-streaming modes.
  - Handle API errors, rate limits, and retries.
  - API key loaded from `~/.axiom/config.toml`, never injected into containers.

4.3. Implement the BitNet client (`internal/broker/bitnet.go`):
  - HTTP client for the local BitNet server at `localhost:3002`.
  - OpenAI-compatible API format.
  - Support grammar constraints (GBNF) in requests.
  - Handle server unavailability gracefully.

4.4. Implement model routing logic:
  - Route based on task tier and model selection.
  - If OpenRouter is unavailable: BitNet-eligible tasks auto-route to local.
  - Non-BitNet tasks queue until connectivity restored.
  - Emit `provider_unavailable` event on outage.

4.5. Implement streaming relay (`internal/broker/streaming.go`):
  - Receive SSE chunks from OpenRouter.
  - Write chunked IPC response files to the container's input directory.
  - Track token counts from streaming responses.

4.6. Implement credential management:
  - API keys stored in `~/.axiom/config.toml`.
  - Support config reload via `axiom config reload` without engine restart.
  - Keys never appear in container environments or IPC messages.

**Acceptance Criteria:**
- Inference requests from containers are correctly routed to OpenRouter or BitNet.
- Model allowlist enforcement prevents tier violations.
- Budget enforcement rejects over-budget requests.
- Rate limiting prevents runaway inference loops.
- Every request is logged in `cost_log` with model, tokens, cost, latency.
- Streaming responses are relayed as chunked IPC files.
- Fallback routing works when OpenRouter is down.

---

## Phase 5: Task System & Concurrency

**Goal:** Implement the task tree, write-set locking, dependency resolution, and work queue.

**Depends on:** Phase 1

### Steps

5.1. Implement the task tree manager:
  - Create hierarchical task trees from orchestrator requests.
  - Support `create_task` and `create_task_batch` (atomic multi-task creation).
  - Validate parent-child relationships.
  - Store SRS references in `task_srs_refs` junction table.
  - Store target files in `task_target_files` with `lock_scope`.

5.2. Implement write-set locking:
  - Lock acquisition using the normalized `task_locks` table (`resource_type`, `resource_key`).
  - Lock scope escalation: file -> package -> module -> schema (based on orchestrator's TaskSpec).
  - Deterministic lock ordering: acquire all locks in alphabetical order by canonical resource key.
  - Atomic acquisition: all-or-nothing. If any lock is held, the task remains `queued`.
  - Lock release on task completion (merge commit) or task failure.

5.3. Implement the work queue:
  - Query for tasks in `queued` state whose dependencies are all `done`.
  - Filter out tasks whose locks cannot be acquired.
  - Respect the concurrency limit.
  - Priority: process tasks in dependency order (earliest unblocked first).

5.4. Implement scope expansion handling:
  - Receive `request_scope_expansion` IPC messages.
  - Check lock availability for requested files.
  - If available: acquire locks, update `task_target_files`, notify orchestrator for approval.
  - If locked: destroy current Meeseeks, set task to `waiting_on_lock`, record blocking task.
  - On blocking task completion: transition `waiting_on_lock` tasks back to `queued` with expanded scope in the new TaskSpec.

5.5. Implement context invalidation warnings:
  - After each merge queue commit, check semantic index for symbol changes.
  - Identify active Meeseeks whose TaskSpec context references changed symbols.
  - Send `context_invalidation_warning` IPC messages to affected containers.
  - This is optional/best-effort — the merge queue is the authoritative safety net.

**Acceptance Criteria:**
- Task trees create correctly with parent-child relationships and SRS refs.
- Write-set locks enforce mutual exclusion.
- Deterministic lock ordering prevents deadlocks.
- Atomic lock acquisition prevents partial locking.
- Work queue correctly identifies and dispatches ready tasks.
- Scope expansion handles both approval and lock-conflict paths.
- `waiting_on_lock` tasks re-queue when blocking tasks complete.

---

## Phase 6: File Router & Approval Pipeline

**Goal:** Implement the multi-stage approval pipeline from Meeseeks output to merge queue.

**Depends on:** Phases 2, 3, 5

### Steps

6.1. Implement manifest parsing (`internal/pipeline/manifest.go`):
  - Parse `manifest.json` from Meeseeks staging directory.
  - Validate: all listed files exist in staging.
  - Validate: no unlisted files exist in staging.
  - Validate: binary flag and size_bytes for binary files.
  - Reject malformed manifests with descriptive errors.

6.2. Implement path safety validation (`internal/pipeline/router.go`):
  - Canonicalize all paths (resolve `..`, normalize separators).
  - Reject symlinks, device files, FIFOs.
  - Reject oversized files (configurable max, default 1MB).
  - Reject paths outside the task's declared `target_files` scope.
  - Validate expanded-scope files if scope expansion was approved.

6.3. Implement the pipeline orchestrator (`internal/pipeline/pipeline.go`):
  - Stage 1: Extraction and manifest validation.
  - Stage 2: Spawn validation sandbox (delegate to validation sandbox subsystem).
  - Stage 3: Spawn reviewer container with ReviewSpec. On reject: destroy current Meeseeks, spawn fresh Meeseeks with new TaskSpec including feedback, spawn fresh reviewer for next round.
  - Stage 4: Forward to orchestrator for final validation. On reject: same fresh-container treatment.
  - Stage 5: Submit to merge queue.
  - Track retry count per task. After 3 retries: escalate model tier. After 2 escalations: mark `blocked`.

6.4. Implement ReviewSpec generation:
  - Combine: original TaskSpec + Meeseeks output files + manifest + validation sandbox results.
  - Format per Architecture Section 11.7.
  - For standard+ tiers: select reviewer from different model family than the Meeseeks.

6.5. Implement batched review for trivial tasks:
  - For local-tier tasks: allow batching multiple related tasks into a single ReviewSpec.
  - Validate that all tasks in a batch are functionally related (same module check).
  - On batch rejection: return entire batch for revision.

**Acceptance Criteria:**
- Manifests are validated completely (file existence, path safety, scope compliance).
- Invalid manifests produce clear rejection messages.
- Pipeline stages execute in order with proper container lifecycle (fresh containers on retry).
- Retry and escalation logic follows the architecture spec.
- ReviewSpecs include all required context.
- Batched review works for local-tier tasks.

---

## Phase 7: Validation Sandbox

**Goal:** Run compilation, linting, and tests against untrusted Meeseeks output in isolated containers.

**Depends on:** Phase 2

### Steps

7.1. Implement validation sandbox spawning:
  - Create an overlay filesystem: read-only project snapshot at HEAD + writable layer with Meeseeks output applied.
  - Apply the same hardening as Meeseeks containers plus:
    - No network access.
    - No secrets or credentials.
    - Separate timeout (default 10 minutes).
    - Higher CPU/memory limits than Meeseeks (configurable).
  - Use the same language-specific image as the project's configured Meeseeks image (Section 13.7).

7.2. Implement language-specific validation profiles (`internal/container/images.go`):
  - Go: vendored modules or read-only GOMODCACHE.
  - Node: `npm ci --ignore-scripts` + read-only node_modules cache.
  - Python: pre-built wheels, `pip install --no-index --find-links`.
  - Rust: cargo with pre-populated registry.
  - Auto-detect project language(s) from config.

7.3. Implement validation execution:
  - Run sequentially: dependency install -> compilation -> linting -> unit tests -> security scan (if enabled).
  - Binary files (manifest `binary: true`) skip compilation and linting but enforce size limits and path validity.
  - Capture pass/fail per check, error output, test coverage.
  - Return structured results to the engine.
  - Destroy the sandbox container after completion.

7.4. Implement integration sandbox (opt-in):
  - When `[validation.integration]` is enabled in config.
  - Scoped network egress to allowed services only.
  - Scoped secrets (only those listed in config).
  - Separate from the default hermetic sandbox.

7.5. Implement warm sandbox pools (behind feature flag):
  - Maintain configurable number of pre-warmed validation containers synced to HEAD.
  - On Meeseeks completion: push file diff to warm sandbox via IPC.
  - Run incremental build/test against the diff.
  - After validation: discard overlay, reap all processes, recreate temp dirs, return to pool.
  - Periodic cold validation every N warm uses (configurable, default 10).
  - Default: `warm_pool_enabled = false`.

**Acceptance Criteria:**
- Validation sandboxes run all checks against Meeseeks output.
- No network access or secrets are available in the sandbox.
- Language-specific profiles handle dependencies correctly.
- Binary files are properly handled (skip compile/lint, enforce size).
- Warm pools (when enabled) reduce latency and maintain isolation invariants.
- Validation results are returned as structured data to the pipeline.

---

## Phase 8: Semantic Indexer

**Goal:** Maintain a queryable code index using tree-sitter for precise TaskSpec context construction.

**Depends on:** Phase 1

**Can be parallelized with:** Phases 2-7

### Steps

8.1. Integrate tree-sitter via Go bindings (`internal/index/indexer.go`):
  - Add language grammars: Go, JavaScript/TypeScript, Python, Rust.
  - Parse source files into ASTs.
  - Extract: functions (name, file, line, params, return type, exported), types/structs (fields, methods, implements), interfaces (method signatures), constants/variables, imports, exports, package dependencies.

8.2. Implement the index storage:
  - Store index data in the project's SQLite database (separate tables or a dedicated index schema).
  - Index contents per Architecture Section 17.3.

8.3. Implement refresh cycles (`internal/index/refresh.go`):
  - Full index: on project initialization and on `axiom index refresh`.
  - Incremental index: after each successful merge queue commit (re-index only changed files).
  - Exclude `.axiom/` directory from indexing.

8.4. Implement the typed query API (`internal/index/queries.go`):
  - `lookup_symbol(name, type)` -> file, line, signature, exported status.
  - `reverse_dependencies(symbol_name)` -> list of files/symbols that reference it.
  - `list_exports(package_path)` -> all exported symbols with types.
  - `find_implementations(interface_name)` -> all implementing types.
  - `module_graph(root_package)` -> dependency graph.
  - All queries are structured, not natural language.

8.5. Wire the indexer to the IPC handler:
  - Handle `query_index` IPC requests from orchestrator/sub-orchestrator containers.
  - Validate query type and parameters.
  - Return results in JSON format.

**Acceptance Criteria:**
- tree-sitter correctly parses Go, JS/TS, Python, and Rust files.
- Index contains all symbol types from the architecture spec.
- Full and incremental refresh work correctly.
- All five query types return accurate results.
- `.axiom/` is excluded from the index.

---

## Phase 9: Merge Queue & Git Integration

**Goal:** Serialize commits, validate against HEAD, and manage the project branch.

**Depends on:** Phases 5, 7, 8

### Steps

9.1. Implement Git manager (`internal/git/manager.go`):
  - Create project branch: `axiom/<project-slug>`.
  - Commit with the format from Architecture Section 23.2 (task ID, SRS refs, models, attempt, cost, base snapshot).
  - Never modify the user's current branch.
  - Refuse to start on a dirty working tree (SHA refuse with clear error message listing uncommitted files).

9.2. Implement base snapshot management (`internal/git/snapshot.go`):
  - Record the git SHA each TaskSpec is generated against.
  - Compare base snapshots when processing the merge queue.

9.3. Implement the merge queue (`internal/merge/queue.go`):
  - Process one approved output at a time (serialized).
  - Steps per Architecture Section 16.4:
    1. Receive approved output.
    2. Validate `base_snapshot` against current HEAD.
    3. If stale: attempt merge or re-queue.
    4. Apply output to working copy of HEAD.
    5. Run integration checks in validation sandbox.
    6. If integration fails: revert, re-queue task.
    7. If integration passes: commit, update HEAD.
    8. Re-index via semantic indexer.
    9. Release write-set locks.
    10. Process next item.

9.4. Implement conflict handling (`internal/merge/conflict.go`):
  - On stale base snapshot: attempt three-way merge.
  - If clean merge: proceed with integration checks.
  - If conflict: re-queue task with updated context (new base snapshot, fresh semantic index data).

9.5. Wire lock release to dependent task unblocking:
  - After lock release: query for `waiting_on_lock` tasks blocked by the released resources.
  - Transition them back to `queued`.
  - Query for `queued` tasks whose dependencies are now all `done`.
  - Dispatch to the work queue.

**Acceptance Criteria:**
- Project branch is created correctly.
- Commits follow the required format.
- Merge queue processes items serially.
- Stale base snapshots trigger re-merge or re-queue.
- Integration checks run in validation sandbox before every commit.
- Lock release correctly unblocks waiting tasks.
- Dirty working tree is refused at startup.

---

## Phase 10: Orchestrator Runtime (Embedded)

**Goal:** Support embedded orchestrator containers (Claude Code, Codex, OpenCode) with managed lifecycle.

**Depends on:** Phases 2, 3, 4

### Steps

10.1. Implement embedded orchestrator spawning (`internal/orchestrator/embedded.go`):
  - Spawn an orchestrator container with the configured runtime.
  - Deliver the user prompt and project config via IPC.
  - All inference goes through the Inference Broker.
  - Track all orchestrator inference in cost_log.

10.2. Implement bootstrap mode (`internal/orchestrator/bootstrap.go`):
  - For existing projects: provide read-only repo-map + full semantic index access.
  - For greenfield projects: provide only user prompt + project config.
  - Scope bootstrap context to SRS generation phase only.
  - After SRS approval: switch to normal TaskSpec-building mode.

10.3. Implement orchestrator action handling:
  - Handle all request types from the orchestrator via IPC:
    - `submit_srs`, `submit_eco`, `create_task`, `create_task_batch`
    - `spawn_meeseeks`, `spawn_reviewer`, `spawn_sub_orchestrator`
    - `approve_output`, `reject_output`
    - `query_index`, `query_status`, `query_budget`
    - `request_inference`
  - Validate each request against current state and policy.
  - Execute authorized actions.
  - Return results via IPC.

10.4. Implement orchestrator reconnection:
  - If orchestrator crashes: persist all state.
  - On reconnection: resume from SQLite state.
  - Running Meeseeks continue until timeout.

**Acceptance Criteria:**
- Embedded orchestrator containers spawn with correct config and bootstrap context.
- All IPC request types are handled correctly.
- Budget tracking covers all orchestrator inference.
- Crash recovery preserves state for orchestrator reconnection.

---

## Phase 11: SRS & ECO System

**Goal:** Implement SRS generation, approval, immutability, and the ECO process.

**Depends on:** Phase 10

### Steps

11.1. Implement SRS format validation (`internal/srs/parser.go`):
  - Validate that submitted SRS follows the mandated structure (Architecture Section 6.1).
  - Check for required sections: Architecture, Requirements, Test Strategy, Acceptance Criteria.
  - Validate requirement numbering (FR-xxx, NFR-xxx, AC-xxx).

11.2. Implement the SRS approval flow:
  - Orchestrator submits SRS via `submit_srs` IPC.
  - Engine writes SRS to `.axiom/srs.md` as draft.
  - Present to user for approval (via CLI prompt, GUI, or API for external Claws).
  - If rejected: return feedback to orchestrator for revision.
  - If approved: lock the SRS.

11.3. Implement SRS immutability (`internal/srs/lock.go`):
  - On approval: set `.axiom/srs.md` to read-only file permissions.
  - Compute SHA-256 hash and store in SQLite.
  - On engine startup: verify SRS hash matches stored hash.
  - Reject any attempt to modify the file after lock.

11.4. Implement SRS approval delegation:
  - When `srs_approval_delegate = "claw"`: auto-approve via the Claw orchestrator.
  - Log delegated approvals with the Claw's identity.
  - ECO approval follows the same delegation setting.

11.5. Implement ECO management (`internal/srs/eco.go`):
  - Validate ECO category against the six allowed categories.
  - Reject ECOs that don't match a defined category.
  - Present ECO to user (or delegated Claw) for approval.
  - On approval: record as versioned addendum (never overwrite original SRS).
  - Mark affected tasks as `cancelled_eco` with `eco_ref` set.
  - Orchestrator replans only affected tasks.

**Acceptance Criteria:**
- SRS format validation catches malformed documents.
- Approval flow works for user, GUI, and API paths.
- SRS becomes immutable after approval (file permissions, hash verification).
- ECO validation enforces the six defined categories.
- ECO approval/rejection flows work correctly.
- Cancelled tasks have correct `eco_ref` traceability.

---

## Phase 12: Budget & Cost Management

**Goal:** Enforce budget limits, track costs at all granularities, and report spending.

**Depends on:** Phase 4

**Can be parallelized with:** Phases 5-11

### Steps

12.1. Implement budget enforcement (`internal/budget/enforcer.go`):
  - On every inference request: calculate maximum possible cost (`max_tokens` x model pricing).
  - Verify it fits within remaining budget.
  - Reject requests whose max cost would exceed remaining budget.
  - This is dynamic per-request pre-authorization, not fixed percentage reservation.

12.2. Implement cost tracking (`internal/budget/tracker.go`):
  - Track at all granularities from Architecture Section 21.2: per-request, per-attempt, per-task, per-agent-type, per-model, per-project.
  - Aggregate from `cost_log` table.
  - Calculate projected total cost based on completion percentage and current spend rate.

12.3. Implement budget-aware orchestration signals:
  - Emit `budget_warning` event at configurable threshold (default 80%).
  - Emit `budget_exhausted` event at 100%.
  - On exhaustion: allow active containers to finish, block new spawns, prompt user.
  - Support budget increase via CLI or GUI.

12.4. Implement external client mode budget handling:
  - In external client mode: budget covers engine-managed costs only.
  - Cost displays include note: "Orchestrator inference cost not tracked (external mode)."

**Acceptance Criteria:**
- Per-request budget verification prevents overspend.
- Cost tracking is accurate at all granularities.
- Budget warnings fire at the configured threshold.
- Budget exhaustion correctly pauses execution.
- External mode displays the appropriate disclaimer.

---

## Phase 13: Model Registry

**Goal:** Aggregate and serve model information from OpenRouter and local BitNet.

**Depends on:** Phase 0

**Can be parallelized with:** Phases 1-12

### Steps

13.1. Implement registry aggregation (`internal/registry/registry.go`):
  - Aggregate models from OpenRouter API and local BitNet server.
  - Store in `~/.axiom/registry.db` (SQLite).
  - Schema per Architecture Section 18.3: id, family, source, tier, context window, max output, pricing, strengths, weaknesses, supports_tools, supports_vision, supports_grammar_constraints, recommended_for, not_recommended_for, historical stats.

13.2. Implement OpenRouter model fetching (`internal/registry/openrouter.go`):
  - Fetch from `/api/v1/models` endpoint.
  - Map models to tiers (local/cheap/standard/premium) based on pricing and capabilities.
  - Merge with curated `models.json` for strengths/weaknesses/recommendations.

13.3. Implement model performance history (`internal/registry/performance.go`):
  - After each project: update `historical_success_rate` and `avg_cost_per_task`.
  - Track per-model, per-codebase-style metrics.

13.4. Implement offline fallback:
  - If network unavailable during refresh: fall back to cached data.
  - Display stale-data warning.

13.5. Implement model selection helpers:
  - Given a task tier and budget: return ranked list of suitable models.
  - Factor in: pricing, capability, historical performance, model family diversity (for reviewer selection).

**Acceptance Criteria:**
- Registry aggregates models from OpenRouter and BitNet.
- Curated capability data merges correctly.
- Historical performance updates after project completion.
- Offline fallback works with stale-data warning.
- Model selection respects tier, budget, and diversity requirements.

---

## Phase 14: BitNet Local Inference

**Goal:** Integrate the BitNet/Falcon3 local inference server with grammar-constrained decoding.

**Depends on:** Phase 4

**Can be parallelized with:** Phases 5-13

### Steps

14.1. Implement BitNet server management:
  - Start/stop the BitNet server (bitnet.cpp with Falcon3 weights).
  - Configure CPU threads, max concurrent requests from config.
  - Track resource usage separately from container concurrency.

14.2. Implement grammar-constrained decoding support:
  - When a task requires structured output: include GBNF grammar constraints in the inference request.
  - Broker passes grammar constraints through to the BitNet server.

14.3. Implement first-run setup:
  - On first `axiom bitnet start`: check for model weights.
  - If absent: prompt user to confirm download, download Falcon3 1-bit weights.
  - Store in `~/.axiom/bitnet/models/`.

14.4. Implement resource monitoring:
  - Track BitNet CPU/memory usage.
  - Warn if combined load (containers + BitNet) exceeds system capacity.

**Acceptance Criteria:**
- BitNet server starts and stops correctly.
- Grammar-constrained decoding produces valid structured output.
- First-run download works with user confirmation.
- Resource usage is tracked and warnings fire on overload.

---

## Phase 15: CLI

**Goal:** Implement the complete CLI using Cobra.

**Depends on:** Phases 1-14

### Steps

15.1. Implement project commands:
  - `axiom init` — create `.axiom/` directory structure, write default config.
  - `axiom run "<prompt>"` — start project lifecycle. Options: `--budget <usd>`.
  - `axiom status` — display task tree, active containers, budget, resources.
  - `axiom pause` — stop spawning new Meeseeks, let active ones complete.
  - `axiom resume` — resume a paused project.
  - `axiom cancel` — kill all containers, revert uncommitted changes, mark cancelled.
  - `axiom export` — export project state as JSON.

15.2. Implement model commands:
  - `axiom models refresh` — update registry from all sources.
  - `axiom models list` — list all models. Flags: `--tier`, `--family`.
  - `axiom models info <model-id>` — show detailed model info.

15.3. Implement BitNet commands:
  - `axiom bitnet start` — start local inference server (with first-run download).
  - `axiom bitnet stop` — stop server.
  - `axiom bitnet status` — show server status, memory, active requests.
  - `axiom bitnet models` — list local models.

15.4. Implement API and tunnel commands:
  - `axiom api start` / `axiom api stop` — API server lifecycle.
  - `axiom api token generate` — generate auth token. Flags: `--scope`, `--expires`.
  - `axiom api token list` — list active tokens.
  - `axiom api token revoke <token-id>` — revoke a token.
  - `axiom tunnel start` / `axiom tunnel stop` — Cloudflare Tunnel.

15.5. Implement skill commands:
  - `axiom skill generate --runtime <claw|claude-code|codex|opencode>`.

15.6. Implement index commands:
  - `axiom index refresh` — force full re-index.
  - `axiom index query --type <query_type> [--name <symbol>] [--package <pkg>]` — structured query.

15.7. Implement utility commands:
  - `axiom version` — show version.
  - `axiom doctor` — check Docker, BitNet, network, resources, warm-pool images, secret scanner regexes.
  - `axiom config reload` — reload config without restart.

**Acceptance Criteria:**
- All commands from Architecture Section 27 are implemented.
- Commands produce clear output and handle errors gracefully.
- `axiom doctor` validates all system requirements.
- Help text is comprehensive for all commands and flags.

---

## Phase 16: API Server & Claw Integration

**Goal:** Expose REST + WebSocket API for external orchestrators (Claws).

**Depends on:** Phase 1

**Can be parallelized with:** Phases 2-15

### Steps

16.1. Implement the HTTP server (`internal/api/server.go`):
  - Listen on configurable port (default 3000).
  - Route all endpoints from Architecture Section 24.2.
  - Middleware: authentication, rate limiting, audit logging, CORS.

16.2. Implement endpoint handlers (`internal/api/handlers.go`):
  - All REST endpoints from the architecture:
    - Project lifecycle: create, run, approve/reject SRS, approve/reject ECO, pause, resume, cancel.
    - Data queries: status, tasks, attempts, costs, events, models.
    - Semantic index: `POST /api/v1/index/query` with structured JSON body.
  - Map API requests to engine operations.

16.3. Implement WebSocket event streaming (`internal/api/websocket.go`):
  - `ws://localhost:3000/ws/projects/:id` — stream real-time events.
  - Subscribe to the engine's event emitter.
  - Send events to connected WebSocket clients.

16.4. Implement token authentication (`internal/api/auth.go`):
  - Generate tokens: `axm_sk_<random>`.
  - Token storage in `~/.axiom/api-tokens/`.
  - Configurable expiration (default 24 hours).
  - Scoped tokens: `read-only` (GET only) or `full-control` (all).
  - Revocation support.
  - Log all auth events (including failures with source IP).

16.5. Implement rate limiting (`internal/api/ratelimit.go`):
  - Per-token rate limiting (configurable, default 120 RPM).
  - Return `429 Too Many Requests` with `Retry-After` header.

16.6. Implement optional IP restrictions:
  - When `allowed_ips` is configured: reject requests from non-allowed IPs.

16.7. Implement Cloudflare Tunnel integration (`internal/tunnel/tunnel.go`):
  - Start/stop `cloudflared` tunnel pointing to the local API server.
  - Output the public URL for remote Claw connections.

**Acceptance Criteria:**
- All REST endpoints work correctly.
- WebSocket streams events in real-time.
- Token authentication works with expiration, scoping, and revocation.
- Rate limiting enforces the configured RPM.
- Tunnel exposes the API to remote Claws.
- All API requests are audit-logged.

---

## Phase 17: Skill System

**Goal:** Generate runtime-specific instruction files that teach orchestrators how to use Axiom.

**Depends on:** Phase 0

**Can be parallelized with:** all other phases

### Steps

17.1. Create skill templates (`skills/*.md.tmpl`):
  - One template per runtime: Claw, Claude Code, Codex, OpenCode.
  - Each template includes all 13 content items from Architecture Section 25.3.
  - Templates use Go `text/template` for dynamic content (project config, model registry summary).

17.2. Implement skill generator (`internal/skill/generator.go`):
  - `axiom skill generate --runtime <runtime>` produces the skill file.
  - Inject current project config, model registry data, and API endpoint info.
  - Write to appropriate location:
    - Claw: `axiom-skill.md`
    - Claude Code: `.claude/CLAUDE.md` + hook config
    - Codex: `codex-instructions.md`
    - OpenCode: `opencode-instructions.md`

17.3. Implement regeneration on config change:
  - When project config changes, re-generate active skill files.

**Acceptance Criteria:**
- Skill files are generated for all four runtimes.
- Generated content covers all 13 required topics.
- Dynamic content (config, models) is correctly injected.
- Files are regenerated when config changes.

---

## Phase 18: Security Hardening

**Goal:** Implement secret scanning, prompt injection mitigation, and file safety rules.

**Depends on:** Phases 3, 6

### Steps

18.1. Implement secret scanning (`internal/security/secrets.go`):
  - File sensitivity classification based on path patterns (`.env*`, `*credentials*`, `*secret*`, etc.).
  - Regex-based content scanning for: API key patterns (`sk-`, `ghp_`, `AKIA`), connection strings, base64 secrets, private key blocks, high-entropy strings.
  - Redaction: replace detected secrets with `[REDACTED]` in TaskSpec context.
  - Option to exclude entire files if secret density is high.
  - Log each redaction event (file, line, pattern) without logging the secret value.
  - User-configurable sensitivity patterns in config.

18.2. Implement sensitive file routing:
  - When `force_local_for_sensitive = true` (default): tasks touching sensitive files are forced to BitNet.
  - Per-task user override option (logged in events table).

18.3. Implement prompt injection mitigation (`internal/security/injection.go`):
  - Data wrapping: all repo content in TaskSpecs wrapped in `<untrusted_repo_content>` tags with source attribution.
  - Instruction separation: prompt templates include explicit instructions to treat wrapped content as data only.
  - Provenance labels: every code snippet includes source file path and line range.
  - Exclusion list: `.axiom/`, `.env*`, generated state files, logs — never included in prompts.
  - Comment sanitization: flag comments containing instruction-like patterns. Option to strip or add reinforced wrapping.

18.4. Implement file safety validation (`internal/security/validation.go`):
  - Path canonicalization (resolve all `..`, normalize separators).
  - Symlink rejection.
  - Device file and FIFO rejection.
  - Oversized file rejection (configurable max).
  - Scope enforcement (staged files must match declared target_files).
  - Deletion safety (only via manifest declaration).

18.5. Implement prompt log safety:
  - When `log_prompts = true`: apply same redaction to prompt log entries.
  - Never store raw secrets in prompt logs.

**Acceptance Criteria:**
- Secret scanning detects common API key patterns and sensitive files.
- Redaction replaces secrets without breaking context structure.
- Sensitive files route to BitNet by default.
- Prompt injection mitigations are applied to all TaskSpecs and ReviewSpecs.
- File safety rules catch path traversal, symlinks, and scope violations.
- Prompt logs never contain raw secrets.

---

## Phase 19: Observability & Crash Recovery

**Goal:** Implement prompt logging, crash recovery, and system health checks.

**Depends on:** Phase 1

### Steps

19.1. Implement prompt logging:
  - When `log_prompts = true`: write full prompt + response to `.axiom/logs/prompts/<task-id>-<attempt>.json`.
  - When `log_prompts = false`: only token counts, model ID, cost, latency in `task_attempts` table.
  - Apply secret redaction to logged prompts.

19.2. Implement crash recovery (`internal/engine/engine.go`):
  - On startup:
    1. Kill orphaned `axiom-*` containers.
    2. Reconcile tasks: `in_progress` or `in_review` with no running container -> reset to `queued`.
    3. Release stale locks (locks held by dead containers).
    4. Clean staging directories.
    5. Verify SRS integrity (SHA-256 hash check).
    6. Resume: orchestrator reads task tree from SQLite.

19.3. Implement the doctor command (`internal/doctor/doctor.go`):
  - Check Docker daemon availability and version.
  - Check BitNet server status.
  - Check network connectivity (OpenRouter reachability).
  - Check system resources (CPU, memory, disk).
  - Validate warm-pool images match project config.
  - Validate secret scanner regex patterns are valid.
  - Report results with pass/fail/warning per check.

**Acceptance Criteria:**
- Prompt logging captures or omits content based on config.
- Crash recovery correctly resets state and cleans up orphans.
- SRS integrity verification catches tampering.
- `axiom doctor` validates all system requirements with clear output.

---

## Phase 20: GUI Dashboard

**Goal:** Build the Wails v2 desktop application with React frontend.

**Depends on:** Phases 1, 15

**Can be parallelized with:** many earlier phases once the event system exists

### Steps

20.1. Initialize Wails project:
  - Set up Wails v2 with Go backend + React frontend.
  - Configure Go-to-React bindings for engine operations.

20.2. Implement the React frontend views (from Architecture Section 26.2):
  - **Project Overview:** SRS summary, budget gauge, progress percentage, elapsed time, ECO history.
  - **Task Tree:** Hierarchical tree visualization with status indicators (color-coded). Expandable nodes showing TaskSpec details, attempt history, cost.
  - **Active Containers:** Live list of running Meeseeks/reviewers with model, task, duration, CPU/memory.
  - **Cost Dashboard:** Spend charts per task, per model, per agent type. Cumulative total. Projected total. Budget gauge with warning threshold.
  - **File Diff Viewer:** Side-by-side diff view of Meeseeks output. Approval pipeline status indicator.
  - **Log Stream:** Real-time scrolling event log.
  - **Timeline:** Chronological event visualization (task starts, completions, reviews, commits, errors, ECOs).
  - **Model Registry:** Browsable model catalog with capabilities, pricing, historical performance.
  - **Resource Monitor:** System CPU/memory usage, container resource consumption, BitNet server load.

20.3. Implement GUI controls (Architecture Section 26.3):
  - New project creation (prompt input + budget configuration).
  - SRS approval/rejection with inline viewing.
  - ECO approval/rejection.
  - Pause, resume, cancel buttons.
  - Budget modification.
  - BitNet server start/stop.
  - API tunnel start/stop.

20.4. Implement real-time updates:
  - Subscribe to engine events via Wails event system.
  - Update UI within 500ms of event receipt.
  - Do not poll SQLite directly — all data through engine events.

20.5. Add external mode budget indicator:
  - When orchestrator is in external client mode: display note "Orchestrator inference cost not tracked (external mode)" on cost dashboard.

**Acceptance Criteria:**
- All nine views render correctly with real data.
- Controls execute engine operations correctly.
- Real-time updates display within 500ms.
- Budget display includes external mode disclaimer when applicable.
- UI handles edge cases (empty task tree, no active containers, budget exhaustion).

---

## Phase 21: Docker Images

**Goal:** Build all Meeseeks and validation sandbox Docker images.

**Depends on:** Phase 0

**Can be parallelized with:** all other phases

### Steps

21.1. Build language-specific Meeseeks images:
  - `axiom-meeseeks-go:latest` — Go toolchain + golangci-lint.
  - `axiom-meeseeks-node:latest` — Node.js + npm + TypeScript + eslint.
  - `axiom-meeseeks-python:latest` — Python 3 + pip + ruff + mypy.
  - `axiom-meeseeks-multi:latest` — Go + Node.js + Python (default).

21.2. Each image must include:
  - Non-root user configured.
  - IPC watcher script (reads from `/workspace/ipc/input/`, writes to `/workspace/ipc/output/`).
  - Minimal shell for the LLM agent to execute within.
  - No pre-installed API keys or secrets.

21.3. Build validation sandbox images:
  - Same language toolchains as Meeseeks images.
  - Include test runners, linters, security scanners (optional).
  - Pre-populated dependency caches for hermetic builds.

21.4. Create optional seccomp profile:
  - `docker/axiom-seccomp.json` restricting unnecessary syscalls.

21.5. Implement image build automation in Makefile:
  - `make docker-images` builds all images.
  - Tag with version for reproducibility.

**Acceptance Criteria:**
- All four Meeseeks images build successfully.
- Images contain correct toolchains and run as non-root.
- IPC watcher script functions correctly inside containers.
- Validation sandbox images include test/lint tooling.
- Images are minimal size (multi-stage builds where appropriate).

---

## Phase 22: Integration Testing

**Goal:** Test subsystem interactions end-to-end within the engine.

**Depends on:** Phases 1-21

### Steps

22.1. Test the full IPC round-trip:
  - Engine writes TaskSpec to container IPC directory.
  - Container (simulated) writes task_output to output directory.
  - Engine detects, validates manifest, runs pipeline.

22.2. Test the approval pipeline end-to-end:
  - Manifest validation -> validation sandbox -> reviewer -> orchestrator -> merge queue.
  - Test all reject/retry paths with fresh container spawning.

22.3. Test write-set locking with concurrent tasks:
  - Two tasks targeting the same file: second queues until first completes.
  - Scope expansion with lock conflict: Meeseeks destroyed, task enters `waiting_on_lock`.
  - Lock release unblocks waiting tasks.

22.4. Test the merge queue:
  - Two tasks complete and enter the queue simultaneously.
  - Verify serialized processing.
  - Verify stale base snapshot triggers re-queue.
  - Verify integration check failure triggers re-queue.

22.5. Test budget enforcement:
  - Task at budget boundary: request rejected.
  - Budget exhaustion: execution pauses, user prompted.
  - Budget increase: execution resumes.

22.6. Test ECO flow:
  - ECO proposed, approved: affected tasks cancelled, replacements created.
  - ECO proposed, rejected: orchestrator continues without change.

22.7. Test crash recovery:
  - Simulate engine crash mid-execution.
  - Restart: verify orphan cleanup, state reconciliation, lock release, SRS integrity.

22.8. Test secret scanning and redaction:
  - Files with API keys: verify redaction in TaskSpecs.
  - Sensitive files: verify BitNet routing.

**Acceptance Criteria:**
- All integration tests pass.
- Edge cases (concurrent locks, budget boundaries, crash recovery) are covered.
- Tests run in CI without requiring external services (mock OpenRouter, BitNet).

---

## Phase 23: End-to-End Testing

**Goal:** Run Axiom against real projects to validate the complete system.

**Depends on:** Phase 22

### Steps

23.1. Create test project fixtures:
  - Simple Go project: single-file CLI tool.
  - Simple Node project: Express API with 3 endpoints.
  - Simple Python project: Flask app with database.

23.2. Test with embedded orchestrator:
  - Run `axiom run` with each test project.
  - Verify: SRS generation, approval, task decomposition, Meeseeks execution, review, merge, completion.
  - Verify: cost tracking is accurate.
  - Verify: git branch contains all committed work.

23.3. Test with external Claw orchestrator:
  - Start API server and tunnel.
  - Connect a Claw via the API.
  - Run a project end-to-end via API.
  - Verify: partial budget tracking, audit logging.

23.4. Test BitNet integration:
  - Run a project with local-tier tasks routed to BitNet.
  - Verify: grammar-constrained decoding produces valid output.
  - Verify: BitNet tasks complete and merge correctly.

23.5. Test error recovery:
  - Introduce intentional failures (bad code, failing tests).
  - Verify: retry with fresh containers, escalation, blocking.
  - Verify: ECO flow when a dependency is unavailable.

23.6. Test GUI:
  - Run a project with the GUI open.
  - Verify: all views update in real-time.
  - Verify: controls (pause, resume, cancel) work correctly.

**Acceptance Criteria:**
- All test projects complete successfully end-to-end.
- Cost tracking matches actual API spend.
- Git history is clean with proper commit messages.
- Error recovery works as specified.
- GUI displays accurate real-time state.

---

## Build Order Summary

The following diagram shows the dependency relationships between phases:

```
Phase 0: Project Scaffolding
    │
    ├── Phase 1: Core Engine Foundation
    │       │
    │       ├── Phase 2: Container Lifecycle
    │       │       │
    │       │       ├── Phase 3: IPC Protocol
    │       │       │       │
    │       │       │       └── Phase 4: Inference Broker
    │       │       │
    │       │       └── Phase 7: Validation Sandbox
    │       │
    │       ├── Phase 5: Task System & Concurrency
    │       │
    │       ├── Phase 8: Semantic Indexer (parallelizable)
    │       │
    │       └── Phase 19: Observability & Crash Recovery
    │
    ├── Phase 13: Model Registry (parallelizable)
    │
    ├── Phase 17: Skill System (parallelizable)
    │
    ├── Phase 21: Docker Images (parallelizable)
    │
    └── Phase 16: API Server (parallelizable)

Phase 5 + Phase 7 + Phase 8 → Phase 9: Merge Queue & Git
Phase 2 + Phase 3 + Phase 4 → Phase 10: Orchestrator Runtime
Phase 10 → Phase 11: SRS & ECO
Phase 4 → Phase 12: Budget & Cost
Phase 4 → Phase 14: BitNet Local Inference
Phase 3 + Phase 6 → Phase 18: Security Hardening

Phases 2-7 → Phase 6: File Router & Approval Pipeline

Phases 1-14 → Phase 15: CLI
Phase 1 + Phase 15 → Phase 20: GUI Dashboard

Phases 1-21 → Phase 22: Integration Testing
Phase 22 → Phase 23: End-to-End Testing
```

Phases marked "parallelizable" have no dependencies on the main critical path and can be built concurrently with other work.

---

## Critical Path

The longest dependency chain determines the minimum sequential build order:

```
Phase 0 → Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 10 → Phase 11
                         │
                         └→ Phase 7 ──┐
                                       ├→ Phase 9
Phase 1 → Phase 5 ───────────────────┘
Phase 1 → Phase 8 ──────────────────┘
```

The critical path runs through: Scaffolding -> Engine -> Containers -> IPC -> Broker -> Orchestrator -> SRS/ECO. Everything else can be parallelized around this spine.

---

## Notes for AI Coding Agents

- Each phase is designed to be a self-contained unit of work with clear inputs, outputs, and acceptance criteria.
- Phases marked "can be parallelized" are safe to assign to separate agents working concurrently.
- The Architecture document (ARCHITECTURE.md v2.2) is the authoritative reference for all implementation details. When this build plan and the architecture document disagree, the architecture document takes precedence.
- All code is Go (engine) and React/TypeScript (GUI). No other languages in the core codebase.
- Test coverage should accompany every phase. Do not defer all testing to Phase 22.
- Use the Architecture document's SQL schema verbatim for database tables.
- Use the Architecture document's IPC message formats verbatim for protocol messages.
- Use the Architecture document's config structure verbatim for TOML parsing.

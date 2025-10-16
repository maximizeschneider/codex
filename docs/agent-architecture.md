# Codex Coding Agent Architecture

This document summarizes the major subsystems that power the Codex coding agent implemented in `codex-rs/core`. It focuses on how a session is constructed, how prompts and tools are assembled, and which guardrails keep the agent safe.

## Runtime Overview
- `Codex::spawn` is the front door for clients. It builds asynchronous submission/event channels, pulls configuration (including base and user instructions), and launches the background submission loop that will service requests for the lifetime of the session.【F:codex-rs/core/src/codex.rs†L150-L205】
- `Session::new` materializes the long-lived session context. It establishes the conversation id, initializes rollout logging, bootstraps MCP connections, discovers the default shell, gathers history metadata, and records any startup failures so they can be surfaced to the UI.【F:codex-rs/core/src/codex.rs†L312-L442】
- Once infrastructure is ready, the session wires telemetry (OpenTelemetry), constructs the model client with provider/model settings, and stores service objects (executor, MCP manager, notification hooks, rollout recorder, shell discovery) for later use.【F:codex-rs/core/src/codex.rs†L443-L515】【F:codex-rs/core/src/state/service.rs†L1-L18】

## Session Context & Services
- A `Session` owns the event channel, shared state, and bookkeeping for the currently running turn; it guarantees that only one task executes at a time and tracks per-turn artifacts.【F:codex-rs/core/src/codex.rs†L240-L250】
- Each turn executes with a `TurnContext` that captures the model client, working directory, instruction overrides, approval/sandbox policy, shell policy, enabled tools, and optional JSON schema for structured outputs.【F:codex-rs/core/src/codex.rs†L252-L268】
- `SessionServices` exposes reusable helpers—MCP connection manager, exec session managers, notifier, rollout recorder, detected shell, reasoning toggle, and the sandbox-aware executor—that tools and tasks can draw from.【F:codex-rs/core/src/state/service.rs†L1-L18】

## Task Lifecycle
- Tasks implement the `SessionTask` trait, providing a `run` entry point and optional `abort`. The session wraps them in `SessionTaskContext` so they can reach shared services safely.【F:codex-rs/core/src/tasks/mod.rs†L13-L53】
- Launching a task aborts any in-flight work, spawns the new asynchronous runner, and remembers its handle so it can be cancelled or awaited. Completion emits a `TaskComplete` event with the final agent message (if any).【F:codex-rs/core/src/tasks/mod.rs†L55-L105】
- If a task needs to stop early, `handle_task_abort` cancels the async handle, calls the task’s `abort` hook, and sends a `TurnAborted` event with the reason, ensuring the UI understands why work stopped.【F:codex-rs/core/src/tasks/mod.rs†L107-L180】

## Prompt Assembly & System Instructions
- Each turn builds a `Prompt` struct holding conversation items, tool specs, optional base-instruction override, and optional response schema. The helper `get_full_instructions` merges model defaults with overrides and appends extra `apply_patch` guidance when the model needs it but no patch tool is present.【F:codex-rs/core/src/client_common.rs†L26-L87】
- When the freeform `apply_patch` tool is enabled, shell outputs are reserialized into human-readable summaries so the model sees structured metadata instead of raw JSON blobs.【F:codex-rs/core/src/client_common.rs†L88-L188】
- The primary system prompt (`core/prompt.md`) defines tone, responsiveness requirements, planning expectations, sandbox awareness, and validation philosophy for the coding agent, while `gpt_5_codex_prompt.md` layers on model-specific guidance about shell usage, editing discipline, approvals, and final messaging etiquette.【F:codex-rs/core/prompt.md†L1-L198】【F:codex-rs/core/gpt_5_codex_prompt.md†L1-L106】
- Specialized prompts such as the review prompt are pulled in via constants like `REVIEW_PROMPT`, making it easy for downstream UIs to swap instructions for review-mode conversations.【F:codex-rs/core/src/client_common.rs†L23-L33】

## Tooling Stack
- `ToolsConfig` inspects the active model family and enabled feature flags to decide which capabilities to expose (shell variant, plan tool, apply_patch style, web search, view image, unified exec, and any model-provided experimental tools).【F:codex-rs/core/src/tools/spec.rs†L18-L83】
- Tool schemas are encoded as JSON-Schema fragments; helper constructors define the built-in shell and unified exec tools with parameters for commands, working directories, PTY reuse, and timeouts.【F:codex-rs/core/src/tools/spec.rs†L86-L188】
- `ToolRouter` packages `ResponseItem`s from the model into typed `ToolCall`s (including MCP tool resolution and legacy local shell calls), dispatches them through the registry, and normalizes failures into structured tool outputs so the model can recover gracefully.【F:codex-rs/core/src/tools/router.rs†L20-L190】
- Shell and patch execution flows through `handle_container_exec_with_params`, which enforces approval policy requirements, validates and optionally auto-applies `apply_patch` invocations, streams output through telemetry-friendly truncation, and converts results into success or retry signals for the model.【F:codex-rs/core/src/tools/mod.rs†L8-L244】

## Safety & Guardrails
- `assess_patch_safety` and `assess_command_safety` gate potentially dangerous actions. They evaluate approval policy, sandbox availability, trusted command lists, and user-approved overrides to decide whether to auto-approve, request approval, or reject outright.【F:codex-rs/core/src/safety.rs†L16-L200】
- The tool layer enforces additional safeguards, such as blocking escalated permissions requests unless the session is in `on-request` mode, and wrapping command output so the model always sees truncated, annotated summaries instead of unbounded logs.【F:codex-rs/core/src/tools/mod.rs†L49-L244】
- Model prompts reiterate behavioral guardrails: use `bash -lc`, avoid destructive git commands, respect sandbox settings, and follow strict final-message formatting, ensuring the language model itself reinforces the runtime checks.【F:codex-rs/core/gpt_5_codex_prompt.md†L5-L106】

## Feature Flags & Configuration
- Feature flags centralize experimental toggles (unified exec, streamable shell, plan tool, freeform apply_patch, view image, web search, auto-approval) and track their lifecycle stage. Defaults can be overridden via configuration files or profiles, and legacy toggles are merged for backward compatibility.【F:codex-rs/core/src/features.rs†L1-L180】
- `ToolsConfig` consumes this feature set so tool availability, schema selection, and experimental hooks automatically reflect the user’s config and the model’s capabilities.【F:codex-rs/core/src/tools/spec.rs†L36-L83】

## External Integrations & Telemetry
- During session startup the agent initializes MCP clients, logs authentication status, and records failures so they can be surfaced to users, enabling remote tool ecosystems to plug in cleanly.【F:codex-rs/core/src/codex.rs†L312-L442】
- The `OtelEventManager` attaches telemetry metadata (model info, sandbox settings, account identifiers) so every turn is traced with consistent identifiers, aiding observability and rollout experimentation.【F:codex-rs/core/src/codex.rs†L443-L476】

Together these components form a layered architecture: configuration and prompts define desired behavior, the session/runtime enforce sequencing and safety, the tooling system carries out work under guardrails, and feature flags plus MCP integration let the agent evolve without destabilizing the core experience.

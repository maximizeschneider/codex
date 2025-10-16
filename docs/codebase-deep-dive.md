# Codex Codebase Deep Dive

This document expands on the architecture summary in `docs/agent-architecture.md` and maps the major crates, packages, and integration points that make up the Codex monorepo.

## Repository Layout
- The root README positions Codex as an installable local coding agent for the terminal, highlights supported distribution channels, and links to supplementary documentation for configuration, sandboxing, authentication, automation, and contribution guidelines.【F:README.md†L1-L99】
- The Rust workspace under `codex-rs/` is the actively maintained implementation. Its README explains installation options and enumerates the key crates: `core` (business logic), `exec` (headless runner), `tui` (full-screen UI), and `cli` (multi-tool entry point).【F:codex-rs/README.md†L1-L114】
- `codex-cli/` houses the legacy TypeScript CLI, which remains documented but is marked as superseded by the Rust toolchain.【F:codex-cli/README.md†L1-L120】
- `sdk/typescript/` exposes a programmatic interface to the Rust binary via a thin Node wrapper that streams `codex exec` output.【F:sdk/typescript/src/index.ts†L1-L34】【F:sdk/typescript/src/codex.ts†L1-L38】【F:sdk/typescript/src/exec.ts†L1-L198】

## Core Runtime (`codex-rs/core`)
### Session lifecycle and task orchestration
- `Codex` provides the submission/event queues that front-ends use to interact with the agent. `Codex::spawn` wires configuration, authentication, session services, and the submission loop, producing a session-scoped `TurnContext` that caches prompts, tool availability, sandbox policy, and working directory.【F:codex-rs/core/src/codex.rs†L130-L205】【F:codex-rs/core/src/codex.rs†L240-L340】
- `Session::new` parallelizes startup work—rolling out telemetry capture, establishing MCP connections, detecting the default shell, and loading history metadata—before exposing shared `SessionServices` (executor, MCP manager, notifier, rollout recorder).【F:codex-rs/core/src/codex.rs†L312-L400】
- The `conversation_manager` crate layer creates and resumes `CodexConversation` handles, enforces that `SessionConfigured` is the first event, persists handles for lookups, and supports forking from recorded rollout logs.【F:codex-rs/core/src/conversation_manager.rs†L26-L173】

### Configuration, features, and history
- `Config` aggregates user preferences, model metadata, sandbox policies, notifier hooks, MCP server definitions, and CLI overrides by merging disk layers with profile-specific and command-line overrides.【F:codex-rs/core/src/config.rs†L1-L200】
- Feature flags are centralized in `features.rs`, defining lifecycle stages, default enablement, and override plumbing that merges legacy toggles, profiles, and runtime overrides into a single `Features` set.【F:codex-rs/core/src/features.rs†L1-L180】
- Persistent transcript storage is handled by `message_history.rs`, which appends JSONL records under `~/.codex`, enforces file permissions, and exposes metadata helpers for UI pickers.【F:codex-rs/core/src/message_history.rs†L1-L160】

### Prompting and instruction stack
- `client_common::Prompt` packages conversation history, tool specs, and optional response schemas while dynamically attaching `apply_patch` guidance for models that require it and rewriting shell outputs when the freeform patch tool is active.【F:codex-rs/core/src/client_common.rs†L23-L132】
- The base system prompt (`core/prompt.md`) captures tone, responsiveness, planning etiquette, sandbox awareness, and final-message rules, while `gpt_5_codex_prompt.md` layers on model-specific shell, editing, and approval guidance.【F:codex-rs/core/prompt.md†L1-L149】【F:codex-rs/core/gpt_5_codex_prompt.md†L1-L71】
- Review-specific instructions are embedded via `REVIEW_PROMPT` for tasks that flip the agent into a critique-oriented mode.【F:codex-rs/core/src/client_common.rs†L23-L69】

### Tooling pipeline
- `ToolsConfig` inspects the active model family and feature flags to decide between default, streamable, or unified exec shells; enable the plan, apply_patch, view image, and web search tools; and register experimental tool names.【F:codex-rs/core/src/tools/spec.rs†L18-L83】
- `ToolRouter` materializes configured tool specs, determines parallel-call eligibility, translates model tool messages (including MCP-qualified names), and dispatches invocations through the registry while normalizing failure payloads.【F:codex-rs/core/src/tools/router.rs†L1-L189】
- `handle_container_exec_with_params` is the heart of shell and patch execution: it enforces approval policy, verifies freeform `apply_patch` requests, configures sandbox-aware executors, streams output through truncation guards, and maps process results back into model-facing responses.【F:codex-rs/core/src/tools/mod.rs†L1-L180】
- The executor subsystem links tool invocations to concrete runners via `PreparedExec`, carrying approval metadata and stream configuration into the async execution pipeline.【F:codex-rs/core/src/executor/mod.rs†L1-L44】

### Safety and approvals
- `safety.rs` contains the policy engine that evaluates command and patch requests. It distinguishes trusted commands, enforces sandbox availability, escalates when user approval is required, and rejects unsafe operations under strict policies.【F:codex-rs/core/src/safety.rs†L1-L198】
- Patch-specific checks ensure edits stay within writable paths or acquire explicit consent, even when sandbox tooling is unavailable.【F:codex-rs/core/src/safety.rs†L28-L83】

### Model Context Protocol integration
- `mcp_connection_manager.rs` manages one MCP client per configured server, handles stdio and HTTP transports (including the RMCP client), normalizes fully qualified tool names, and exposes aggregated listings and call helpers with per-server timeouts.【F:codex-rs/core/src/mcp_connection_manager.rs†L1-L160】

## Front-Ends in Rust
- The multitool `codex-rs/cli` crate composes subcommands for interactive TUI sessions, headless execution, login flows, sandbox utilities, MCP management, cloud-task browsing, and protocol generation, all behind a shared Clap interface.【F:codex-rs/cli/src/main.rs†L1-L199】
- The `codex-rs/exec` crate implements non-interactive runs (`codex exec`), handling prompt sourcing, JSONL streaming, tracing setup, sandbox overrides, config resolution, and telemetry initialization before relaying events to human or JSON processors.【F:codex-rs/exec/src/lib.rs†L1-L199】
- The `codex-rs/tui` crate builds the full-screen UI: it locks down stdout/stderr usage, wires tracing, manages update prompts, and exposes CLI flags that influence sandboxing, approvals, and feature toggles before launching the Ratatui application state machine.【F:codex-rs/tui/src/lib.rs†L1-L200】

## Supporting Crates and Utilities
- The `apply-patch` crate parses and validates patch bodies, supports heredoc extraction, exposes canonical tool instructions, and surfaces structured `ApplyPatchAction` data for downstream safety checks and execution.【F:codex-rs/apply-patch/src/lib.rs†L1-L170】
- The shared protocol crate (`codex-rs/protocol`) defines the SQ/EQ message schema, tool call payloads, approval workflows, reasoning settings, and utility tags that bind the Rust runtime, CLI front-ends, and SDKs together.【F:codex-rs/protocol/src/protocol.rs†L1-L200】

## TypeScript Surfaces
- The legacy CLI documentation details installation, sandboxing philosophy, configuration, provider support, and contribution workflow for the Node implementation, clarifying its experimental status relative to the Rust CLI.【F:codex-cli/README.md†L1-L120】
- The TypeScript SDK constructs a `CodexExec` wrapper that spawns the Rust binary with `exec --experimental-json`, forwards authentication and sandbox options, and streams structured output lines back to Node consumers, which are then wrapped by `Codex` and `Thread` abstractions for higher-level usage.【F:sdk/typescript/src/codex.ts†L1-L38】【F:sdk/typescript/src/exec.ts†L1-L198】

## Documentation and Configuration Ecosystem
- The repository-level README curates links to configuration guides, sandbox explanations, automation pathways, installation instructions, and FAQs so contributors can navigate the broader documentation set maintained under `docs/`.【F:README.md†L66-L99】
- Configuration docs referenced throughout the Rust README emphasize that the maintained CLI uses `config.toml`, ties into MCP clients/servers, and supports notifier hooks and sandbox presets aligned with the core runtime’s configuration model.【F:codex-rs/README.md†L16-L105】

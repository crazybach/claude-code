# Claude Code Main Architecture Summary

This repository represents a Bun + TypeScript implementation of a terminal-first agentic coding assistant. The architecture is centered on a **query loop** that combines LLM streaming, tool invocation, permissions, and UI rendering.

## 1) Entry and startup orchestration

- `src/main.tsx` is the launch point for the CLI.
- Startup aggressively overlaps expensive operations (MDM reads, keychain prefetch) before full imports complete.
- It assembles runtime context, initializes services, and then hands control to the REPL launcher.

## 2) Interactive shell + UI runtime

- `src/replLauncher.tsx` lazily imports UI-heavy modules and mounts the app with Ink/React.
- This keeps startup responsive while preserving a component-based architecture similar to web React apps.

## 3) Core engine (conversation + tool loop)

- `src/QueryEngine.ts` owns per-conversation lifecycle/state.
- One `QueryEngine` instance persists conversation state (messages, usage, permission denials, file cache) across turns.
- `submitMessage()` runs the core loop: normalize user input, query model, handle tool calls, stream results, and continue until terminal output is produced.

## 4) Tool contract and execution model

- `src/Tool.ts` defines the normalized tool interfaces and runtime context passed to tool implementations.
- Tool context includes permissions, app state mutation hooks, MCP access, notification hooks, and model/query settings.
- `src/tools.ts` is the tool registry/composition layer:
  - assembles built-in tools,
  - conditionally includes tools using feature flags / environment gates,
  - filters final availability via permission configuration.

## 5) Command system (user-triggered workflows)

- `src/commands.ts` is the command registry for slash commands.
- Commands are loaded as a mix of static and lazy imports, with feature-flag gating for optional surfaces.
- Commands integrate with the same tool/query pipeline, and also load dynamic commands from skills/plugins.

## 6) Extensibility layers

The codebase exposes several extension planes:

- **Skills** (`src/skills/` + docs): markdown-first, workflow-oriented capabilities.
- **Plugins** (`src/plugins/`, `src/services/plugins/`): package-style extensions.
- **MCP integration** (`src/services/mcp/`, plus MCP-related tools): remote tool/resource federation.
- **Bridge** (`src/bridge/`): IDE-connected mode behind feature gates.

## 7) Architectural characteristics

- **Feature-flag-driven binary shaping** using `feature('...')` and conditional requires.
- **Lazy-loading** to reduce cold-start overhead for heavy modules.
- **Permission-first execution** where tool visibility and runtime invocations are policy mediated.
- **Single-threaded async orchestration** with stateful session management, plus optional background/task subsystems.

## 8) Practical mental model

A useful simplified model is:

1. Parse CLI + initialize environment (`main.tsx`)  
2. Mount interactive UI (`replLauncher.tsx`)  
3. For each user turn, run QueryEngine loop (`QueryEngine.ts`)  
4. Resolve and run tools through shared contracts (`Tool.ts`, `tools.ts`)  
5. Surface results in REPL and persist session state

## 9) Notes on this repository

The docs in this repo describe it as a **reference / exploration codebase** with architecture guides intended to map important systems quickly.

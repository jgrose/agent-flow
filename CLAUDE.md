# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Agent Flow is a VS Code extension that visualizes Claude Code agent orchestration in real time. It renders an interactive node graph showing agents, tool calls, subagent coordination, and execution flows. Built by Simon Patole for CraftMyGame.

## Repository Structure

This is a **two-package monorepo** with no root package.json:

- **`extension/`** — VS Code extension (Node.js, TypeScript, CommonJS). Entry point: `extension/src/extension.ts`. Bundled with esbuild to `extension/dist/extension.js`.
- **`web/`** — React UI for the webview (React 19, TypeScript, Tailwind CSS v4). Has two build targets:
  - **Vite** (`vite.config.webview.ts`) builds the webview IIFE bundle to `extension/dist/webview/` (entry: `web/webview-entry.tsx`)
  - **Next.js** for standalone dev/demo site

## Build Commands

All build commands run from `extension/`:

```bash
cd extension && npm run build          # Extension only (esbuild)
cd extension && npm run build:webview  # Webview only (vite, runs in web/)
cd extension && npm run build:all      # Both (webview first, then extension)
cd extension && npm run watch          # Extension watch mode
cd extension && npm run lint           # TypeScript type-check (tsc --noEmit)
cd extension && npm run package        # vsce package for publishing
```

Web dev server (standalone, not embedded in VS Code):
```bash
cd web && pnpm run dev                 # Next.js dev server
```

**Dependencies**: Extension uses `npm`, web uses `pnpm`.

## No Automated Tests

There are no test files or test scripts. The project relies on manual testing and visual QA.

## Architecture

### Event Pipeline

Events flow in one direction: **Claude Code -> Extension -> Webview**

The extension has three event sources that feed into the webview:
1. **HookServer** (`hook-server.ts`) — HTTP server receiving events from Claude Code hooks
2. **SessionWatcher** (`session-watcher.ts`) — Watches `~/.claude/projects/` for JSONL transcript files
3. **JsonlEventSource** (`event-source.ts`) — Direct file watcher for user-specified JSONL logs

When both HookServer and SessionWatcher are active, `extension.ts` deduplicates: SessionWatcher handles orchestrator transcript events, HookServer passes through subagent tool/message events but filters out subagent lifecycle events (spawn/complete) to avoid duplicate nodes.

### Extension Key Modules

- `protocol.ts` — Shared types for all agent events and extension<->webview messages. This is the central type definition file.
- `transcript-parser.ts` — Parses Claude Code JSONL transcripts into `AgentEvent`s
- `session-watcher.ts` — Discovers and monitors multiple concurrent sessions, tracks per-session state (`WatchedSession`)
- `subagent-watcher.ts` — Watches subagent transcript directories for spawned subagents
- `hook-server.ts` — HTTP server for Claude Code hooks
- `hooks-config.ts` — Manages hook configuration in Claude Code settings
- `discovery.ts` — Hook script installation and discovery file management (PID-based port discovery)
- `webview-provider.ts` — VS Code webview lifecycle, CSP, messaging
- `tool-summarizer.ts` — Human-readable summaries of tool calls/results
- `token-estimator.ts` — Token cost estimation for context tracking

### Web/UI Key Modules

- `components/agent-visualizer/index.tsx` — Root visualizer component, orchestrates all panels and canvas
- `components/agent-visualizer/canvas.tsx` — Main 60fps animation loop using HTML5 Canvas
- `components/agent-visualizer/canvas/` — Modular draw functions (`draw-agents.ts`, `draw-edges.ts`, `draw-particles.ts`, `draw-tool-calls.ts`, etc.) plus `hit-detection.ts` and `render-cache.ts`
- `hooks/use-agent-simulation.ts` — Core simulation state management with d3-force physics and event replay
- `hooks/use-vscode-bridge.ts` — Communicates with extension host via `postMessage`
- `hooks/use-canvas-camera.ts` — Viewport panning, zooming, auto-fit
- `hooks/use-canvas-interaction.ts` — Click, drag, context menu on canvas
- `lib/agent-types.ts` — Web-side type definitions for simulation state (agents, tool calls, edges, particles)
- `lib/vscode-bridge.ts` — Low-level VS Code webview API wrapper
- `lib/canvas-constants.ts` — Visual constants (sizes, colors, timing)
- `lib/mock-scenario.ts` — Test data for development/demo mode

### Extension <-> Webview Protocol

Defined in `extension/src/protocol.ts`. Extension sends: `agent-event`, `agent-event-batch`, `connection-status`, `session-list/started/ended/updated`, `reset`, `config`. Webview sends: `ready`, `request-connect`, `request-disconnect`, `open-file`, `log`.

### Key Patterns

- **Event-driven**: Async event emitters throughout, all state updates flow from events
- **Ref-based state in canvas**: Simulation state stored in React refs (not state) to avoid re-renders during 60fps animation
- **Deduplication**: Tool use IDs track request/result pairs; message hashes prevent duplicates from context compression; `spawnedSubagents` set prevents duplicate spawn events between hook server and session watcher
- **Graceful degradation**: Works without Claude Code if given a JSONL file; mock data mode for development
- **Multi-session**: Independent state per session with cache on tab switch

## Development Workflow

Set `agentVisualizer.devServerPort` to `3002` in VS Code settings to load the webview from the Next.js dev server (hot reload) instead of the built bundle. Run `cd web && pnpm run dev` alongside.

For production testing, run `cd extension && npm run build:all` then press F5 in VS Code to launch the Extension Development Host.

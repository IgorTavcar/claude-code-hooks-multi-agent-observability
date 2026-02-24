# Claude Code Integration Report

## Architecture Overview

```
Claude Code CLI
  |
  |-- (12 hook event types via settings.json)
  |
  v
Python Hook Scripts (.claude/hooks/*.py)
  |
  |-- 1. Local logging (logs/<session_id>/*.json)
  |-- 2. HTTP POST to observability server
  |
  v
Bun Server (apps/server/, port 4000)
  |-- SQLite (events.db) for persistence
  |-- WebSocket broadcast (/stream)
  |
  v
Vue 3 Client (apps/client/, port 5173)
  |-- Real-time event timeline
  |-- Live pulse chart
  |-- Agent swim lanes
  |-- Filtering & theming
```

---

## Control Flow

### 1. Hook Trigger (Claude Code -> Python)

Claude Code fires events at defined lifecycle points. Each event type is configured in `.claude/settings.json` with **two hooks** per event:

1. **Domain-specific hook** (e.g., `pre_tool_use.py`) - performs local logging, validation, and optionally blocks actions
2. **`send_event.py`** - universal forwarder that POSTs the event payload to the Bun server

All hooks receive JSON on **stdin** from Claude Code and must exit `0` to avoid blocking the agent.

### 2. Event Forwarding (Python -> Server)

`send_event.py` (`.claude/hooks/send_event.py:58-178`) is the central dispatcher:
- Reads hook data from stdin
- Extracts model name from transcript via `model_extractor.py`
- Promotes event-specific fields to top-level (`tool_name`, `agent_id`, `notification_type`, etc.)
- Optionally generates AI summaries via `summarizer.py` (uses Anthropic API)
- Optionally attaches chat transcript (`--add-chat` flag)
- POSTs to `http://localhost:4000/events`

### 3. Server Processing (Bun Server)

`apps/server/src/index.ts:124-159`:
- Validates required fields (`source_app`, `session_id`, `hook_event_type`, `payload`)
- Inserts into SQLite via `db.ts`
- Broadcasts to all connected WebSocket clients

### 4. Client Display (Vue App)

`apps/client/src/composables/useWebSocket.ts`:
- Connects to `ws://localhost:4000/stream`
- Receives `initial` (bulk history) and `event` (live) messages
- Events flow into `App.vue` -> `EventTimeline.vue` -> `EventRow.vue`

---

## Agent Lifecycle

### Session Start
- `SessionStart` hook fires -> `session_start.py` logs session metadata (session_id, source, model, agent_type)
- Optionally loads development context (git branch, uncommitted changes, TODO files)
- `send_event.py` forwards to server

### Active Operation
For each tool use cycle:
1. `PreToolUse` -> `pre_tool_use.py` validates (blocks dangerous `rm -rf`, logs tool summary) -> `send_event.py` forwards
2. Tool executes
3. `PostToolUse` -> `post_tool_use.py` logs result (detects MCP tools) -> `send_event.py` forwards
4. On failure: `PostToolUseFailure` fires instead

Other mid-session events:
- `UserPromptSubmit` - captures each user prompt, optionally validates/blocks, generates agent names
- `Notification` - tracks idle prompts, permission prompts with optional TTS announcements
- `PermissionRequest` - logs permission decisions
- `PreCompact` - fires before context window compaction, optionally backs up transcript

### Subagent Lifecycle
- `SubagentStart` -> `subagent_start.py` logs the full input payload
- `SubagentStop` -> `subagent_stop.py` logs with `agent_id`, `agent_type`, `agent_transcript_path`; has `stop_hook_active` guard to prevent infinite loops

### Session End
- `Stop` -> `stop.py` logs session stop, optionally copies transcript to `chat.json`, optional TTS announcement with LLM-generated message
- `SessionEnd` -> `session_end.py` logs with reason (`clear`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`)

---

## Most Important Scripts

| Script | Purpose |
|--------|---------|
| `.claude/settings.json` | Central hook configuration - defines all 12 event hooks, status line, env vars |
| `.claude/hooks/send_event.py` | Universal event forwarder to observability server (the backbone) |
| `.claude/hooks/pre_tool_use.py` | Safety guard - blocks dangerous commands, logs tool summaries |
| `.claude/hooks/stop.py` | Session completion - transcript capture, TTS, stop_hook_active guard |
| `.claude/hooks/user_prompt_submit.py` | Prompt logging, session data management, agent naming |
| `.claude/hooks/utils/summarizer.py` | AI-powered event summarization using Anthropic API |
| `.claude/hooks/utils/model_extractor.py` | Extracts model name from JSONL transcripts with caching |
| `.claude/hooks/utils/constants.py` | Session log directory management |
| `.claude/hooks/utils/hitl.py` | Human-in-the-loop WebSocket server for interactive decisions |
| `apps/server/src/index.ts` | Bun HTTP/WebSocket server - receives events, stores in SQLite, broadcasts |
| `apps/server/src/db.ts` | SQLite database layer with migrations and HITL support |
| `apps/client/src/composables/useWebSocket.ts` | Client-side WebSocket connection with auto-reconnect |
| `apps/client/src/App.vue` | Main UI orchestrator - timeline, charts, swim lanes, themes |
| `scripts/start-system.sh` | Launches server + client, kills conflicting ports |
| `scripts/reset-system.sh` | Stops all processes, cleans up WAL files |
| `justfile` | Task runner with recipes for all common operations |

---

## tmux-Related Scripts

**There are no tmux scripts in this codebase.** The README mentions that multi-agent teams are "deployed into their own Tmux panes" but this is handled by **Claude Code's built-in Agent Teams feature** (enabled via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json), not by custom scripts in this repo.

The orchestration workflow (TeamCreate -> TaskCreate -> Task -> SendMessage -> TeamDelete) is managed by Claude Code's native team tools, and the observability system simply **observes** these events through hooks like `SubagentStart` and `SubagentStop`.

The `start-system.sh` and `reset-system.sh` scripts manage the **observability infrastructure** (server + client), not the agents themselves.

---

## Architecture Patterns

### Dual-Hook Pattern
Every event type has two hooks in `settings.json`:
1. **Local handler** - validation, logging, side effects (TTS, blocking)
2. **`send_event.py`** - forwards to the centralized server

This separation means local functionality works even without the server running.

### Agent Identification
Per `CLAUDE.md`: agents are identified by `source_app:session_id` (session_id truncated to 8 chars). The `source_app` is set via `--source-app` flag on `send_event.py`.

### Fail-Safe Design
All hooks exit `0` on any error to never block Claude Code operations. The `stop_hook_active` guard in `stop.py:162` and `subagent_stop.py:103` prevents infinite hook recursion.

### Human-in-the-Loop (HITL)
`utils/hitl.py` implements a bidirectional HITL system:
1. Hook starts a local WebSocket server on a random port
2. Sends event to observability server with `humanInTheLoop.responseWebSocketUrl`
3. User responds via the Vue client
4. Server connects to agent's WebSocket and delivers the response

---

## Missing Pieces / Issues

### 1. No `test-system.sh` Script
`start-system.sh:103` references `./scripts/test-system.sh` and README mentions it, but only `start-system.sh` and `reset-system.sh` exist in `scripts/`.

### 2. HITL Not Wired Into Hooks
`utils/hitl.py` exists as a library but no hook script actually imports or uses it. The HITL flow requires hooks to call `HITLRequest.send_and_wait()`, but none do. There is a `test_hitl.py` and `examples/hitl_example.py` but these aren't referenced from `settings.json`.

### 3. `ollama.py` LLM Utility Missing
`user_prompt_submit.py:93` falls back to `utils/llm/ollama.py` for agent naming, but only `oai.py` and `anth.py` exist in `utils/llm/`.

### 4. Demo Agent Hooks Are a Partial Copy
`apps/demo-cc-agent/.claude/hooks/` contains a subset of the main hooks but is missing several (session_start, session_end, pre_compact, subagent_start, permission_request, post_tool_use_failure, pre_compact). The demo's `settings.json` was not examined but likely references hooks that don't exist there.

### 5. No Automated Tests
No test files exist for hooks, server, or client. The `justfile` has `test-event` for manual testing but no unit/integration test suite.

### 6. `.env` File Protection Commented Out
`pre_tool_use.py:325-327` has the `.env` file access protection commented out with a note about worktree commands needing `.env` files. This leaves sensitive files unprotected.

### 7. Session Data Persistence Gap
`session_start.py` writes to `logs/session_start.json` (flat file accumulation), while `user_prompt_submit.py` writes to `.claude/data/sessions/<session_id>.json`. Two different session tracking systems coexist without integration.

### 8. No Database Cleanup / Retention Policy
`events.db` grows indefinitely. There's `just db-reset` to nuke it entirely but no mechanism for time-based retention or archival.

### 9. Model Extractor Caching Disabled
`model_extractor.py:12` has `ENABLE_CACHING = False`, meaning every hook invocation reads the entire transcript file to extract the model name. This could be slow for long sessions.

### 10. Summarizer Has No Fallback
`summarizer.py` calls `prompt_llm()` from `utils/llm/anth.py` with no fallback if the Anthropic API is unavailable. If `--summarize` is used and the API fails, the summary is silently `None`.

### 11. Client Config Assumes localhost
`apps/client/src/config.ts` hardcodes `localhost` for both API and WebSocket URLs. No support for remote server deployment without env var overrides.

---

## Vue Client Analysis

### Component Hierarchy

```
App.vue (orchestrator)
├── FilterPanel.vue          — dropdown filters (source_app, session_id, event_type)
├── LivePulseChart.vue       — canvas-based real-time bar chart
├── EventTimeline.vue        — scrollable event list with search
│   └── EventRow.vue         — individual event card (regular + HITL variants)
│       └── ChatTranscriptModal.vue  — modal for viewing chat transcripts
│           └── ChatTranscript.vue   — message renderer (user/assistant/system/tool)
├── AgentSwimLaneContainer.vue — wrapper for per-agent charts
│   └── AgentSwimLane.vue    — per-agent activity chart with metrics
├── ThemeManager.vue         — theme selection modal
│   └── ThemePreview.vue     — live color preview
├── ToastNotification.vue    — new-agent detection toasts
└── StickScrollButton.vue    — auto-scroll toggle
```

### Composables

| Composable | Purpose |
|------------|---------|
| `useWebSocket.ts` | WebSocket connection to server with auto-reconnect (3s). Handles `initial` (bulk history) and `event` (live) messages. Max 300 events (`VITE_MAX_EVENTS_TO_DISPLAY`). |
| `useChartData.ts` | Aggregates events into time-bucketed bar chart data. Time ranges: 1m/3m/5m/10m with bucket sizes 1s/3s/5s/10s. Tracks `uniqueAgentIdsInWindow`, `toolCallCount`, `eventTimingMetrics` (min/max/avg gap). Debounced at 50ms. |
| `useEventColors.ts` | Deterministic color assignment via hash function (seed 7151). Tailwind palette for sessions, HSL generation for apps (`hsl(hash%360, 70%, 50%)`). |
| `useEventEmojis.ts` | Emoji maps for 12 event types and 20+ tool names. `formatEventTypeLabel()` creates combo emojis (e.g., `🔧💻` for PreToolUse:Bash). MCP tools get `🔌` prefix. |
| `useEventSearch.ts` | Regex-based event search. Extracts searchable text from event fields. Validates regex patterns with error feedback. |
| `useHITLNotifications.ts` | Browser Notification API for HITL requests. `requireInteraction: true` keeps notifications visible until dismissed. |
| `useThemes.ts` | Theme CRUD operations via REST API. Manages current theme state and CSS variable application. |
| `useAgentChartData.ts` | Per-agent chart data (used by AgentSwimLane). Filters events by agent ID before bucketing. |
| `useMediaQuery.ts` | Reactive CSS media query matching. |

### Component Details

#### App.vue — Orchestrator
- Wires all composables together: `useWebSocket(WS_URL)`, `useThemes()`, `useEventColors()`
- Manages global state: filters, stickToBottom, showThemeManager, selectedAgentLanes
- Watches `uniqueAppNames` to fire toast notifications when new agents appear
- Passes filtered events down to EventTimeline, chart data to LivePulseChart

#### EventTimeline.vue — Event List
- Scrollable container with `TransitionGroup` animations (fade + slide)
- Agent tag bar at top showing active (✨) vs sleeping (😴) agents based on last-seen timestamps
- Regex search bar that filters events via `useEventSearch`
- Auto-scrolls to bottom when `stickToBottom` is true (watches event count changes)
- Emits `selectAgent` to toggle swim lane visibility

#### EventRow.vue — Event Card
Two rendering paths:

1. **Regular events**: Expandable card with dual-color left border (app color + session color), tool info badge, summary text, expandable JSON payload viewer, chat transcript button
2. **HITL events**: Interactive UI per type — text input for questions, approve/deny buttons for permissions, radio buttons for choices. Optimistic updates with rollback on API failure via `POST /events/:id/respond`

Model name formatting: strips `claude-` prefix and date suffix (e.g., `claude-haiku-4-5-20251001` → `haiku-4-5`).

#### LivePulseChart.vue — Real-time Chart
- Canvas-based rendering at 30 FPS via `requestAnimationFrame`
- Shows metrics header: agents count, event count, tool call count, avg gap
- Time range selector (1m/3m/5m/10m) with left/right arrow keyboard navigation
- Pulse animation on new events (ring expanding outward)
- `ResizeObserver` for responsive sizing, `MutationObserver` for theme changes (re-reads CSS variables)
- Uses `createChartRenderer` from `useChartData` for the actual bar drawing

#### AgentSwimLane.vue — Per-Agent Chart
- Shows `app:session` split label, model badge, per-agent event/tool/timing metrics
- Filters events by matching `source_app` and `session_id.slice(0,8)`
- Own canvas renderer at 30 FPS, same bar chart approach as LivePulseChart
- Receives agent ID from parent, computes its own chart data via `useAgentChartData`

#### ChatTranscriptModal.vue — Transcript Viewer
- Teleported modal (`85vw × 85vh`) with search and role filtering
- Filter buttons for: User, Assistant, System, Tool Use, Tool Result, Read, Write, Edit, Glob
- Text search across message content, roles, tool names, UUIDs
- ESC key closes, click-outside closes

#### ChatTranscript.vue — Message Renderer
- Renders messages by role: User (blue), Assistant (green), System (gray)
- Handles two formats: simple `{role, content}` and complex `{type, message}` structures
- ANSI code stripping for system messages (`/\x1B\[[0-9;]*m/g`)
- Per-message copy button and expandable JSON detail viewer

#### FilterPanel.vue — Filters
- Three dropdown filters: Source App, Session ID, Event Type
- Fetches available options from `GET /events/filter-options` every 10 seconds
- Emits filter changes to App.vue for event filtering

#### ThemeManager.vue / ThemePreview.vue — Theming
- Theme grid modal (teleported) with color preview bars
- Current theme indicator, click to switch
- ThemePreview shows simulated event card, filter panel, color palette, status indicators

#### ToastNotification.vue — Agent Detection
- Centered toast for new agent detection
- Auto-dismisses after 4 seconds
- Stacks vertically with index-based positioning

#### StickScrollButton.vue — Scroll Control
- Fixed bottom-right toggle button
- Different SVG icons for enabled (arrow down) vs disabled (pause) states

### Data Flow

```
WebSocket (/stream)
  │
  ▼
useWebSocket.ts → reactive events[]
  │
  ├──► App.vue filters events by FilterPanel selections
  │     │
  │     ├──► EventTimeline.vue (filtered + searched events)
  │     │     └──► EventRow.vue × N
  │     │
  │     └──► useChartData.ts (time-bucketed aggregation)
  │           ├──► LivePulseChart.vue (global chart)
  │           └──► AgentSwimLane.vue × N (per-agent charts)
  │
  └──► useHITLNotifications.ts → browser notifications
```

### Client Issues Found

#### 1. HelloWorld.vue is Unused
Default Vite/Vue scaffold component still in the codebase. Not referenced by any parent.

#### 2. Hard 300-Event Cap
`useWebSocket.ts` caps at 300 events (`VITE_MAX_EVENTS_TO_DISPLAY`). For long-running multi-agent sessions this means older events silently disappear from the timeline. No pagination or "load more" mechanism exists.

#### 3. Canvas Charts Don't Clean Up Consistently
Both `LivePulseChart.vue` and `AgentSwimLane.vue` create `ResizeObserver` and `MutationObserver` instances. While they do clean up in `onUnmounted`, the animation frame loop (`requestAnimationFrame`) is cancelled via a flag rather than storing the frame ID, which could theoretically leak one extra frame.

#### 4. FilterPanel Polling
`FilterPanel.vue` polls `GET /events/filter-options` every 10 seconds via `setInterval`. Since the WebSocket already provides real-time events, filter options could be derived client-side from the events array instead of polling the server.

#### 5. HITL Response Has No Timeout
`EventRow.vue` HITL interactions have no client-side timeout. If the agent's WebSocket server has already shut down by the time the human responds, the `POST /events/:id/respond` will fail silently (the server attempts to connect to a dead port). No retry or "agent no longer listening" feedback is shown.

#### 6. Agent Activity Detection is Time-Based Only
`EventTimeline.vue` marks agents as active (✨) vs sleeping (😴) based on whether their last event was within 30 seconds. This doesn't account for agents that are genuinely working but on a long-running tool (e.g., a multi-minute build). A heartbeat mechanism would be more accurate.

#### 7. No Error Boundary
No Vue error boundary or `onErrorCaptured` handler exists. A rendering error in any component (e.g., malformed event data) would crash the entire app.

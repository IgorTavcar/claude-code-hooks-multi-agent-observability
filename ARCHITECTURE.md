# Architecture: Multi-Agent Observability System

## Table of Contents

- [1. System Overview](#1-system-overview)
- [2. The Agentic Society](#2-the-agentic-society)
- [3. Infrastructure Layers](#3-infrastructure-layers)
- [4. Hook Event Lifecycle](#4-hook-event-lifecycle)
- [5. Multi-Agent Orchestration](#5-multi-agent-orchestration)
- [6. Inter-Agent Communication](#6-inter-agent-communication)
- [7. Agent Roles & Specialization](#7-agent-roles--specialization)
- [8. Observability Pipeline](#8-observability-pipeline)
- [9. Human-in-the-Loop (HITL)](#9-human-in-the-loop-hitl)
- [10. Security Model](#10-security-model)
- [11. Data Model](#11-data-model)
- [12. Client Architecture](#12-client-architecture)
- [13. Configuration Reference](#13-configuration-reference)
- [14. Official Documentation References](#14-official-documentation-references)

---

## 1. System Overview

This system provides **real-time observability** over Claude Code sessions, including multi-agent teams. It captures every lifecycle event from every agent through Claude Code's hook system, forwards them to a centralized server, and renders them in a live dashboard.

```
                              The Agentic Society
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │   Lead Agent              Builder Agent         Validator Agent  │
  │   ┌──────────┐           ┌──────────┐          ┌──────────┐    │
  │   │ TeamCreate│──spawn──>│  Task     │          │  Task     │    │
  │   │ TaskCreate│          │  Execute  │──done──> │  Verify   │    │
  │   │ SendMsg   │<──msg────│  Report   │          │  Report   │    │
  │   └────┬─────┘           └────┬─────┘          └────┬─────┘    │
  │        │                      │                      │          │
  └────────┼──────────────────────┼──────────────────────┼──────────┘
           │ hooks                │ hooks                │ hooks
           ▼                      ▼                      ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Hook Scripts (Python)                         │
  │  12 event types × dual-hook pattern (handler + send_event.py)   │
  └──────────────────────────┬──────────────────────────────────────┘
                             │ HTTP POST localhost:4000
                             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Bun Server (SQLite + WS)                     │
  │  Receives events, persists to SQLite, broadcasts via WebSocket  │
  └──────────────────────────┬──────────────────────────────────────┘
                             │ WebSocket ws://localhost:4000/stream
                             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Vue 3 Dashboard                              │
  │  Real-time timeline, pulse chart, swim lanes, HITL UI, themes   │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 2. The Agentic Society

Multi-agent orchestration in Claude Code follows a **society** metaphor. Agents are not isolated workers — they form structured teams with defined roles, communication channels, shared task boards, and lifecycle governance.

### 2.1 Society Structure

```
                    ┌──────────────┐
                    │  Human User  │
                    │  (Governor)  │
                    └──────┬───────┘
                           │ prompts, approvals, HITL responses
                           ▼
                    ┌──────────────┐
                    │  Lead Agent  │
                    │  (Director)  │
                    └──────┬───────┘
                           │ TeamCreate, TaskCreate, SendMessage
                    ┌──────┼──────────────────┐
                    ▼      ▼                  ▼
             ┌──────────┐ ┌──────────┐ ┌──────────┐
             │ Builder  │ │ Builder  │ │ Validator │
             │ (Worker) │ │ (Worker) │ │ (Auditor) │
             └──────────┘ └──────────┘ └──────────┘
```

**Roles in the society:**

| Role | Analogy | Responsibility | Tools |
|------|---------|---------------|-------|
| **Human User** | Governor | Sets goals, approves plans, responds to HITL requests | Browser dashboard |
| **Lead Agent** | Director | Creates teams, decomposes work into tasks, assigns agents, coordinates | TeamCreate, TaskCreate, SendMessage, Task |
| **Builder** | Worker | Executes ONE task: writes code, creates files, implements features | All tools (Write, Edit, Bash, etc.) |
| **Validator** | Auditor | Verifies work meets acceptance criteria. Read-only — cannot modify files | Read, Glob, Grep, Bash (read-only) |
| **Scout** | Analyst | Investigates codebase issues, identifies problems, suggests fixes. Read-only | Read, Glob, Grep |
| **Meta-Agent** | Architect | Creates new agent definitions from descriptions | Write, WebFetch |

### 2.2 Agent Identity

Every agent in the society is uniquely identified by its **source_app** and **session_id**:

```
Agent ID = source_app : session_id[0:8]
Example  = cc-hook-multi-agent-obvs : 2176a933
```

This identity is:
- Assigned at session creation by Claude Code
- Passed via stdin JSON to every hook invocation
- Used for color assignment in the dashboard
- Used for event correlation and swim lane grouping
- Consistent across the agent's entire lifetime

### 2.3 Agent Lifecycle

```
SessionStart ──> Active Operation ──> Stop/SessionEnd
                      │
                      ├── UserPromptSubmit (each user message)
                      ├── PreToolUse ──> PostToolUse (each tool call)
                      ├── PreToolUse ──> PostToolUseFailure (on error)
                      ├── Notification (idle/permission prompts)
                      ├── PermissionRequest (permission decisions)
                      ├── SubagentStart (spawning child agents)
                      ├── SubagentStop (child agent completion)
                      └── PreCompact (context window compaction)
```

---

## 3. Infrastructure Layers

### 3.1 Hook Scripts (Python)

**Location:** `.claude/hooks/*.py`

Every hook event type has **two hooks** configured in `.claude/settings.json`:

1. **Domain handler** — performs local validation, logging, side effects (TTS, blocking)
2. **`send_event.py`** — universal forwarder that POSTs the event to the Bun server

This **dual-hook pattern** means local functionality (logging, safety guards, TTS) works even when the server is down.

```json
"PreToolUse": [
  {
    "hooks": [
      { "command": "uv run .claude/hooks/pre_tool_use.py" },
      { "command": "uv run .claude/hooks/send_event.py --source-app cc-hook-multi-agent-obvs --event-type PreToolUse --summarize" }
    ]
  }
]
```

### 3.2 Observability Server (Bun + SQLite)

**Location:** `apps/server/src/`

- **Runtime:** Bun (TypeScript) — chosen for built-in WebSocket, SQLite, and HTTP server support
- **Database:** SQLite with WAL mode for concurrent read/write
- **Binding:** `127.0.0.1:4000` (configurable via `SERVER_HOST`, `SERVER_PORT`)
- **CORS:** Restricted to configured origins (default: `localhost:5173,localhost:4000`)

**Endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/events` | Receive hook events, store in SQLite, broadcast via WebSocket |
| `GET` | `/events/recent?limit=N` | Fetch recent events |
| `GET` | `/events/filter-options` | Available filter values (apps, sessions, event types) |
| `POST` | `/events/:id/respond` | Submit HITL response to an agent |
| `WS` | `/stream` | WebSocket endpoint for real-time event streaming |
| `*` | `/api/themes/*` | Theme CRUD API |

### 3.3 Dashboard Client (Vue 3)

**Location:** `apps/client/src/`

- **Framework:** Vue 3 with Composition API
- **Styling:** Tailwind CSS with dynamic theming
- **Charts:** Canvas-based rendering at 30 FPS
- **Connection:** WebSocket to server for real-time events

---

## 4. Hook Event Lifecycle

### 4.1 The 12 Event Types

| Event | When | Key Data |
|-------|------|----------|
| **SessionStart** | Agent session begins | `agent_type`, `source`, `model` |
| **SessionEnd** | Agent session ends | `reason` (clear, logout, exit, bypass_disabled) |
| **UserPromptSubmit** | Each user/lead message | Prompt text |
| **PreToolUse** | Before tool execution | `tool_name`, `tool_input` |
| **PostToolUse** | After tool succeeds | `tool_name`, `tool_result` |
| **PostToolUseFailure** | After tool fails | `tool_name`, `error`, `is_interrupt` |
| **PermissionRequest** | Permission decision point | `tool_name`, `permission_suggestions` |
| **Notification** | Idle/permission prompts | `notification_type` |
| **SubagentStart** | Child agent spawned | `agent_id`, `agent_type` |
| **SubagentStop** | Child agent completes | `agent_id`, `agent_type`, `agent_transcript_path` |
| **Stop** | Session stopping | `stop_hook_active` guard, optional transcript |
| **PreCompact** | Context window compaction | `transcript_path`, `custom_instructions` |

### 4.2 Event Data Flow

```
Claude Code fires event
       │
       ▼
stdin JSON ──> Domain Handler (e.g., pre_tool_use.py)
       │           │
       │           ├── Local logging (logs/<session_id>/*.json)
       │           ├── Safety validation (block dangerous commands)
       │           ├── Side effects (TTS announcements)
       │           └── Exit 0 (never block Claude Code)
       │
       ▼
stdin JSON ──> send_event.py
                   │
                   ├── Extract model name from transcript
                   ├── Promote event-specific fields to top level
                   ├── Optionally generate AI summary (--summarize)
                   ├── Optionally attach chat transcript (--add-chat)
                   │
                   ▼
            HTTP POST localhost:4000/events
                   │
                   ▼
            Bun Server
                   ├── INSERT into SQLite (events table)
                   └── Broadcast to all WebSocket clients
                              │
                              ▼
                        Vue Dashboard (real-time update)
```

### 4.3 Fail-Safe Design

All hooks follow a critical safety contract:

- **Always exit 0** — a hook failure must never block Claude Code operation
- **`stop_hook_active` guard** — prevents infinite recursion when Stop hooks themselves trigger tool calls
- **5-second HTTP timeout** — `send_event.py` will not hang if the server is down
- **Silent error handling** — failures are logged to stderr but never raised

---

## 5. Multi-Agent Orchestration

### 5.1 Team Formation

Teams are created by the lead agent using Claude Code's native team tools. The `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` env var (set in `.claude/settings.json`) enables these capabilities.

```
Lead Agent                              System
    │                                      │
    ├── TeamCreate("project-x")  ────────> │ Creates ~/.claude/teams/project-x/config.json
    │                                      │ Creates ~/.claude/tasks/project-x/
    │                                      │
    ├── TaskCreate(subject, desc)  ──────> │ Adds task to task list (status: pending)
    ├── TaskCreate(subject, desc)  ──────> │ Adds task to task list (status: pending)
    ├── TaskCreate(subject, desc)  ──────> │ Adds task to task list (status: pending)
    │                                      │
    ├── Task(prompt, subagent_type)  ───> │ Spawns Builder agent (new session)
    ├── Task(prompt, subagent_type)  ───> │ Spawns Builder agent (new session)
    ├── Task(prompt, subagent_type)  ───> │ Spawns Validator agent (new session)
    │                                      │
```

**Team config** (`~/.claude/teams/{team-name}/config.json`):
```json
{
  "members": [
    { "name": "builder-1", "agentId": "abc-123", "agentType": "builder" },
    { "name": "builder-2", "agentId": "def-456", "agentType": "builder" },
    { "name": "validator",  "agentId": "ghi-789", "agentType": "validator" }
  ]
}
```

### 5.2 Task Coordination

Tasks are the shared work units that coordinate the society:

```
┌─────────────────────────────────────────────────────┐
│                   Shared Task Board                  │
│  ~/.claude/tasks/{team-name}/                        │
│                                                      │
│  #1 [completed] "Implement auth API"    owner:build1 │
│  #2 [in_progress] "Add login UI"        owner:build2 │
│  #3 [pending] "Validate auth flow"      blocked_by:1 │
│  #4 [pending] "Write API tests"         blocked_by:1 │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Task lifecycle:**

```
pending ──> in_progress ──> completed
                │
                └── (if blocked) stays in_progress, agent reports blockers
```

**Task dependencies** (`blockedBy` / `blocks`) enable sequencing:
- Task #3 "Validate auth flow" is `blockedBy: [1]` — it can't start until #1 completes
- When task #1 completes, task #3 becomes unblocked and available for claiming

**Agent workflow:**
1. Agent reads `TaskList` to find available work (pending, unblocked, no owner)
2. Agent claims task via `TaskUpdate` (sets `owner` to its name)
3. Agent reads full details via `TaskGet`
4. Agent does the work
5. Agent marks `TaskUpdate` status: `completed`
6. Agent checks `TaskList` for next available task

### 5.3 Parallel Execution

Agents execute in parallel with independent context windows. Each agent:

- Has its own Claude Code session (unique `session_id`)
- Operates in its own context window
- Can read/write files in the shared repository
- Communicates only through the task board and messages

```
Time ──────────────────────────────────────────────>

Lead:     [create team] [create tasks] [spawn agents] ... [shutdown]
Builder1: ................[claim #1] [implement] [complete]
Builder2: ................[claim #2] [implement] [complete]
Validator: ..............................[claim #3] [verify] [report]
```

---

## 6. Inter-Agent Communication

### 6.1 Message Types

Agents communicate via the `SendMessage` tool, which supports three message types:

#### Direct Message (1:1)
```
SendMessage {
  type: "message",
  recipient: "builder-1",          // target agent by name
  content: "Task #1 is ready...",  // message body
  summary: "Auth API task ready"   // 5-10 word preview
}
```

#### Broadcast (1:all)
```
SendMessage {
  type: "broadcast",
  content: "Blocking bug found in shared module...",
  summary: "Critical blocking issue found"
}
```
Broadcasts are **expensive** (N messages for N teammates). Reserved for critical, team-wide announcements.

#### Shutdown Protocol
```
// Leader requests shutdown
SendMessage {
  type: "shutdown_request",
  recipient: "builder-1",
  content: "All tasks complete, wrapping up"
}

// Agent responds
SendMessage {
  type: "shutdown_response",
  request_id: "abc-123",
  approve: true
}
```

### 6.2 Message Delivery

- Messages are **automatically delivered** to teammates (no polling required)
- If a teammate is mid-turn (busy), messages are **queued** and delivered when their turn ends
- Teammates go **idle** after every turn — this is normal, not an error
- Idle teammates can receive messages and **wake up** to process them

### 6.3 Communication Patterns

**Status reporting:**
```
Builder ──SendMessage──> Lead: "Task #1 complete, 3 files changed"
```

**Coordination:**
```
Builder1 ──SendMessage──> Builder2: "I changed the auth module, update your imports"
```

**Escalation:**
```
Builder ──SendMessage──> Lead: "Blocked on missing API spec, need clarification"
```

**Plan approval:**
```
Builder ──ExitPlanMode──> Lead: [plan approval request]
Lead ──plan_approval_response──> Builder: approved / rejected with feedback
```

---

## 7. Agent Roles & Specialization

### 7.1 Agent Definitions

Agent roles are defined as markdown files in `.claude/agents/`:

```
.claude/agents/
├── team/
│   ├── builder.md       # Full-capability engineering agent
│   └── validator.md     # Read-only verification agent
├── scout-report-suggest.md     # Codebase analysis agent
├── scout-report-suggest-fast.md # Quick analysis variant
├── meta-agent.md        # Agent definition generator
├── docs-scraper.md      # Documentation fetcher
├── create_worktree_subagent.md # Worktree management
├── fetch-docs-haiku45.md       # Docs fetch (Haiku model)
└── fetch-docs-sonnet45.md      # Docs fetch (Sonnet model)
```

### 7.2 Builder Agent

**File:** `.claude/agents/team/builder.md`

- **Purpose:** Executes ONE task at a time — writes code, creates files, implements features
- **Model:** Opus (most capable)
- **Tools:** All tools available
- **Post-tool hooks:** Runs `ruff_validator.py` and `ty_validator.py` after Write/Edit operations
- **Workflow:** TaskGet -> Execute -> Verify -> TaskUpdate(completed)

### 7.3 Validator Agent

**File:** `.claude/agents/team/validator.md`

- **Purpose:** Verifies that a task was completed correctly. Read-only — cannot modify files
- **Model:** Opus
- **Disallowed tools:** Write, Edit, NotebookEdit
- **Workflow:** TaskGet -> Inspect files -> Run checks -> TaskUpdate(completed with pass/fail report)

### 7.4 Scout Agent

**File:** `.claude/agents/scout-report-suggest.md`

- **Purpose:** Investigates codebase problems, identifies root causes, suggests resolutions
- **Tools:** Read, Glob, Grep (read-only)
- **Output:** Structured SCOUT REPORT with findings, root cause analysis, and resolution suggestions

### 7.5 Meta-Agent

**File:** `.claude/agents/meta-agent.md`

- **Purpose:** Generates new agent definitions from user descriptions
- **Model:** Opus
- **Workflow:** Scrapes latest Claude Code docs -> analyzes requirements -> generates `.claude/agents/<name>.md`

---

## 8. Observability Pipeline

### 8.1 send_event.py — The Backbone

`send_event.py` is the universal event forwarder that every hook type uses:

```
stdin JSON from Claude Code
       │
       ├── Parse JSON from stdin
       ├── Extract session_id
       ├── Extract model name from transcript (model_extractor.py)
       │       └── Reads .jsonl transcript in reverse to find model field
       │
       ├── Build event payload:
       │   {
       │     source_app: "cc-hook-multi-agent-obvs",
       │     session_id: "2176a933-...",
       │     hook_event_type: "PreToolUse",
       │     payload: { ...raw hook data... },
       │     timestamp: 1708789200000,
       │     model_name: "claude-opus-4-6",
       │     tool_name: "Bash",           // promoted from payload
       │     agent_id: "abc-123",         // promoted from payload
       │   }
       │
       ├── [--summarize] Generate AI summary via Anthropic Haiku
       │       └── Sends truncated payload (max 1000 chars) to Haiku
       │           Returns one-sentence summary under 15 words
       │
       ├── [--add-chat] Attach full chat transcript
       │       └── Reads .jsonl transcript, converts to JSON array
       │           Only for Stop events (full session history)
       │
       └── HTTP POST to localhost:4000/events (5s timeout)
```

### 8.2 Domain Handlers

Each event type has a specialized handler:

| Handler | Key Behaviors |
|---------|--------------|
| `pre_tool_use.py` | Blocks dangerous commands, logs tool summaries per type, handles 20+ tool types including Task, SendMessage, TaskCreate, TeamCreate |
| `post_tool_use.py` | Logs tool results, detects MCP tools (splits `mcp__server__tool` names) |
| `stop.py` | Session completion, transcript copy to `chat.json`, TTS announcement, `stop_hook_active` guard |
| `subagent_stop.py` | Tracks `agent_id`, `agent_type`, `agent_transcript_path`, same `stop_hook_active` guard |
| `subagent_start.py` | Logs full input payload for child agent creation |
| `notification.py` | TTS for permission prompts, multiple message variants, engineer name personalization |
| `user_prompt_submit.py` | Prompt logging, session data management, LLM-powered agent naming |
| `session_start.py` | Logs session metadata, optional git context loading |
| `session_end.py` | Logs session end reason, optional statistics |
| `pre_compact.py` | Transcript backup before context window compaction |
| `permission_request.py` | Permission decision logging |
| `post_tool_use_failure.py` | Tool failure logging |

### 8.3 AI Summarization

When `--summarize` is used, `send_event.py` calls `utils/summarizer.py` which:

1. Truncates the event payload to 1000 characters
2. Sends it to Anthropic's Haiku 4.5 model (`claude-haiku-4-5-20251001`)
3. Requests a one-sentence, under-15-word summary
4. Returns summaries like: "Reads configuration file from project root"

This uses your `ANTHROPIC_API_KEY` and sends payload excerpts to Anthropic's API. It can be disabled by removing `--summarize` from settings.json hooks.

### 8.4 Model Extraction

`utils/model_extractor.py` determines which model powers each agent:

- Reads the agent's `.jsonl` transcript file in reverse
- Finds the most recent assistant message containing a `"model"` field
- Returns the model name (e.g., `claude-opus-4-6`)
- File-based caching available but currently disabled (`ENABLE_CACHING = False`)

---

## 9. Human-in-the-Loop (HITL)

### 9.1 Architecture

HITL enables agents to ask humans questions and block until answered:

```
Agent (Python hook)                    Server (Bun)              Dashboard (Vue)
       │                                  │                          │
       ├── Start WebSocket server         │                          │
       │   on random localhost port       │                          │
       │                                  │                          │
       ├── POST event with                │                          │
       │   humanInTheLoop: {              │                          │
       │     question: "...",             │                          │
       │     responseWebSocketUrl:        │                          │
       │       "ws://localhost:RAND",     │                          │
       │     type: "permission",          │                          │
       │     timeout: 300                 │                          │
       │   }                              │                          │
       │                          ──────> │ Store event              │
       │                                  │ ──────────────────────> │ Show HITL card
       │                                  │                          │ (question + buttons)
       │   [agent blocks, waiting]        │                          │
       │                                  │                          │ Human responds
       │                                  │ <────────────────────── │ POST /events/:id/respond
       │                                  │                          │
       │   <────── WS connection ──────── │ Connect to agent's WS   │
       │          (server = client)       │ Send response JSON       │
       │                                  │                          │
       ├── Receive response               │                          │
       ├── Close WebSocket                │                          │
       └── Continue execution             │                          │
```

**Key insight:** The agent acts as WebSocket **server**, and the Bun server acts as WebSocket **client** for the response path. This reversal lets the agent block synchronously while waiting.

### 9.2 HITL Request Types

| Type | UI | Response |
|------|-----|----------|
| `permission` | Approve / Deny buttons | `{ "approved": true/false }` |
| `question` | Text input field | `{ "answer": "user's text" }` |
| `choice` | Radio buttons from `choices[]` | `{ "selectedChoice": "option" }` |

### 9.3 HITL Library

**File:** `.claude/hooks/utils/hitl.py`

```python
from utils.hitl import ask_permission, ask_question, ask_choice

# In any hook:
result = ask_permission("Deploy to production?", timeout=300)
if result and result.get("approved"):
    # proceed with deployment
```

**Note:** HITL is implemented as a library but not currently wired into any production hooks. See `test_hitl.py` and `examples/hitl_example.py` for usage.

---

## 10. Security Model

### 10.1 Hook-Level Safety

**Dangerous command blocking** (`pre_tool_use.py`):
- Splits commands on chaining operators (`;`, `|`, `` ` ``, `$(...)`, `\n`) before checking
- Blocks `rm` with recursive + force flags targeting dangerous paths (`/`, `~`, `$HOME`, `..`, `.`, `*`)
- Allows `rm` in explicitly configured directories (e.g., `trees/`)

**Path traversal prevention** (`constants.py`):
- Session IDs validated against `^[a-zA-Z0-9_-]+$`
- Resolved paths verified to be under the log base directory
- Crafted session IDs like `../../etc/cron.d/evil` fall back to `invalid-session`

**Transcript path validation** (`constants.py`):
- `validate_transcript_path()` only allows reads from `~/.claude` and `/tmp`
- Applied in `send_event.py`, `stop.py`, `subagent_stop.py`

### 10.2 Server-Level Safety

- **Binding:** `127.0.0.1` by default (not `0.0.0.0`)
- **CORS:** Restricted to configured origins (not wildcard `*`)
- **SSRF prevention:** HITL WebSocket URLs validated to only connect to localhost
- **SQL injection:** Sort order validated against allowlist (`ASC` / `DESC` only)
- **ID generation:** `crypto.randomUUID()` instead of `Math.random()`
- **Content-Disposition:** Filename sanitized to `[a-z0-9_-]` only
- **Dependencies:** `@types/bun` pinned (not `latest`)

### 10.3 Network Boundaries

All data stays local by default:

| Destination | What | When |
|---|---|---|
| `localhost:4000` | Hook event payloads | Every hook invocation |
| Anthropic API | Payload excerpts (1000 chars max) | Only with `--summarize` flag AND `ANTHROPIC_API_KEY` set |
| ElevenLabs API | TTS notification text | Only with `ELEVENLABS_API_KEY` set on Notification events |

No telemetry, no analytics, no unknown remote servers.

---

## 11. Data Model

### 11.1 Event Schema (SQLite)

```sql
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  source_app TEXT NOT NULL,          -- e.g., "cc-hook-multi-agent-obvs"
  session_id TEXT NOT NULL,          -- Claude Code session UUID
  hook_event_type TEXT NOT NULL,     -- One of 12 event types
  payload TEXT NOT NULL,             -- Full hook JSON (stringified)
  chat TEXT,                         -- Chat transcript JSON (Stop events only)
  summary TEXT,                      -- AI-generated one-sentence summary
  timestamp INTEGER NOT NULL,        -- Unix ms
  humanInTheLoop TEXT,               -- HITL request JSON
  humanInTheLoopStatus TEXT,         -- HITL response status JSON
  model_name TEXT                    -- e.g., "claude-opus-4-6"
);
```

### 11.2 Agent Identification

Per `CLAUDE.md`, agents are identified by `source_app:session_id` with session_id truncated to 8 characters for display:

```
Full:    cc-hook-multi-agent-obvs : 2176a933-8a56-4a1d-bf6b-9989a7d7e070
Display: cc-hook-multi-agent-obvs : 2176a933
```

### 11.3 Local Logging

Each hook writes structured JSON logs per session:

```
logs/
└── {session_id}/
    ├── pre_tool_use.json
    ├── post_tool_use.json
    ├── notification.json
    ├── subagent_stop.json
    ├── session_start.json
    └── ...
```

Session metadata is also stored in `.claude/data/sessions/{session_id}.json`.

---

## 12. Client Architecture

### 12.1 Component Hierarchy

```
App.vue
├── FilterPanel.vue              — dropdown filters (app, session, type)
├── LivePulseChart.vue           — canvas 30FPS real-time bar chart
├── EventTimeline.vue            — scrollable event list + agent tags
│   └── EventRow.vue             — event card (regular or HITL variant)
│       └── ChatTranscriptModal  — modal transcript viewer
│           └── ChatTranscript   — message renderer
├── AgentSwimLaneContainer.vue   — per-agent chart wrapper
│   └── AgentSwimLane.vue        — per-agent activity chart
├── ThemeManager.vue             — theme selection modal
├── ToastNotification.vue        — new-agent detection toasts
└── StickScrollButton.vue        — auto-scroll toggle
```

### 12.2 Composables

| Composable | Purpose |
|---|---|
| `useWebSocket` | WebSocket connection with auto-reconnect (3s). Handles `initial` + `event` messages. Max 300 events. |
| `useChartData` | Time-bucketed aggregation (1m/3m/5m/10m). Tracks unique agents, tool calls, timing metrics. |
| `useEventColors` | Deterministic color assignment via hash (seed 7151). Tailwind palette for sessions, HSL for apps. |
| `useEventEmojis` | Emoji maps for 12 event types and 20+ tool names. Combo emojis for tool-specific events. |
| `useEventSearch` | Regex-based event search across all event fields. |
| `useHITLNotifications` | Browser Notification API for HITL requests (`requireInteraction: true`). |
| `useThemes` | Theme CRUD via REST API. CSS variable application. |
| `useAgentChartData` | Per-agent chart data filtering. |

### 12.3 Real-Time Data Flow

```
WebSocket /stream
    │
    ├── "initial" message: bulk history (up to 300 events)
    └── "event" message: single new event
            │
            ▼
      useWebSocket → reactive events[]
            │
            ├──> App.vue: apply FilterPanel selections
            │       │
            │       ├──> EventTimeline: filtered + searched events
            │       │       └── EventRow × N (with HITL interactive UI)
            │       │
            │       └──> useChartData: time-bucketed aggregation
            │               ├──> LivePulseChart (global activity)
            │               └──> AgentSwimLane × N (per-agent)
            │
            └──> useHITLNotifications: browser notifications for HITL
```

---

## 13. Configuration Reference

### 13.1 settings.json

**Location:** `.claude/settings.json`

| Key | Purpose |
|-----|---------|
| `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Set to `"1"` to enable team tools (TeamCreate, SendMessage, etc.) |
| `teammateMode` | Set to `"in-process"` for in-process agent spawning |
| `statusLine` | Custom status line command |
| `hooks.*` | Hook configurations for all 12 event types |

### 13.2 Environment Variables

| Variable | Used By | Purpose |
|----------|---------|---------|
| `SERVER_PORT` | Server, start script | Server port (default: 4000) |
| `SERVER_HOST` | Server | Bind address (default: 127.0.0.1) |
| `CLIENT_PORT` | Client, start script | Client dev server port (default: 5173) |
| `ALLOWED_ORIGINS` | Server | CORS origins (default: localhost:5173,localhost:4000) |
| `ANTHROPIC_API_KEY` | summarizer, anth.py | Required for AI summaries and agent naming |
| `OPENAI_API_KEY` | oai.py | Alternative LLM for summaries |
| `ELEVENLABS_API_KEY` | elevenlabs_tts.py | TTS notifications |
| `ELEVENLABS_VOICE_ID` | elevenlabs_tts.py | ElevenLabs voice selection |
| `ENGINEER_NAME` | notification.py, anth.py | Personalizes TTS announcements |
| `CLAUDE_HOOKS_LOG_DIR` | constants.py | Log directory (default: logs) |

### 13.3 Just Recipes

```bash
just start          # Start server + client (foreground)
just stop           # Stop all processes
just restart        # Stop then start
just server         # Server only (dev mode)
just client         # Client only (dev mode)
just install        # Install all dependencies
just db-reset       # Delete events database
just db-clean-wal   # Clear SQLite WAL files
just test-event     # Send a test event
just health         # Check if server + client are running
just hook-test NAME # Test a hook script directly
just hooks          # List all hook scripts
just open           # Open dashboard in browser
```

---

## 14. Official Documentation References

### Claude Code Core

| Topic | URL |
|-------|-----|
| **Hooks System** — event types, configuration, stdin/stdout contract | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks) |
| **Sub-Agents** — Task tool, agent types, isolation modes | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents) |
| **Agent Teams** — TeamCreate, SendMessage, TaskCreate, coordination | [code.claude.com/docs/en/agent-teams](https://code.claude.com/docs/en/agent-teams) |
| **Settings** — settings.json, env vars, permissions, tools | [code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings) |
| **Permissions** — permission modes, allowlists, hook-based control | [code.claude.com/docs/en/permissions](https://code.claude.com/docs/en/permissions) |
| **MCP Integration** — Model Context Protocol server configuration | [code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp) |
| **Status Line** — custom status line configuration | [code.claude.com/docs/en/statusline](https://code.claude.com/docs/en/statusline) |

### Key Concepts from Official Docs

**Hooks** are shell commands that execute at defined lifecycle points. They receive JSON on stdin and can:
- **Observe** — log, forward, analyze events (exit 0)
- **Block** — output `{"decision": "block", "reason": "..."}` to prevent tool execution (PreToolUse only)
- **Modify** — output `{"tool_input": {...}}` to alter tool inputs (PreToolUse only)

**Sub-agents** (Task tool) run in isolated contexts. Key parameters:
- `subagent_type` — selects agent definition from `.claude/agents/`
- `isolation: "worktree"` — runs in a temporary git worktree
- `run_in_background: true` — non-blocking execution
- `model` — override model for the agent (haiku, sonnet, opus)

**Agent Teams** provide structured coordination:
- `TeamCreate` — creates team with shared task list
- `TaskCreate/TaskUpdate/TaskList/TaskGet` — shared work coordination
- `SendMessage` — direct messages, broadcasts, shutdown protocol
- Agents go idle between turns (normal behavior, not an error)
- Messages are automatically delivered to idle teammates

**Settings precedence:** project `.claude/settings.json` > user `~/.claude/settings.json` > defaults.

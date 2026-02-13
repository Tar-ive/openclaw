---
summary: "Specification: mapping Rewind's OS-inspired three-tier scheduling system onto OpenClaw primitives"
read_when:
  - Designing a task management system on top of OpenClaw
  - Integrating calendar/email/Slack context signals with agent scheduling
  - Building energy-aware or priority-based scheduling with heartbeat + cron
title: "Rewind Scheduling Integration Spec"
---

# Rewind Scheduling: Integration Spec for OpenClaw

This document maps the **Rewind** three-tier scheduling architecture (modeled on OS process scheduling) onto OpenClaw's existing primitives. It covers which OpenClaw files, tools, and subsystems to use, how to wire them, and where new code is needed.

## 1. Architecture Summary

Rewind uses three schedulers that mirror OS scheduling theory:

| Scheduler | OS Analogy | Rewind Role | OpenClaw Mapping |
|-----------|-----------|-------------|-----------------|
| **Long-Term Scheduler (LTS)** | Job scheduler (disk to ready queue) | Pulls tasks from the Task Buffer into daily/weekly view | **Cron job** (isolated, daily at planning time) + workspace `bank/` files |
| **Medium-Term Scheduler (MTS)** | Swapper (memory to disk) | Swaps tasks in/out of today's active schedule on context changes | **Heartbeat** (context-aware periodic checks) + **hooks** (event-driven) |
| **Short-Term Scheduler (STS)** | CPU scheduler (ready queue dispatch) | Optimizes ordering + handles preemption | **Agent loop** (continuous, within a heartbeat or cron turn) |

### Why OpenClaw is a good fit

- **Heartbeat** already runs periodic agent turns in the main session (default 30 min), making it a natural host for the MTS context-monitoring loop.
- **Cron** provides precise one-shot and recurring scheduling for the LTS daily planning pass and STS-triggered reminders.
- **Hooks** fire on lifecycle events (`agent:bootstrap`, `gateway:startup`, command events), enabling event-driven swap triggers.
- **Multi-agent routing** allows dedicated agents for different scheduler tiers or task domains.
- **Memory** (daily logs + vector search + `MEMORY.md`) provides durable task state that survives compaction.

## 2. OpenClaw Files and Where They Fit

### 2.1 Workspace layout

Extend the standard OpenClaw workspace (`~/.openclaw/workspace`) with Rewind-specific files:

```
~/.openclaw/workspace/
  AGENTS.md              # Add scheduler instructions (LTS/MTS/STS behavior)
  SOUL.md                # Persona: productivity-focused scheduler personality
  HEARTBEAT.md           # MTS checklist (context monitoring tasks)
  TOOLS.md               # Document any custom tools (calendar, email, etc.)
  MEMORY.md              # Core durable facts (user preferences, energy model)
  USER.md                # User profile + energy preferences

  # Rewind-specific additions
  rewind/
    task-buffer.json     # LTS: hash-table-style task store
    active-schedule.json # MTS/STS: today's active task list
    energy-model.json    # Energy state + history
    config.json          # Scheduling weights, thresholds, active hours

  memory/
    YYYY-MM-DD.md        # Daily logs (existing OpenClaw pattern)

  bank/
    schedule-history.md  # Curated scheduling decisions + outcomes
    opinions.md          # Learned scheduling preferences with confidence
    entities/
      # Entity pages for recurring tasks, projects, people
```

### 2.2 Bootstrap files (injected every turn)

These files are injected into the agent context on every run (see [System Prompt](/concepts/system-prompt)):

| File | Rewind Role | Size Guidance |
|------|------------|---------------|
| `AGENTS.md` | Scheduler behavior rules, LTS/MTS/STS logic | Keep under 2000 chars; reference `rewind/` files for detail |
| `HEARTBEAT.md` | MTS monitoring checklist | Keep under 500 chars; drives the swap engine |
| `MEMORY.md` | Durable user preferences, energy patterns | Keep under 1000 chars; use `memory_search` for deeper recall |
| `USER.md` | Energy preferences, work hours, task style | Keep under 500 chars |
| `SOUL.md` | Scheduler personality and boundaries | Keep under 500 chars |

**Important:** All bootstrap files consume context tokens every turn. Keep them concise and use `read` tool calls for larger data (see [Context](/concepts/context)).

## 3. Tier-by-Tier Integration

### 3.1 Long-Term Scheduler (LTS) — Cron + Workspace Files

**What it does:** Pulls tasks from the Task Buffer into the daily/weekly view based on deadline pressure, energy fit, and time availability.

**OpenClaw mechanism:** An isolated cron job that runs at planning time (e.g., 7 AM daily).

#### Cron job setup

```bash
openclaw cron add \
  --name "LTS: Daily planning" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Run LTS daily planning pass. Read rewind/task-buffer.json and rewind/config.json. Score tasks using the hash function (D*0.45 + E*0.30 + P*0.25). Select tasks for today based on available time and energy forecast. Write the result to rewind/active-schedule.json. Summarize what was scheduled." \
  --model "sonnet" \
  --announce \
  --channel whatsapp \
  --to "+15551234567"
```

#### Task Buffer data model (`rewind/task-buffer.json`)

```json
{
  "bucketCount": 16,
  "tasks": [
    {
      "id": "task-001",
      "title": "Review PR #42",
      "deadline": "2026-02-15T17:00:00Z",
      "estimatedMinutes": 30,
      "preferredStart": "2026-02-14T10:00:00Z",
      "energyCost": 3,
      "tags": ["work", "code-review"],
      "status": "buffered",
      "bucket": 4
    }
  ]
}
```

The hash function clusters similar tasks:

```
hash(task) = floor(D * 0.45 + E * 0.30 + P * 0.25) mod BUCKET_COUNT
```

Where:
- **D** (Deadline Urgency): `1 / max(1, hoursUntilDeadline)` — normalized 0..1
- **E** (Execution Time): `1 - (estimatedMinutes / maxEstimatedMinutes)` — shorter tasks score higher
- **P** (Preferred Start): `1 / max(1, hoursUntilPreferredStart)` — normalized 0..1

The agent reads this file, scores tasks, and writes selected tasks to `rewind/active-schedule.json`.

#### Active Schedule data model (`rewind/active-schedule.json`)

```json
{
  "date": "2026-02-14",
  "energyBudget": 20,
  "slots": [
    {
      "taskId": "task-001",
      "priority": 0,
      "scheduledStart": "2026-02-14T10:00:00Z",
      "scheduledEnd": "2026-02-14T10:30:00Z",
      "status": "pending",
      "savedState": null
    }
  ]
}
```

### 3.2 Medium-Term Scheduler (MTS) — Heartbeat + Hooks

**What it does:** Monitors real-time context signals and swaps tasks between the active schedule and the task buffer.

**OpenClaw mechanism:** The heartbeat checklist (`HEARTBEAT.md`) drives periodic context checks. Hooks enable event-driven reactions.

#### HEARTBEAT.md (MTS monitoring checklist)

```markdown
# Heartbeat checklist

## MTS context checks
- Read rewind/active-schedule.json for current task state
- Check calendar for schedule changes (cancelled meetings, early endings, new conflicts)
- Detect free time gaps; if gap > 15 min, consider SWAP-IN from task buffer
- Check energy signals (time of day, task completion velocity)
- If deadline pressure changed for buffered tasks, evaluate SWAP-IN
- If current task is blocked or overflowing, evaluate SWAP-OUT

## Swap rules
- SWAP-IN: query task buffer for tasks with estimatedMinutes <= available gap, filter by energy compatibility, rank by deadline urgency
- SWAP-OUT: if meeting ran over or energy is low, move displaced tasks back to buffer with saved state
- After any swap, update rewind/active-schedule.json and rewind/task-buffer.json
- Notify user only if a swap affects the next 2 hours

## STS micro-optimization
- Re-sort today's remaining tasks by priority queue (P0 > P1 > P2 > P3)
- If a P0 task arrived, preempt current task (save state first)
```

#### Heartbeat configuration

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "15m",  // MTS needs faster checks than default 30m
        target: "last",
        activeHours: { start: "07:00", end: "22:00" },
        // Optional: use a faster/cheaper model for routine checks
        // model: "anthropic/claude-haiku-3",
      },
    },
  },
}
```

See [Heartbeat](/gateway/heartbeat) and [Cron vs Heartbeat](/automation/cron-vs-heartbeat) for configuration details.

#### Event-driven hooks (swap triggers)

For real-time context changes that cannot wait for the next heartbeat, use hooks:

**Hook: `calendar-delta`** (workspace hook at `<workspace>/hooks/calendar-delta/`)

```
hooks/calendar-delta/
  HOOK.md       # metadata
  handler.ts    # implementation
```

`HOOK.md`:
```yaml
---
name: calendar-delta
description: Trigger MTS swap evaluation on calendar changes
events: [gateway:startup]
---
```

The hook can set up a polling interval or webhook listener for calendar changes, then enqueue a system event:

```bash
openclaw system event --mode now --text "Calendar change detected: meeting cancelled at 2pm. Run MTS swap-in evaluation."
```

See [Hooks](/automation/hooks) for the full hook API.

### 3.3 Short-Term Scheduler (STS) — Agent Loop Logic

**What it does:** Optimizes task ordering within the active schedule using a priority queue. Handles preemption.

**OpenClaw mechanism:** The STS runs as agent logic within each heartbeat or cron turn. It is not a separate daemon; it is instructions in `AGENTS.md` that the model follows.

#### Priority queue model

Encode priority levels in `AGENTS.md`:

```markdown
## STS Priority Queues

When re-sorting today's active schedule, assign tasks to priority queues:

- **P0 (Urgent)**: Hard deadline within 2 hours, external dependency waiting
- **P1 (Important)**: Deadline today, high-impact, upstream blockers
- **P2 (Normal)**: Routine tasks, flexible deadlines, personal goals
- **P3 (Background)**: Nice-to-haves, low-energy fillers, optional

### Preemption rules
- If a P0 task arrives, save current task state (progress notes, position) and switch
- When P0 completes, resume previous task from saved state
- Never preempt for P2/P3 tasks; only swap-in during idle gaps
```

#### Preemption via system events

When an urgent task arrives (e.g., from an email hook or Slack webhook), use a system event to trigger preemption:

```bash
openclaw system event --mode now --text "URGENT P0: Boss needs the quarterly report reviewed immediately. Preempt current task, save state, and switch."
```

See [Cron Jobs: system events](/automation/cron-jobs) for the full API.

### 3.4 Energy-Aware Scheduling

**What it does:** Models human energy as a depleting resource and adjusts task selection accordingly.

**OpenClaw mechanism:** Energy state is stored in `rewind/energy-model.json` and referenced by the MTS/STS logic in `HEARTBEAT.md` and `AGENTS.md`.

#### Energy model (`rewind/energy-model.json`)

```json
{
  "currentLevel": 4,
  "maxLevel": 5,
  "lastUpdated": "2026-02-14T14:30:00Z",
  "curve": {
    "morning": 5,
    "midday": 4,
    "afternoon": 3,
    "evening": 2,
    "night": 1
  },
  "signals": {
    "taskCompletionVelocity": 1.2,
    "lastBreak": "2026-02-14T12:00:00Z",
    "userReportedMood": null
  }
}
```

Add to `HEARTBEAT.md`:

```markdown
## Energy monitoring
- Read rewind/energy-model.json
- Infer energy level from time of day (curve) + task velocity + break recency
- Update currentLevel; write back to energy-model.json
- If energy < 2, only schedule energyCost <= 2 tasks (fillers, low-cognitive)
- If energy >= 4, prefer high-value energyCost 4-5 tasks
```

## 4. Multi-Agent Architecture (Optional)

For teams or complex setups, map Rewind's specialized agents to OpenClaw's multi-agent routing:

| Rewind Agent | OpenClaw Agent | Purpose |
|-------------|---------------|---------|
| Context Sentinel | `sentinel` agent | Fuses calendar/email/Slack signals into context events |
| Scheduler Kernel | `main` agent (default) | Runs LTS/MTS/STS logic via heartbeat + cron |
| GhostWorker | `ghost` agent (sandboxed) | Executes low-cognitive tasks via browser automation |
| Energy Monitor | Part of `main` agent | Tracked via heartbeat energy checks |

### Multi-agent configuration example

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Scheduler Kernel",
        workspace: "~/.openclaw/workspace",
        // Full tool access for scheduling logic
      },
      {
        id: "ghost",
        name: "GhostWorker",
        workspace: "~/.openclaw/workspace-ghost",
        sandbox: {
          mode: "all",
          scope: "agent",
        },
        tools: {
          allow: ["exec", "read", "write", "browser"],
          deny: ["cron", "sessions_send"],
        },
      },
    ],
  },
  bindings: [
    // Route Slack urgent messages to main for preemption
    { agentId: "main", match: { channel: "slack" } },
    // Route Telegram to main
    { agentId: "main", match: { channel: "telegram" } },
  ],
}
```

See [Multi-Agent Routing](/concepts/multi-agent) for full configuration.

### Delegating to GhostWorker

The main agent can delegate low-cognitive tasks to the GhostWorker using `sessions_spawn`:

```
When a P3 (Background) task is marked as "delegatable" in the task buffer,
use sessions_spawn to start a GhostWorker session with the task prompt.
The GhostWorker completes the task and writes results back to the workspace.
```

## 5. Memory Integration

Rewind benefits from OpenClaw's memory system for durable scheduling state:

### 5.1 Daily memory logs

Use `memory/YYYY-MM-DD.md` (existing OpenClaw pattern) to record scheduling decisions:

```markdown
## Retain
- S @rewind: LTS scheduled 6 tasks for today; 2 swapped out by MTS after 2pm meeting overran.
- B @rewind: Energy dropped to 2 by 3pm; STS shifted to P3 fillers for remaining afternoon.
- O(c=0.8) @user: User prefers deep-work tasks before 11am; schedule accordingly.
```

### 5.2 Vector memory search

Use `memory_search` to recall past scheduling patterns:

```
memory_search("scheduling preferences morning deep work", k=10, since="30d")
```

This returns relevant past decisions that inform future LTS planning.

See [Memory](/concepts/memory) and [Workspace Memory Research](/experiments/research/memory) for the full memory architecture.

### 5.3 Pre-compaction memory flush

Enable memory flush so scheduling insights are preserved before context compaction:

```json5
{
  agents: {
    defaults: {
      compaction: {
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
        },
      },
    },
  },
}
```

## 6. Implementation Roadmap

### Phase 1: Core scheduling (LTS + STS)

1. **Create workspace files**: `rewind/task-buffer.json`, `rewind/active-schedule.json`, `rewind/config.json`
2. **Write `AGENTS.md`**: Add LTS/STS scheduling logic and priority queue rules
3. **Add LTS cron job**: Daily planning pass at configured time
4. **Test**: Manually add tasks to buffer, run cron job, verify active schedule output

### Phase 2: MTS swap engine (heartbeat-driven)

1. **Write `HEARTBEAT.md`**: MTS monitoring checklist with swap rules
2. **Configure heartbeat**: Set interval to 15 min during active hours
3. **Add calendar integration**: Hook or heartbeat check for schedule changes
4. **Test**: Cancel a calendar event, verify MTS detects gap and swaps in a task

### Phase 3: Energy model

1. **Create `rewind/energy-model.json`**: Initialize with time-of-day curve
2. **Add energy checks to `HEARTBEAT.md`**: Infer and update energy level
3. **Wire energy to STS**: Only schedule energy-compatible tasks
4. **Test**: Verify afternoon low-energy correctly blocks high-cost tasks

### Phase 4: Multi-agent delegation (optional)

1. **Add GhostWorker agent**: Sandboxed agent for delegatable tasks
2. **Wire `sessions_spawn`**: Main agent delegates P3 tasks
3. **Test**: Delegate a background task, verify completion and result

### Phase 5: Context signal integration

1. **Add hooks**: Calendar delta, email urgency, Slack webhook triggers
2. **Wire system events**: Hooks emit events that trigger MTS/STS re-evaluation
3. **Test**: Send an urgent Slack message, verify preemption occurs

## 7. Key OpenClaw Docs to Reference

| Topic | Doc | URL |
|-------|-----|-----|
| Agent loop lifecycle | [Agent Loop](/concepts/agent-loop) | `docs/concepts/agent-loop.md` |
| What the model sees | [Context](/concepts/context) | `docs/concepts/context.md` |
| Heartbeat configuration | [Heartbeat](/gateway/heartbeat) | `docs/gateway/heartbeat.md` |
| Cron job scheduling | [Cron Jobs](/automation/cron-jobs) | `docs/automation/cron-jobs.md` |
| Choosing heartbeat vs cron | [Cron vs Heartbeat](/automation/cron-vs-heartbeat) | `docs/automation/cron-vs-heartbeat.md` |
| Event-driven hooks | [Hooks](/automation/hooks) | `docs/automation/hooks.md` |
| System prompt assembly | [System Prompt](/concepts/system-prompt) | `docs/concepts/system-prompt.md` |
| Multi-agent routing | [Multi-Agent](/concepts/multi-agent) | `docs/concepts/multi-agent.md` |
| Memory architecture | [Memory](/concepts/memory) | `docs/concepts/memory.md` |
| Memory research | [Memory Research](/experiments/research/memory) | `docs/experiments/research/memory.md` |
| Agent runtime | [Agent](/concepts/agent) | `docs/concepts/agent.md` |

## 8. Configuration Checklist

Minimal `~/.openclaw/openclaw.json` additions for Rewind scheduling:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "15m",
        target: "last",
        activeHours: { start: "07:00", end: "22:00" },
      },
      compaction: {
        memoryFlush: { enabled: true },
      },
    },
  },
  cron: {
    enabled: true,
  },
  hooks: {
    internal: {
      enabled: true,
    },
  },
}
```

Then add the LTS cron job:

```bash
openclaw cron add \
  --name "LTS: Daily planning" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Run LTS daily planning. Read rewind/task-buffer.json, score tasks, write rewind/active-schedule.json." \
  --announce
```

## 9. Limitations and Trade-offs

| Concern | Mitigation |
|---------|-----------|
| Heartbeat interval (15 min) is not truly continuous | Acceptable for human task management; sub-minute reactions use hooks + system events |
| Task buffer is a JSON file, not a real hash table | The agent interprets the hash function; file I/O is fast enough for personal task counts |
| Energy model is inferred, not measured | Time-of-day curves + task velocity are reasonable proxies; user can override via commands |
| Multi-agent adds complexity | Start single-agent (Phase 1-3); add agents only when delegation is needed |
| Context window limits | Keep `HEARTBEAT.md` and `AGENTS.md` small; use `read` tool for `rewind/*.json` files instead of injecting them |

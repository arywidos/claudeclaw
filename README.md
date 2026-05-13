# ClaudeClaw — Skill, Multi-Agent Orchestration Framework for Claude Code

Alternative multi-agent orchestration for Claude Code. Replaces OpenClaw's PM layer with a reliable, blocking delegation model.

## Why ClaudeClaw?

### vs. Regular Claude Code

Regular Claude Code loses all context when you close a session. Start a new conversation tomorrow and it's a blank slate — no memory of what you built, what's pending, or what decisions were made.

**ClaudeClaw persists everything to disk.** Close your session, come back next week, and Claude picks up exactly where you left off:

| | Regular Claude Code | ClaudeClaw |
|--|---------------------|------------|
| **Context across sessions** | Lost — every session starts from zero | Persisted in `project-tracker.md` |
| **Project state** | You have to re-explain everything | Auto-loaded — status, blockers, decisions, all on disk |
| **Delegation** | You describe the task yourself each time | Agent personas embedded, tasks delegated with full context |
| **Resume** | Not possible — start over | "Continue or start new?" — instant resume |
| **Status updates** | You must manually ask "update the status" | Proactive — auto-writes after every milestone, decision, or blocker |

### vs. OpenClaw

OpenClaw PM had 3 critical failures:
1. **Fire-and-forget** — 81.5% of delegated tasks never returned results (74/92 backend-dev spawns)
2. **PM idle** — PM stops working unless manually prodded, projects stall
3. **Timeout chaos** — Short timeouts (15-300s) killed subagents before they even started

ClaudeClaw fixes all of these by making the orchestrator (Claude Code) the active PM that **waits for results** before proceeding.

---

## Setup

### Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed (`npm install -g @anthropic-ai/claude-code`)
- Git

### Install

```bash
# Clone the framework
git clone https://github.com/your-org/ClaudeClaw.git

# Windows
cd C:\Claude\ClaudeClaw

# macOS / Linux
cd ~/Claude/ClaudeClaw
```

That's it. No `npm install`, no dependencies. ClaudeClaw is a prompt-based framework — all you need is Claude Code and the files in this repo.

### How It Auto-Loads

When you open Claude Code in this directory, `CLAUDE.md` is automatically read. It tells Claude to:

1. Ask you to pick a mode (**Project Mode** or **Normal Mode**)
2. If Project Mode — read `project-tracker.md`, load agent personas, and start orchestrating

You don't need to manually configure anything. Just `cd` into the directory and open Claude Code.

---

## Usage

### Starting a Session

```bash
# Open Claude Code in the ClaudeClaw directory
cd C:\Claude\ClaudeClaw    # Windows
cd ~/Claude/ClaudeClaw     # macOS / Linux

# Launch Claude Code
claude
```

Claude will greet you with a mode picker:

```
Welcome to ClaudeClaw! Choose your mode:
1. Project Mode — Orchestrate agents, track state, persist progress across sessions
2. Normal Mode — Regular Claude Code, no delegation or state tracking

Type 1 or 2, or say "project mode" / "normal mode".
```

### Switching Modes Mid-Session

| Command | Action |
|---------|--------|
| **"project mode"** or **"mulai project"** | Enter Project Mode — read tracker, load personas, start orchestrating |
| **"normal mode"** or **"stop project"** | Exit to Normal Mode — regular Claude, no delegation or tracking |

### Project Mode Flow

After choosing Project Mode, Claude will:

1. Read `project-tracker.md` for active projects
2. Ask: *"Active project found. Continue or start new?"*
3. You decide → continue an existing project or create a new one
4. Claude delegates tasks to specialized agents, **waiting for results** each time
5. After every action, `project-tracker.md` is updated automatically

### Working on a Project

In Project Mode, Claude acts as orchestrator. You give high-level instructions, Claude breaks them down and delegates:

```
You: "Build a REST API for user management"
Claude: (reads Backend Dev persona from agent-personas.md)
        (spawns Backend Dev agent with task)
        (waits for result)
        (verifies result quality)
        (updates project-tracker.md)
        (reports back to you)
```

You can also request specific agents:

```
You: "Have QA test the auth flow"
You: "Run a security audit on the payments module"
You: "Research competing SaaS image generators"
```

---

## Two Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Normal** | Default / "normal mode" | Regular Claude Code — Q&A, coding, no project tracking |
| **Project** | "project mode" / "mulai project" | Orchestrator mode — delegate to agents, track state, persist progress |

---

## Architecture

```
┌──────────────┐
│     You       │
└──────┬──────┘
       │
┌──────▼──────┐
│ Claude Code  │ ← PM + Orchestrator (blocking, not fire-and-forget)
│ (Project    │
│  Mode)       │ ← Reads project-tracker.md on session start
│              │ ← Updates tracker after every action
│              │ ← Spawns specialized agents via Agent tool
└──────┬──────┘
       │
  ┌────┼────┬────────┬──────────┬──────────┐
  │    │    │        │          │          │
  ▼    ▼    ▼        ▼          ▼          ▼
Backend Frontend DevOps  QA      Security  Researcher
```

Key difference from OpenClaw: **Claude Code waits for each agent result before proceeding.** No fire-and-forget.

---

## Agent Personas

### Core (always available)

| Persona | Specialty | When to use |
|---------|-----------|-------------|
| Backend Dev | API, DB, Python, scraper, data pipeline | Backend tasks, data engineering |
| Frontend Dev | UI, React/Vue/Streamlit, charts, accessibility | Frontend tasks, dashboard, landing page |
| DevOps | Docker, CI/CD, deploy, monitoring, infra | Deployment, infra setup, automation |

### Situational (when project needs it)

| Persona | Specialty | When to use |
|---------|-----------|-------------|
| QA Tester | Testing, bug reports, acceptance criteria | Before deploy, end-to-end verification |
| Security Eng | Threat model, audit, OWASP, penetration testing | Auth/payment/user data features |
| Researcher | Market intel, data gathering, analysis | Before starting project, tech comparison |

Full persona prompts in [`agent-personas.md`](agent-personas.md).

---

## Project State Persistence

File: `project-tracker.md` — the backbone of cross-session continuity.

On new session start (project mode):
1. Claude reads `project-tracker.md` → finds active projects
2. Asks: *"Active project found. Continue or start new?"*
3. You decide → continue or start fresh

### Proactive Auto-Update

ClaudeClaw doesn't wait for you to ask "update the status." In Project Mode, the tracker is **automatically updated** after every qualifying event:

| Trigger | What gets written |
|---------|-------------------|
| Agent task completes (success or failure) | Status, phase, last activity |
| Milestone reached (feature done, bug fixed) | Phase, last activity, next steps |
| Blocker discovered or resolved | Blockers field updated |
| Decision made (tech choice, scope change) | Decisions field updated |
| New project created | Full project entry added |
| User switches to normal mode | Mode set to `normal` |

This means you can close your session mid-task, come back tomorrow, and Claude will know exactly where things stand — no re-explaining, no lost context.

---

## Session Lifecycle

```
NEW SESSION STARTS
  │
  ├─ CLAUDE.md auto-loaded by Claude Code
  │
  ├─ Claude asks: "Project Mode or Normal Mode?"
  │
  ├─ Normal Mode?
  │   └─ Regular Claude Code behavior
  │
  └─ Project Mode?
      ├─ Read project-tracker.md
      ├─ Find ACTIVE projects
      ├─ Ask user what to work on
      ├─ Read persona from agent-personas.md
      ├─ Spawn Agent with persona + task
      ├─ WAIT for result (blocking, not fire-and-forget)
      ├─ Verify result
      ├─ Update project-tracker.md
      └─ Continue or report
```

---

## File Structure

```
ClaudeClaw/
  CLAUDE.md              ← Auto-read by Claude Code (framework config + startup prompt)
  README.md              ← This file (full documentation)
  project-tracker.md     ← Active project state
  agent-personas.md      ← 6 persona prompts (core + situational)
  two-mode-system.md     ← Mode switching rules
  lessons-learned.md    ← What went wrong with OpenClaw and how we fix it
  projects/              ← Per-project working directories
```

---

## Delegation Pattern

When delegating in project mode:
1. Read persona prompt from `agent-personas.md`
2. Spawn Agent with persona + task
3. **WAIT for result** (blocking, never fire-and-forget)
4. Verify result quality
5. Update `project-tracker.md`
6. Proceed to next task

---

## Comparison with OpenClaw

| Aspect | OpenClaw PM (Bunga) | ClaudeClaw |
|--------|---------------------|------------|
| Orchestrator | Agent (idle, needs cron) | Claude Code (active, blocking) |
| Delegation | sessions_spawn (fire-and-forget) | Agent tool (blocking, waits for result) |
| Success rate | 18.5% (17/92) | Near 100% (blocking) |
| State persistence | progress.md (unreliable) | project-tracker.md (reliable, checked every session) |
| Idle problem | PM idle, project stalls | No PM — you + Claude Code, always active when session is open |
| Persona | SOUL.md per agent (117-276 lines) | Compressed prompts (~40 lines, embedded) |
| Cost | 12 persistent agents running | 1 Claude Code session + on-demand agents |
| Context across sessions | Unreliable (auto-announce fails 81.5%) | File-based (project-tracker.md, always on disk) |

---

## Lessons from OpenClaw

Full analysis in [`lessons-learned.md`](lessons-learned.md). Key findings:

- 81.5% of delegated tasks never returned results
- PM set timeouts as low as 15s (context loading alone takes 30-60s)
- PM spawned replacement tasks before checking if original was still running
- Session compaction lost in-flight auto-announce messages
- Each new spawn had no context from previous attempts (no resume mechanism)

ClaudeClaw fixes all of these by design.

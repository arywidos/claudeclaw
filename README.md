# ClaudeClaw — Skill, Multi-Agent Orchestration Framework for Claude Code

Alternative multi-agent orchestration for Claude Code. Replaces OpenClaw's PM layer with a reliable, blocking delegation model.

## Why ClaudeClaw?

OpenClaw PM had 3 critical failures:
1. **Fire-and-forget** — 81.5% of delegated tasks never returned results (74/92 backend-dev spawns)
2. **PM idle** — PM stops working unless manually prodded, projects stall
3. **Timeout chaos** — Short timeouts (15-300s) killed subagents before they even started

ClaudeClaw fixes this by making the orchestrator (Claude Code) the active PM that **waits for results** before proceeding.

---

## Quick Start

### Enter Project Mode
```
cd D:\AI\ClaudeClaw
```
Then say: **"project mode"**

Or from any directory: **"baca D:\AI\ClaudeClaw\CLAUDE.md lalu project mode"**

### Exit to Normal Mode
Say: **"normal mode"** or **"stop project"**

### New Session Checklist
1. `cd D:\AI\ClaudeClaw` — Claude Code auto-reads `CLAUDE.md`
2. Say "project mode"
3. Claude reads `project-tracker.md` for active projects
4. Claude asks: "Ada X project aktif. Lanjutkan?"
5. You decide → continue or start fresh

---

## Two Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Normal** | Default | Regular Claude Code — Q&A, coding, no project tracking |
| **Project** | "project mode", "mulai project" | Orchestrator mode — delegate to agents, track state, persist progress |

---

## How It Works

```
┌──────────────┐
│  You (Mas Ary) │
└──────┬──────┘
       │
┌──────▼──────┐
│  Claude Code │ ← PM + Orchestrator (blocking, not fire-and-forget)
│  (Project    │
│   Mode)      │ ← Reads project-tracker.md on session start
│              │ ← Updates tracker after every action
│              │ ← Spawns specialized agents via Agent tool
└──────┬──────┘
       │
  ┌────┼────┬────────┬──────────┬──────────┐
  │    │    │        │          │          │
  ▼    ▼    ▼        ▼          ▼          ▼
🏗️   🖥️   ⚙️    🧐       🔒       🔭
Back  Front  Dev    QA      Security  Research
end    end    Ops    Test    Eng       er
```

Key difference from OpenClaw: **Claude Code waits for each agent result before proceeding.** No fire-and-forget.

---

## Agent Personas

### Core (always available)

| Persona | Specialty | When to use |
|---------|-----------|-------------|
| 🏗️ Backend Dev | API, DB, Python, scraper, data pipeline | Backend tasks, data engineering |
| 🖥️ Frontend Dev | UI, React/Vue/Streamlit, charts, accessibility | Frontend tasks, dashboard, landing page |
| ⚙️ DevOps | Docker, CI/CD, deploy, monitoring, infra | Deployment, infra setup, automation |

### Situational (when project needs it)

| Persona | Specialty | When to use |
|---------|-----------|-------------|
| 🧐 QA Tester | Testing, bug reports, acceptance criteria | Before deploy, end-to-end verification |
| 🔒 Security Eng | Threat model, audit, OWASP, penetration testing | Auth/payment/user data features |
| 🔭 Researcher | Market intel, data gathering, analysis | Before starting project, tech comparison |

Full persona prompts in [`agent-personas.md`](agent-personas.md).

---

## Project State Persistence

File: `project-tracker.md` — updated after every significant action.

On new session start (project mode):
1. Read `MEMORY.md` → see reference to project-tracker
2. Read `project-tracker.md` → find active projects
3. Ask user: "Ada X project aktif. Lanjutkan?"
4. User decides → continue or start fresh

This solves the "PM idle" problem — state is always on disk, ready for any session.

---

## Session Lifecycle

```
NEW SESSION STARTS
  │
  ├─ Read MEMORY.md (auto-loaded)
  │
  ├─ Mode = normal?
  │   └─ Regular Claude Code behavior
  │
  └─ Mode = project?
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
D:\AI\ClaudeClaw\
  CLAUDE.md                  ← Auto-read by Claude Code (framework overview)
  README.md                   ← This file (full documentation)
  project-tracker.md          ← Active project state
  agent-personas.md            ← 6 persona prompts (core + situational)
  two-mode-system.md           ← Mode switching rules
  lessons-learned.md           ← What went wrong with OpenClaw and how we fix it
```

Auto-loaded by Claude Code (memory system):
```
C:\Users\arywidos\.claude\projects\C--Users-arywidos\memory\
  MEMORY.md                    ← Index (references ClaudeClaw files)
  project-tracker.md           ← Project state (mirror of workspace)
  agent-personas-core.md       ← Agent persona prompts (mirror)
  feedback_two-mode-system.md  ← Mode switching rules
  openclaw-spawn-yield-fix.md  ← Lesson learned about spawn+yield
```

Home directory auto-read:
```
C:\Users\arywidos\CLAUDE.md   ← Pointer to ClaudeClaw framework
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

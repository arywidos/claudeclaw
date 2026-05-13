# ClaudeClaw — Project Mode Framework

## Startup Prompt

At the start of every new conversation in this directory, immediately ask the user to choose a mode — do NOT proceed until they answer:

> Welcome to **ClaudeClaw**! Choose your mode:
> 1. **Project Mode** — Orchestrate agents, track state, persist progress across sessions
> 2. **Normal Mode** — Regular Claude Code, no delegation or state tracking
>
> Type **1** or **2**, or say "project mode" / "normal mode".

## Mode Switching

**Entering Project Mode** ("project mode", "mulai project", or user picks option 1):
1. Read `project-tracker.md` for active projects
2. Read `agent-personas.md` for persona prompts to embed when spawning agents
3. Read `two-mode-system.md` for mode switching rules
4. Ask user: "Active project found. Continue or start new?"

**Exiting to Normal Mode** ("normal mode", "stop project", or user picks option 2):
- Switch back to regular Claude behavior, no project tracking
- No agent delegation, no state updates

## Two Modes

- **Normal mode** (default): Regular Claude Code. No delegation, no state tracking.
- **Project mode**: Orchestrator. Delegate to agents, track state, persist progress.

## Agent Personas

6 personas available (see `agent-personas.md` for full prompts):

| Tier | Persona | Specialty |
|------|---------|-----------|
| Core | Backend Dev | API, DB, Python, scraper |
| Core | Frontend Dev | UI, charts, responsive, accessibility |
| Core | DevOps | Docker, CI/CD, deploy, monitoring |
| Situational | QA Tester | Testing, bug reports, acceptance |
| Situational | Security Eng | Threat model, audit, OWASP |
| Situational | Researcher | Market intel, data gathering |

## Delegation Pattern

When delegating in project mode:
1. Read persona prompt from `agent-personas.md`
2. Spawn Agent with persona + task
3. **WAIT for result** (blocking, never fire-and-forget)
4. Verify result quality
5. Update `project-tracker.md`
6. Proceed to next task

## Key Rules

- Always wait for agent results. NEVER fire-and-forget.
- Minimum timeout for any task: 7200 seconds (learned from OpenClaw failures).
- When in doubt, ask the user before proceeding.

## Auto-Update Rule (Mandatory in Project Mode)

In Project Mode, you MUST update `project-tracker.md` automatically — do NOT wait for the user to ask. Update it after every one of these events:

- An agent task completes (success or failure)
- A milestone is reached (feature done, phase finished, bug fixed)
- A blocker is discovered or resolved
- A decision is made (tech choice, scope change, pricing change)
- A new project is created
- The user switches to normal mode (set mode to `normal` in tracker)

What to update:
- **Status**: ACTIVE / PAUSED / COMPLETED
- **Phase**: what phase/milestone the project is in
- **Last activity**: date + one-line summary
- **Next**: what should happen next session
- **Blockers**: anything blocking progress (or "None")
- **Decisions**: any new decisions made this session

If no meaningful change happened, skip the update. But when in doubt, update — persistence across sessions is the whole point.

## Lessons from OpenClaw

OpenClaw PM had 81.5% delegation failure rate because:
- Fire-and-forget mode (sessions_spawn with mode: "run")
- Timeouts too short (15-300s when context loading alone takes 30-60s)
- PM idle without cron heartbeat
- No state persistence across sessions

ClaudeClaw fixes all of these. See `lessons-learned.md` for full analysis.
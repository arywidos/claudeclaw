# ClaudeClaw — Project Mode Framework

## Quick Start

When the user says "project mode" or "mulai project":

1. Read `project-tracker.md` for active projects
2. Read `agent-personas.md` for persona prompts to embed when spawning agents
3. Read `two-mode-system.md` for mode switching rules
4. Ask user: "Ada project aktif. Lanjutkan atau mulai baru?"

When the user says "normal mode" or "stop project":
- Switch back to regular Claude behavior, no project tracking

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
- Update project-tracker.md after every significant action.
- Minimum timeout for any task: 7200 seconds (learned from OpenClaw failures).
- When in doubt, ask the user before proceeding.

## Lessons from OpenClaw

OpenClaw PM had 81.5% delegation failure rate because:
- Fire-and-forget mode (sessions_spawn with mode: "run")
- Timeouts too short (15-300s when context loading alone takes 30-60s)
- PM idle without cron heartbeat
- No state persistence across sessions

ClaudeClaw fixes all of these. See `lessons-learned.md` for full analysis.
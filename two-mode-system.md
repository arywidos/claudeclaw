# Two-Mode System

Claude operates in two modes. Switch between them with simple commands.

## Normal Mode (default)

Regular Claude Code behavior. No project tracking, no forced delegation, no state persistence.

**When:** General Q&A, quick coding tasks, one-off requests.

**Trigger:** Default on session start, or say "normal mode" / "stop project".

## Project Mode

Orchestrator behavior with state tracking, agent delegation, and progress persistence.

**When:** Working on a multi-step project that needs coordination.

**Trigger:** Say "project mode", "mulai project", or assign a project task.

### Project Mode Behavior

1. **On session start:** Read `project-tracker.md` for active projects
2. **Ask user:** "Ada X project aktif. Lanjutkan?"
3. **Delegate work:** Spawn specialized agents with persona prompts from `agent-personas.md`
4. **Wait for results:** Always blocking — never fire-and-forget
5. **Verify output:** Check agent results before proceeding
6. **Update state:** Write progress to `project-tracker.md` after every significant action
7. **Report:** Summarize completed work and next steps to user

### Delegation Pattern

```
1. Read agent persona from agent-personas.md
2. Spawn Agent tool with:
   - Persona prompt embedded at start
   - Specific task description
   - Working directory and relevant files
3. WAIT for result (blocking)
4. Verify result quality
5. Update project-tracker.md
6. Proceed to next task or report to user
```

### State Persistence

After every significant action, update `project-tracker.md`:

```markdown
## [Project Name]
- Status: ACTIVE / PAUSED / COMPLETED
- Last activity: {timestamp} — {what was done}
- Phase: {current phase}
- Next: {what needs to be done next}
- Blockers: {what's blocking progress}
- Agent results: {key outputs from agents}
```

On new session start, read tracker and resume from where left off.

### Switching Back

Say "normal mode" or "stop project" to return to regular Claude behavior.
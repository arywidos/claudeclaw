# Lessons Learned — Why OpenClaw PM Failed

Analysis based on actual log data from 92 backend-dev spawns by OpenClaw PM (Bunga).

## The Problem

### Statistic: 81.5% of delegated tasks never returned results

- Total backend-dev spawns: 92
- Completion announcements received by PM: **17 (18.5%)**
- Fire-and-forget (no result): **75 (81.5%)**

### Root Causes

#### 1. Fire-and-Forget by Design (`spawnMode: "run"`)

Every spawn used `mode: "run"` which means PM sends task and immediately continues.
The system says "auto-announces on completion" but auto-announce failed 81.5% of the time.

```json
{
  "status": "accepted",
  "mode": "run",
  "note": "auto-announces on completion, do not poll/sleep"
}
```

This note is misleading — auto-announce is unreliable.

#### 2. Timeouts Too Short

PM set wildly inconsistent timeouts:

| Timeout | Count | Result |
|---------|-------|--------|
| 15s | ~10 | Always fails — context loading alone takes 30-60s |
| 30-60s | ~8 | Almost always fails |
| 120-300s | ~15 | Often fails for complex tasks |
| 600-900s | ~15 | Sometimes works |
| 7200s (2hr) | 3 | Works — but only adopted late after user frustration |

Lesson: **Minimum timeout should be 7200s for all tasks.** Context loading alone takes 30-60s.

#### 3. PM Doesn't Wait for Results

PM spawned backend-dev, said "Auto-announce sebentar lagi", then immediately spawned another task or moved on. When auto-announce didn't arrive, PM assumed failure and re-spawned with adjusted parameters.

Average time between consecutive backend-dev spawns: **47-200 seconds** — many tasks weren't even done loading context before being replaced.

#### 4. Session Compaction Lost In-Flight Announcements

PM session ran for 7+ days (2.2MB, 1301 lines). When context got too large, OpenClaw performed compaction. During compaction, in-flight auto-announce messages could be lost.

#### 5. No Resume Mechanism

Each spawn created a fresh session. New subagent had no context from previous attempts — same mistakes repeated.

## How ClaudeClaw Fixes This

| Problem | OpenClaw PM | ClaudeClaw |
|---------|-------------|------------|
| Fire-and-forget | `mode: "run"`, 81.5% failure | Agent tool (blocking), near 100% success |
| Timeout too short | 15-300s | No artificial timeout — waits for result |
| PM doesn't wait | Says "auto-announce" then moves on | Blocking — waits for agent result before proceeding |
| Session compaction | Lost in-flight messages | State persisted to disk, always recoverable |
| No resume | Fresh session each time | project-tracker.md persists state across sessions |
| PM idle | Needs cron to wake up | Active when session is open, state on disk when closed |

## Key Metrics (OpenClaw PM Session Analysis)

- PM session duration: April 30 → May 7 (7+ days)
- Total subagent spawns: 182
- Backend-dev spawns alone: 92
- Completion rate: 18.5%
- Most common timeout: 120-300s (too short for most tasks)
- Only reliable timeout: 7200s (adopted after user explicitly complained)
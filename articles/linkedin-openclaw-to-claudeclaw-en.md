# From OpenClaw to ClaudeClaw: When a Simpler Solution Answers All the Problems

---

## The Beginning: OpenClaw and the Multi-Agent Ambition

Over the past month, I built an agentic squad on OpenClaw — a local multi-agent AI orchestration platform. The architecture was ambitious: 12 persistent agents, coordination via Discord, and a Project Manager (PM) agent named "Bunga" whose job was to delegate all work to sub-agents.

CEO(Human) → PM → backend-dev, frontend-dev, qa-tester, devops, and so on. Clean hierarchy, clear task division.

On paper, perfect.

In reality? **81.5% of delegated tasks failed to return to the PM.** Out of 92 tasks sent to backend-dev, only 17 had their results successfully received.

That number isn't a bug — it's a direct consequence of an architecture that relies on asynchronous communication without guaranteed delivery.

---

## What Went Wrong with OpenClaw?

### 1. Unreliable Delegation — Both Modes Failed

OpenClaw provides two delegation modes, and both fail:

**"Run" mode — Fire-and-forget.** The PM spawns a sub-agent for a one-shot task, then immediately moves on without waiting for results. The "auto-announce" mechanism is supposed to return results to the PM, but it fails 81.5% of the time. The PM never knows: did the task succeed? Fail? Or disappear along the way?

**"Session" mode — Persistent but without follow-up.** The sub-agent stays alive after the initial task, bound to a Discord thread. The PM or user can send follow-up instructions through the thread. The problem: the PM never reliably sends follow-ups. Sub-agents wait for next instructions that never arrive. Session mode is even more wasteful — the agent continues consuming context and memory until the 120-minute idle timeout, with no guarantee that follow-up will come.

Out of 28 recorded sub-agent runs in production, **27 used "session" mode and only 1 used "run" mode.** This means the core problem isn't the mode — it's structural. Both modes rely on the same unreliable delivery mechanism.

OpenClaw also has `sessions_yield` — a separate tool that makes delegation blocking. But the PM agent "Bunga" has `sessions_yield` on its **deny list**. Meaning the one tool that could save delegation is explicitly forbidden for the PM.

The analogy: send a courier with two options — express mail that might arrive, or might not. Or a courier that waits at the location but you forget to give follow-up instructions. Both fail, and the only solution (a tracking number) is prohibited.

### 2. A Delegate-Only PM — An Intentional Rule That Became a Trap

"Bunga" has an explicit rule: **NO EXECUTION, ALWAYS DELEGATE**. Tools like read, write, exec, bash — all forbidden.

This rule is intentional. Without this constraint, Bunga tends to execute tasks herself rather than delegating — even though the goal is for tasks to be handled by specialist agents with matching personas and expertise, not by a PM who only has coordination capabilities. Furthermore, Bunga also tends to hallucinate — producing inaccurate results when executing outside her domain.

So the "NO EXECUTION" rule isn't a design flaw. It's a reasonable constraint: PMs should coordinate, not implement.

The problem: this rule depends on the assumption that delegation will succeed. And in OpenClaw, delegation fails 81.5% of the time.

When a sub-agent fails and the PM can't execute, the only strategy is to spawn another sub-agent. And another. And another. A cycle that never ends because the root cause — the unreliable delivery mechanism — is never resolved.

The correct rule should be: **delegate first, but if it fails, execute as a fallback.** Not delegate-only without a safety net.

### 3. Premature Re-spawning

Because the PM doesn't receive results, it assumes the task failed and immediately respawns. From the logs, the average time between respawns is only **47-200 seconds** — while context loading alone takes 30-60 seconds. Many sub-agents haven't finished loading before being replaced by a new spawn.

Send a courier, 3 minutes no response, send another. The first one is actually still on the way — but you've already sent a replacement.

### 4. Session Compaction Swallowing Data

The PM session ran for 7+ days, accumulating 2.2MB of context. When it gets too large, OpenClaw performs compaction — and in-flight messages (sub-agent results being delivered) can be lost in this process.

The courier arrives at the office, but the receptionist is doing a shift change. The letter disappears.

### 5. 12 Persistent Agents, 80% Idle

OpenClaw runs 12 agents simultaneously, each with its own session, memory database (13-27MB per agent), and workspace. In a typical workflow, only 2-3 agents are active. The rest are idle but still consuming resources.

80% of allocated resources are unproductive.

### 6. Cron Watchdog — A Solution That Isn't a Solution

To handle the idle PM, a cron job was created every 15 minutes to "wake up" the PM. This is a workaround, not a solution. A task that finishes at minute 2 has to wait until minute 15 to be processed. And the cron itself often errors: 5 consecutive failures, then it gets disabled.

A system that needs a cron watchdog to function is a system with a fundamental design flaw.

---

## Migration to ClaudeClaw: Rethink, Not Rebuild

ClaudeClaw is essentially Claude with its framework augmented with skills to provide agentic features similar to OpenClaw.

I'm very satisfied with how Claude works with its async loop, but some features I needed for working on a project weren't available in Claude — so we added features from OpenClaw to Claude. Including memory and more persistent sessions — when project mode = active, Claude always remembers the progress of the project being worked on, even after the previous session has closed.

And we've since added three more features that make it even better: **automatic mode selection on session start** (no more forgetting to switch modes), **proactive status updates** (the tracker updates itself after every milestone — you never have to ask "please update the status"), and a **decision log** that records *why* choices were made, not just *what* was done.

https://github.com/arywidos/claudeclaw

From all those failures, I didn't fix OpenClaw. I **completely rethought** its orchestration principles.

ClaudeClaw is not a rebuild with incremental improvements. It's a new architecture built on top of Claude Code, directly addressing every OpenClaw flaw.

### Flaw → Fix

| OpenClaw Problem | ClaudeClaw Solution |
|---|---|
| Fire-and-forget (81.5% fail) | **Blocking delegation** — orchestrator waits for results before proceeding. Nearly 100% success rate |
| PM delegate-only (intentional, but no fallback) | **Delegate first, fallback to execution** — if the specialist fails, the orchestrator steps in |
| Premature re-spawn (47-200s) | **No need to re-spawn** — blocking waits for results, no resending needed |
| Session compaction eating data | **State on disk (file-based)** — not lost during crash, compaction, or restart |
| 12 persistent agents, 80% idle | **1 orchestrator + on-demand agents** — spawn only when needed, destroy after completion |
| Cron watchdog every 15 minutes | **Real-time monitoring** — orchestrator is active, no polling needed |
| No mode switching — always project mode | **Auto mode prompt** — asks Project or Normal on every session start |
| PM never updates status unless prodded | **Proactive auto-update** — tracker writes itself after every milestone |
| Decisions scattered in threads and memory | **Decision log** — structured record of why choices were made |

### Metrics Comparison

| Metric | OpenClaw | ClaudeClaw |
|--------|----------|-------------|
| Delegation success rate | 18.5% | ~100% |
| Re-spawn interval | 47-200 seconds | N/A (not needed) |
| Persona prompt per agent | 117-276 lines SOUL.md | ~40 lines compressed |
| Active agents | 12 persistent | 1 + on-demand |
| Memory DB footprint | ~200 MB | 0 (file-based) |
| State recovery after crash | Unreliable | Reliable |
| PM execution capability | None | Full |
| Status updates | Manual (often forgotten) | Proactive (auto-written after events) |
| Decision history | None (scattered in threads) | Structured log with reasoning |

---

## Bringing Agentic Concepts from OpenClaw to Claude

What's interesting about this process: **the agentic concepts from OpenClaw aren't wrong.** What's wrong is the implementation.

CEO → PM → specialist agents hierarchy? Good idea. Task delegation with persona-specific prompts? Good idea. Project state tracking? Good idea.

What failed is the delivery mechanism — not the concept.

ClaudeClaw brings these concepts to Claude Code, but with fundamentally different implementation:

### 1. Agentic Delegation — But Blocking

OpenClaw concept: PM delegates tasks to specialist sub-agents.
OpenClaw implementation: Asynchronous, fire-and-forget, 81.5% failure.
**ClaudeClaw implementation:** Agent tool that's blocking — orchestrator spawns sub-agent with persona prompt, waits for results, then continues. Every delegation succeeds because results are returned directly.

### 2. Persona System — But Lightweight

OpenClaw concept: Each agent has a 117-276 line SOUL.md defining personality, skills, and constraints.
OpenClaw implementation: 12 agents × ~300 lines = 3,600+ lines of context to load.
**ClaudeClaw implementation:** Compressed persona prompt ~40 lines, embedded directly in the task description when spawning. Persona is only loaded when needed. Agents that aren't spawned consume zero context.

### 3. Project Tracking — But Persistent on Disk

OpenClaw concept: PM holds project state in session memory (SQLite + in-memory).
OpenClaw implementation: State is lost when sessions crash, compact, or timeout.
**ClaudeClaw implementation:** Project state is saved in `project-tracker.md` — a markdown file that any session can read. Crash-safe, session-agnostic, human-readable, and version-controllable.

### 4. Specialization — But On-Demand

OpenClaw concept: 12 specialist agents running simultaneously, each with their own context and memory.
OpenClaw implementation: 80% idle, 200MB+ of unproductive memory.
**ClaudeClaw implementation:** 1 active orchestrator. Sub-agents are spawned on-demand with the needed persona, destroyed after completion. Resources are used only when needed.

---

## Adding Memory: So Claude Stays Connected Across Sessions

One thing OpenClaw tried to solve with 12 persistent agents: **continuity**. Each agent had its own memory database so they "remembered" context between sessions.

The problem: continuity that depends on always-alive sessions is fragile. Sessions crash, compact, or timeout — and their memory can become inconsistent.

In ClaudeClaw, I took a different approach to continuity:

### Memory in Claude Code

Claude Code has a built-in memory system that enables persistence across sessions:

- **CLAUDE.md** — Project-level instructions that are automatically loaded every new session. This isn't just a config file — it's the project's "DNA" that gives Claude context about preferences, architecture, and constraints to follow.

- **Memory files** — File-based memory that can be written and read across sessions. Every new session can access state from the previous session without depending on a persistently alive session.

- **Project tracker** — Project state saved as a file, not in session memory. Anyone — a new session, developer, or orchestrator — can read the latest state at any time.

**The result:** Continuity without persistent agents. A new session can immediately pick up where the previous one left off. No 200MB memory database. No sessions that need to be kept alive. No compaction that can swallow data.

This is what OpenClaw tried to achieve with 12 always-active agents, but ClaudeClaw achieves with file-based state that doesn't depend on session lifecycle.

---

## What We Added Next: Three Features That Make Persistence Even Stronger

After using ClaudeClaw in production, we noticed three friction points that OpenClaw never solved — and in some cases, made worse.

### 1. Automatic Mode Selection — No More "I Forgot to Switch Modes"

OpenClaw's PM always operated in project mode. There was no way to have a casual conversation without the overhead of agent orchestration.

In ClaudeClaw, we added an **automatic mode prompt** at the start of every new session. When you open Claude Code in the ClaudeClaw directory, the first thing you see is:

```
Welcome to ClaudeClaw! Choose your mode:
1. Project Mode — Orchestrate agents, track state, persist progress across sessions
2. Normal Mode — Regular Claude Code, no delegation or state tracking
```

No need to remember to say "project mode" every time. The framework proactively asks you. And you can switch mid-session anytime with a simple command.

### 2. Proactive Status Updates — The Tracker Writes Itself

OpenClaw's progress tracking depended on the PM writing updates — but the PM was idle most of the time. Project status was outdated, incomplete, or simply missing.

In ClaudeClaw, the tracker **updates itself automatically** after every qualifying event:

| When | What gets written |
|------|-------------------|
| An agent task completes (or fails) | Status, phase, last activity |
| A milestone is reached | Phase, next steps |
| A blocker is found (or resolved) | Blockers field |
| A decision is made | Decision logged with reasoning |
| User switches to normal mode | Mode set to `normal` |

You never have to ask "please update the status." It happens as part of the workflow.

### 3. Decision Log — Recording Why, Not Just What

This is the feature I wish we had from day one.

Every project makes decisions: tech stack, pricing model, architecture choices, scope changes. When you resume a project after two weeks, you remember *what* you decided, but often forget *why*. That leads to re-litigating choices that were already made, or worse — reversing a decision without understanding the original reasoning.

OpenClaw had no mechanism for this. Decisions were scattered across Discord threads and session memory, impossible to reconstruct.

ClaudeClaw now has a `decision-log.md` that automatically records every significant decision with:

- **What was decided** and what alternatives were considered
- **Why** the choice was made (reasoning, constraints, trade-offs)
- **Impact** — what this decision means for the project

Example:

```
### D-003: Midtrans for payments (not Stripe)
- Context: Payment processor for Indonesian market
- Options considered:
  1. Stripe only
  2. Midtrans only
  3. Midtrans + Stripe (Midtrans for ID, Stripe for intl)
- Chosen: Option 3
- Why: Midtrans supports QRIS, GoPay, VA — essential for ID market.
  Stripe doesn't support these. Midtrans Starter Pack only needs KTP, no PT.
- Impact: No recurring revenue yet, but higher initial conversion.
```

Resume a project months later, and you don't just know *what* happened — you know *why*. No more "why did we choose X again?"

---

## Claude Patching OpenClaw's Flaws — Right Now

What's most interesting about this process: **Claude Code already has the capability to patch every fundamental OpenClaw flaw.**

No custom runtime needed. No Discord integration needed. No cron watchdog needed. No 12 persistent agents needed.

| OpenClaw Flaw | Claude Code Capability That Patches It |
|---|---|
| Asynchronous delegation without reliable delivery | **Agent tool** — blocking, synchronous, results return directly to orchestrator |
| PM delegate-only (intentional, but no fallback) | **Claude Code can execute as fallback** — delegate first, if it fails, step in |
| Session compaction losing state | **File-based memory** — state on disk, not in volatile session memory |
| 12 persistent agents, 80% idle | **On-demand agent spawning** — spawn when needed, destroy after completion |
| 300+ line persona prompts per agent | **Compressed persona in task description** — ~40 lines, loaded only when needed |
| Cron watchdog to wake up PM | **Not needed** — orchestrator is naturally active when receiving a task |
| State lost when session ends | **CLAUDE.md + memory files** — continuity across sessions without persistent sessions |

Every OpenClaw failure isn't coincidental — it's a direct consequence of an architecture that chose async over blocking, in-memory over file-based, persistent over on-demand, and delegate-only over executor-capable.

Claude Code, as it exists today, has chosen the opposite at every decision point.

---

## Why Claude Outperforms OpenClaw

It's not just "more stable" — Claude Code is fundamentally different in how it handles orchestration. Here's what makes the entire OpenClaw architecture unnecessary:

### 1. Native Orchestration — No Custom Runtime

OpenClaw needs a Node.js runtime, gateway process, Discord bot integration, and cron system — all need to be maintained, debugged, and restarted when they crash.

Claude Code already has orchestration capability built-in. Agent tool, file system access, command execution — all native. No need to build infrastructure from scratch for things that already exist.

### 2. Blocking Delegation That Guarantees Success

OpenClaw: `sessions_spawn` → task sent → PM continues → results might come back, might not (81.5% fail).

Claude Code: `Agent tool` → task sent → **orchestrator waits** → results returned directly → orchestrator continues.

The difference isn't gradual. It's binary: unreliable vs reliable. 18.5% vs ~100%.

### 3. Orchestrator with Fallback — Not a Delegator Without a Safety Net

OpenClaw: PM is forbidden from executing anything. This rule is intentional — without constraint, the PM tends to execute on its own and hallucinate outside its domain. But when delegation fails 81.5% of the time, the PM has no fallback. It can only respawn endlessly.

Claude Code: Orchestrator has full tool access — read, write, edit, bash, grep, glob. Strategy: delegate to specialist first, if it fails, step in. Not "delegate only" or "execute only" — but delegation-first with fallback execution.

### 4. Memory That Connects Across Sessions — Without 12 Persistent Agents

OpenClaw: 12 agents × session memory × SQLite database = ~200MB footprint to store context that can be lost during compaction.

Claude Code: CLAUDE.md (project DNA loaded automatically) + memory files (file-based, persistent, crash-safe) + project tracker (markdown readable by anyone). Continuity across sessions without a single byte of memory database.

New sessions can immediately pick up where the previous one left off. No warm-up, no context loss, no "sorry, I forgot."

### 5. On-Demand Resource Usage — Not Always-On Waste

OpenClaw: 12 agents always active, 80% idle, 200MB+ of unproductive memory.

Claude Code: 1 active session. Sub-agents are spawned only when needed, destroyed after completion. Resources are used when needed, not reserved for agents that may never be active.

### 6. Zero Infrastructure Overhead

OpenClaw needs:
- Node.js runtime (gateway process)
- Discord bot + channel bindings
- SQLite databases per agent (13-27MB each)
- Cron system (which itself frequently errors)
- Memory management + compaction
- Config files with multiple backups (indicating frequent rollbacks)

Claude Code needs:
- One CLAUDE.md file
- One project-tracker.md file
- One decision-log.md file
- That's it.

### 7. Human-Readable State — Not a Black Box

OpenClaw: State scattered across 12 SQLite databases and PM session memory. To understand project status, you must query each agent's database and hope they're consistent.

Claude Code: Project state in `project-tracker.md` + decision history in `decision-log.md`. Anyone — developer, new session, or orchestrator — can open the file and immediately understand the latest status *and* the reasoning behind every choice. It can even be edited manually without special tools.

Summary comparison:

| Aspect | OpenClaw | Claude Code |
|--------|----------|-------------|
| Orchestration | Custom runtime + Discord | Native Agent tool |
| Delegation | Async, 81.5% fail | Blocking, ~100% success |
| Orchestrator | Delegate-only, no execution | Full execution capability |
| Cross-session memory | 12 SQLite DBs, lost during compaction | CLAUDE.md + file-based, crash-safe |
| Resource usage | 12 agents always-on, 80% idle | On-demand, near-zero waste |
| Infrastructure | Gateway + bot + cron + SQLite | CLAUDE.md + tracker file |
| State visibility | Black box (12 databases) | Human-readable markdown |
| Recovery after crash | Unreliable (in-memory state) | Reliable (file-based state) |
| Cross-session continuity | Depends on sessions staying alive | File-based, always connected |
| Mode selection | Always project mode, no escape | Auto-prompt on session start, switch anytime |
| Status updates | PM must manually write | Proactive — auto-writes after every event |
| Decision history | Scattered in threads and memory | Structured log with reasoning and impact |

---

## Key Takeaways

1. **Asynchronous without reliable delivery is an anti-pattern.** If the orchestrator can't reliably receive results, the entire orchestration layer only adds noise without adding reliability.

2. **Delegation without fallback is a trap.** The "NO EXECUTION" rule makes sense — an unconstrained PM tends to execute on its own and hallucinate. But when the delegation mechanism itself is unreliable, this constraint becomes a dead end. Solution: delegate first, fallback to execution. Not one or the other — but both with clear priorities.

3. **State must be persistent on disk, not in session memory.** Sessions crash, compact, timeout. Files don't.

4. **Fewer but controlled agents > many but unreliable agents.** 12 agents with 81.5% failure is worse than 1 orchestrator + on-demand agents with near-100% success.

5. **Agentic concepts aren't wrong — the implementation needs to be chosen carefully.** Hierarchy, persona, and delegation remain valid. But the delivery mechanism must be blocking, state must be persistent, and lifecycle must be on-demand.

6. **Continuity doesn't require always-alive sessions.** File-based memory and project trackers provide cross-session continuity without the cost and complexity of persistent agents.

7. **Record why, not just what.** Project trackers tell you the status. Decision logs tell you the reasoning. Two weeks from now, "we chose Midtrans" is a fact. "We chose Midtrans because QRIS and GoPay are essential for the Indonesian market and it only needs KTP" is knowledge. Both are needed for true project continuity.

---

The first iteration (OpenClaw) taught what NOT to do. The second iteration (ClaudeClaw) proved that the same concepts — agentic delegation, persona system, project tracking — can work well when the implementation is correct.

Sometimes, the solution isn't adding complexity. It's removing it.

And sometimes, an existing platform — in this case Claude Code — already has everything needed to patch the flaws of a custom architecture. No need to reinvent the wheel. Just use the existing wheel correctly.

---

*Similar experiences with multi-agent orchestration? Or a different approach? Drop a comment or DM.*

#MultiAgentAI #AIArchitecture #SoftwareEngineering #LLMOrchestration #ClaudeCode #LessonsLearned
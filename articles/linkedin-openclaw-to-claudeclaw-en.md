# ClaudeClaw: Shipping OpenClaw's Best Ideas on a Framework That Actually Works

What do you do when a multi-agent orchestration platform fails 81.5% of the time? You don't fix it. You take its best ideas and rebuild them on a foundation that works.

---

## What Is ClaudeClaw?

ClaudeClaw is a lightweight orchestration framework built on top of Claude Code. It gives you the multi-agent workflow that platforms like OpenClaw promise — specialist personas, project state tracking, task delegation — but implemented with blocking delegation, file-based persistence, and zero infrastructure overhead.

One orchestrator. On-demand specialists. State on disk. No cron watchdogs, no 12 persistent agents, no 200MB memory footprint.

The result: **near-100% delegation success rate** instead of 18.5%.

But to understand why ClaudeClaw exists, you need to understand what we went through with OpenClaw.

---

## The OpenClaw Experience: Great Concept, Broken Delivery

Over the past month, I built an agentic squad on OpenClaw — a local multi-agent AI orchestration platform. The architecture was ambitious: 12 persistent agents, coordination via Discord, and a Project Manager (PM) agent named "Bunga" whose job was to delegate all work to sub-agents.

CEO(Human) → PM → backend-dev, frontend-dev, qa-tester, devops, and so on. Clean hierarchy, clear task division.

On paper, perfect.

In reality? **81.5% of delegated tasks failed to return to the PM.** Out of 92 tasks sent to backend-dev, only 17 had their results successfully received.

That number isn't a bug — it's a direct consequence of an architecture that chose asynchronous communication without guaranteed delivery.

### The Six Problems That Broke Everything

**1. Fire-and-forget delegation.** OpenClaw's `sessions_spawn` sends a task and immediately moves on. The "auto-announce" mechanism that's supposed to return results? It fails 81.5% of the time. The PM never knows: did the task succeed? Fail? Or disappear?

**2. A delegate-only PM with no fallback.** The PM has an explicit "NO EXECUTION" rule — can't read, write, or run commands. This is intentional (unconstrained PMs hallucinate). But when delegation fails 81.5% of the time, a PM that can't execute has no safety net. It can only re-spawn endlessly.

**3. Premature re-spawning.** The PM assumes tasks failed and respawns. Average time between spawns: 47-200 seconds. Context loading alone takes 30-60 seconds. Many agents haven't finished loading before being replaced.

**4. Session compaction swallowing data.** The PM session ran for 7+ days, accumulating 2.2MB. When compacted, in-flight messages — sub-agent results being delivered — can be lost.

**5. 12 persistent agents, 80% idle.** Each agent has its own session, memory database (13-27MB), and workspace. Only 2-3 are active at any time. 80% of allocated resources are unproductive.

**6. Cron watchdog — a solution that isn't.** To handle the idle PM, a cron job was created every 15 minutes. A task finishing at minute 2 waits until minute 15. And the cron itself frequently errors and gets disabled. A system that needs a cron watchdog to function has a fundamental design flaw.

---

## The Pivot: Don't Fix It — Port the Good Parts

From all those failures, I didn't fix OpenClaw. I asked a different question:

> What if the agentic concepts from OpenClaw aren't wrong — just the implementation?

CEO → PM → specialist hierarchy? Good idea. Task delegation with persona-specific prompts? Good idea. Project state tracking? Good idea.

What failed was the delivery mechanism. So instead of patching an unreliable platform, I rebuilt the concepts on Claude Code — which already had everything needed: blocking agent delegation, file system access, command execution, and a built-in memory system.

ClaudeClaw was born.

---

## What We Kept from OpenClaw

These concepts survived the migration. Same ideas, radically different implementation.

### 1. Agentic Delegation — But Blocking

OpenClaw concept: PM delegates tasks to specialist sub-agents.
OpenClaw implementation: Asynchronous, fire-and-forget, 81.5% failure.
**ClaudeClaw implementation:** Agent tool that's blocking — orchestrator spawns sub-agent with persona prompt, waits for results, then continues. Every delegation succeeds because results are returned directly.

The difference isn't gradual. It's binary: unreliable vs reliable. 18.5% vs ~100%.

### 2. Persona System — But Lightweight

OpenClaw concept: Each agent has a 117-276 line SOUL.md defining personality, skills, and constraints.
OpenClaw implementation: 12 agents × ~300 lines = 3,600+ lines of context. Always loaded, always consuming memory.
**ClaudeClaw implementation:** Compressed persona prompt ~40 lines, embedded directly in the task description when spawning. Persona is only loaded when needed. Agents that aren't spawned consume zero context.

### 3. Project Tracking — But Persistent on Disk

OpenClaw concept: PM holds project state in session memory (SQLite + in-memory).
OpenClaw implementation: State is lost when sessions crash, compact, or timeout.
**ClaudeClaw implementation:** Project state in `project-tracker.md` — a markdown file that any session can read. Crash-safe, session-agnostic, human-readable, and version-controllable.

### 4. Specialization — But On-Demand

OpenClaw concept: 12 specialist agents running simultaneously.
OpenClaw implementation: 80% idle, 200MB+ of unproductive memory.
**ClaudeClaw implementation:** 1 active orchestrator. Sub-agents spawned on-demand, destroyed after completion. Resources used only when needed.

### 5. Orchestrator with Fallback — Not Delegate-Only

OpenClaw: PM forbidden from executing anything. When delegation fails, no fallback.
**ClaudeClaw:** Orchestrator has full tool access — read, write, edit, bash, grep, glob. Strategy: delegate to specialist first. If it fails, step in. Delegation-first with fallback execution.

---

## What We Threw Away

| OpenClaw Feature | Why We Dropped It |
|---|---|
| 12 persistent agents | 80% idle, 200MB waste. On-demand is better. |
| Discord coordination | Claude Code has native agent tool. No middleware needed. |
| SQLite memory per agent | File-based state is crash-safe and human-readable. |
| Cron watchdog | Orchestrator is active when session is open. No polling needed. |
| sessions_spawn (fire-and-forget) | Blocking delegation. 18.5% → ~100% success. |
| PM deny list (no execution) | Delegate first, fallback to execution. Safety net matters. |
| Node.js runtime + gateway | Zero infrastructure. CLAUDE.md + tracker file. |

---

## What We Added That OpenClaw Never Had

Three features we built after running ClaudeClaw in production — solving problems OpenClaw never addressed.

### Auto Mode Selection — No More "I Forgot to Switch"

OpenClaw's PM always ran in project mode. No way to have a casual conversation without orchestration overhead.

In ClaudeClaw, every new session starts with a mode prompt:

```
Welcome to ClaudeClaw! Choose your mode:
1. Project Mode — Orchestrate agents, track state, persist progress across sessions
2. Normal Mode — Regular Claude Code, no delegation or state tracking
```

No need to remember a command. The framework proactively asks. And you can switch mid-session anytime.

### Proactive Status Updates — The Tracker Writes Itself

OpenClaw's PM only updated project status when manually prodded. Status was outdated, incomplete, or missing.

In ClaudeClaw, the tracker **updates itself automatically** after every qualifying event:

| When | What gets written |
|------|-------------------|
| Agent task completes or fails | Status, phase, last activity |
| Milestone reached | Phase, next steps |
| Blocker found or resolved | Blockers field |
| Decision made | Decision logged with reasoning |
| User switches to normal mode | Mode set to `normal` |

You never have to ask "please update the status." It happens as part of the workflow.

### Decision Log — Recording Why, Not Just What

This is the feature I wish we had from day one.

Every project makes decisions: tech stack, pricing model, architecture choices. When you resume after two weeks, you remember *what* you decided, but often forget *why*. That leads to re-litigating choices that were already made — or reversing a decision without understanding the original reasoning.

OpenClaw had no mechanism for this. Decisions were scattered across Discord threads and session memory, impossible to reconstruct.

ClaudeClaw now has a `decision-log.md` that records every significant decision with:

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

## Before and After: The Full Comparison

| Aspect | OpenClaw | ClaudeClaw |
|--------|----------|-------------|
| Orchestration | Custom runtime + Discord | Native Agent tool |
| Delegation | Async, 81.5% fail | Blocking, ~100% success |
| Orchestrator | Delegate-only, no fallback | Full execution capability |
| Cross-session memory | 12 SQLite DBs, lost during compaction | CLAUDE.md + file-based, crash-safe |
| Resource usage | 12 agents always-on, 80% idle | On-demand, near-zero waste |
| Infrastructure | Gateway + bot + cron + SQLite | CLAUDE.md + tracker + decision log |
| State visibility | Black box (12 databases) | Human-readable markdown |
| Mode selection | Always project mode, no escape | Auto-prompt on session start, switch anytime |
| Status updates | Manual (often forgotten) | Proactive — auto-writes after every event |
| Decision history | None (scattered in threads) | Structured log with reasoning and impact |
| Recovery after crash | Unreliable (in-memory state) | Reliable (file-based state) |

---

## The Result

| Metric | OpenClaw | ClaudeClaw |
|--------|----------|-------------|
| Delegation success rate | 18.5% | ~100% |
| Re-spawn interval | 47-200 seconds | N/A (not needed) |
| Persona prompt per agent | 117-276 lines | ~40 lines compressed |
| Active agents | 12 persistent | 1 + on-demand |
| Memory DB footprint | ~200 MB | 0 (file-based) |
| Infrastructure needed | Node.js + Discord + cron | None |
| Status update mechanism | Manual | Proactive (automatic) |
| Decision tracking | None | Structured log |

---

## What About Cost? OpenClaw Is Free, Claude Code Is Expensive

Here's the obvious objection: OpenClaw is open-source and free. Claude Code is not.

That's true. But "free" and "cheap" are different things.

### The Real Cost of "Free"

OpenClaw is free to install. But 81.5% delegation failure means:
- Every task that fails is time wasted — your time, waiting for results that never come
- Re-spawning agents costs API tokens on tasks that were already running
- 12 persistent agents running 24/7, even when idle, burning through your API quota
- The PM's cron watchdog, re-spawns, and session overhead all consume tokens on unproductive work

You're not paying for OpenClaw itself. You're paying for the API calls that go nowhere. The hidden cost of unreliability.

### The Affordable Path: ClaudeClaw + Cost-Effective Models

ClaudeClaw is a framework, not a product. It works with Claude Code, but Claude Code supports multiple model providers — including affordable ones.

The most cost-effective approach I've found: **using the ClaudeClaw framework with open-source cloud models like GLM-5.1:Cloud at $20/month**, which resets weekly. That's enough to handle multiple projects per week with near-100% delegation success.

Compare the real cost:

| Cost Factor | OpenClaw (Free) | ClaudeClaw + GLM-5.1:Cloud |
|---|---|---|
| Platform cost | $0 | $20/month |
| API tokens wasted on failed delegations | High (81.5% fail rate) | Near zero (~100% success) |
| API tokens on idle agents | 12 agents × 24/7 | 1 orchestrator + on-demand |
| Re-spawn cost | High (47-200s intervals) | $0 (blocking, no re-spawns) |
| Time cost of debugging failed delegations | Significant | Minimal |
| Infrastructure | Node.js + Discord + cron | None |

**$20/month with near-100% success is cheaper than $0/month with 81.5% failure.** You're not paying for the framework — you're paying for reliability. And when your time has value, reliability is the cheapest option.

---

## Key Takeaways

1. **Asynchronous without reliable delivery is an anti-pattern.** If the orchestrator can't reliably receive results, the entire orchestration layer only adds noise without adding reliability.

2. **Delegation without fallback is a trap.** The "NO EXECUTION" rule makes sense — an unconstrained PM tends to execute on its own and hallucinate. But when the delegation mechanism itself is unreliable, this constraint becomes a dead end. Delegate first, fallback to execution.

3. **State must be persistent on disk, not in session memory.** Sessions crash, compact, timeout. Files don't.

4. **Fewer but controlled agents > many but unreliable agents.** 12 agents with 81.5% failure is worse than 1 orchestrator + on-demand agents with near-100% success.

5. **Agentic concepts aren't wrong — the implementation needs to be chosen carefully.** Hierarchy, persona, and delegation remain valid. But the delivery mechanism must be blocking, state must be persistent, and lifecycle must be on-demand.

6. **Continuity doesn't require always-alive sessions.** File-based memory and project trackers provide cross-session continuity without the cost and complexity of persistent agents.

7. **Record why, not just what.** Project trackers tell you the status. Decision logs tell you the reasoning. Two weeks from now, "we chose Midtrans" is a fact. "We chose Midtrans because QRIS and GoPay are essential for the Indonesian market and it only needs KTP" is knowledge. Both are needed for true project continuity.

8. **"Free" is not the same as "cheap."** An $0/month platform with 81.5% failure rate costs more in wasted time, API tokens, and debugging than a $20/month framework with near-100% success. When your time has value, reliability is the cheapest option.

---

The first iteration (OpenClaw) taught what NOT to do. The second iteration (ClaudeClaw) proved that the same concepts — agentic delegation, persona system, project tracking — can work well when the implementation is correct.

Sometimes, the solution isn't adding complexity. It's removing it.

And sometimes, an existing platform — in this case Claude Code — already has everything needed to patch the flaws of a custom architecture. No need to reinvent the wheel. Just use the existing wheel correctly.

https://github.com/arywidos/claudeclaw

---

*Similar experiences with multi-agent orchestration? Or a different approach? Drop a comment or DM.*

#MultiAgentAI #AIArchitecture #SoftwareEngineering #LLMOrchestration #ClaudeCode #LessonsLearned
# Worked Example: Complete Five-Question RCA Walkthrough

> **Parent skill**: [deep-trace SKILL.md](../SKILL.md)
>
> This is the primary worked example from the deep-trace skill. It walks through 
> a complete five-question RCA of a real production incident: a dispatcher agent 
> sent 7 identical routing requests in 50 seconds.

## Worked Example

This example walks through a complete five-question RCA of a real production
incident. It was chosen as the primary worked example because it is the
cleanest case: single root cause, clear error signature pair, and action items
that generalize to any event-driven agent system.

> **Source**: Case: Duplicate Routing Request Burst (originally traced from production issue #412) —
> Dispatcher agent sent 7 identical routing requests in 50 seconds.

### Trigger
Signal **S3** (Unproductive Loop) + **S6** (Temporal Anomaly): Dispatcher and
Router exchange messages repeatedly without progress, and 7 nearly-identical
routing requests appear within 50 seconds. [→ When to Use: Signal Detection Triggers]

### Q1: Facts *(pure timeline, no causal words)* [→ Question 1]

```
10:28:14 — Dispatcher sends routing request #1 for [a task]
10:28:18 — Dispatcher sends routing request #2 (4s later, identical content)
10:28:24 — Dispatcher: "test" (unusual — agent generating exploratory output)
10:28:30 — Dispatcher sends routing request #3
10:28:34 — Dispatcher sends routing request #4
10:28:41 — Dispatcher sends routing request #5
10:28:44 — Dispatcher sends routing request #6
10:28:48 — Dispatcher sends routing request #7
10:30:37 — Router agent responds with routing result
10:31:07 — Dispatcher acknowledges
... (continues @mentioning router and executor for ~40 more minutes)
```

**Q1 constraint check**: ✅ No causal words. Pure timestamped event sequence.
Includes the anomalous "test" entry — don't filter out data that looks odd.

### Q2: Direct Cause *(pinpoint to concrete element)* [→ Question 2]

Dispatcher instructions lacked two specific guards:

1. **No pre-routing dedup check**: "Before routing, check if this task was
   already routed" — absent from instructions.
2. **@mention re-trigger vulnerability**: When replying to a completed agent,
   the dispatcher used agent @mention syntax (which re-triggers the mentioned
   agent) instead of a passive issue link. Each reply became a new trigger
   event, causing the dispatcher to re-evaluate "needs routing?" → true →
   route again.

**Root mechanism**: The dispatcher is stateless across runs. Each trigger event
(including its own replies) causes it to restart from scratch. With no "check
existing results" step, every restart produces the same conclusion: "this task
needs routing."

**Transient check**: ❌ Not transient — the pattern is deterministic given the
instruction gap. Continue to Q3.

### Q3: Why It Survived *(exact process gaps)* [→ Question 3]

| Gap | Location | Type |
|-----|----------|------|
| No "check existing routing results" step | Dispatcher routing workflow | Missing check |
| @mention rules in CLAUDE.md covered "silence" prevention but not "re-trigger" prevention | Global instructions | Wrong scope |
| No rule: "If executor already responded 'done', do not @mention them again" | Dispatcher instructions | Missing check |

### Q4: Why It Wasn't Detected *(detection chain)* [→ Question 4]

| Detection Node | Should Have | Actually Did | Break Reason |
|---------------|-------------|--------------|--------------|
| Real-time anomaly detection | Flag ≥3 near-identical comments from same agent in <60s | Nothing | No volume/velocity monitoring |
| First occurrence (request #2) | Detect duplicate and suppress | Sent identical request | No dedup logic |
| Router response | Short-circuit: "already routed, stop asking" | Processed each request individually | Router treats each request as independent |
| Human observation | Notice 7-request burst immediately | Noticed ~12.75h later (next review cycle) | No real-time alert |

**Detection delay**: ~12.75 hours. The burst happened at 10:28; it was noticed
during the next review cycle around 23:00–00:00.

**Key insight**: The detection chain existed (humans review) but had no
real-time component. The delay meant 7 redundant operations executed and the
Router wasted processing cycles before anyone noticed.

### Q5: Root Pattern *(principle + substitution test)* [→ Question 5]

> When a stateless agent is re-triggered by events on the same context, and
> has no cross-run state awareness, it re-evaluates task needs from scratch
> and may re-execute already-completed actions. Instructions must include a
> mandatory "check existing results before acting" step.

**Substitution test**:
```
Original: "When a stateless agent is re-triggered by events on the same
context, and has no cross-run state awareness..."

Substituted: "When [a stateless processing unit] is re-triggered by events on
the same [work item], and has no cross-run state awareness..."
→ ✅ Holds. This is a universal principle for any event-driven stateless system.
```

### Classification & Error Signature [→ RCA Output Template]

| Dimension | Value | Rationale |
|-----------|-------|-----------|
| **Classification** | `流程缺失` (primary) | The core gap is a missing process step: "check before acting" |
| **error_signature** (primary) | `I-RED` | Interaction Redundancy — the observable symptom |
| **error_signature** (secondary) | `P-REP` | Repeated Execution — the mechanism |

Note: Classification (`流程缺失`) and error_signature (`I-RED`) are different
dimensions. The classification says *where the fix goes* (add a process step);
the error_signature says *what broke* (redundant agent interactions).

### Action Items [→ Action Item Decomposition]

#### [action] Add pre-routing dedup check to dispatcher instructions

**Problem**: Dispatcher has no cross-run state awareness; re-evaluates from
scratch on every trigger.

**Fix**: Add to dispatcher instructions: "Before routing, check if this task
already has a routing result. If yes, link to existing result; do not re-route."

**Acceptance Criteria**: Given a task that was routed within the last 24h,
re-triggering the dispatcher on the same task produces ≤ 1 routing action
(the original link, not a new route).

**error_signature**: `P-REP`

#### [action] Add @mention guard for completed agents

**Problem**: @mentioning a completed agent re-triggers it, creating feedback loops.

**Fix**: Add to CLAUDE.md global rules: "When referring to work done by an
agent whose task is complete, use the issue link, not agent @mention."

**Acceptance Criteria**: After an agent marks a task complete, subsequent
references to that task in comments use `mention://issue/` links, never
`mention://agent/` for the completed agent.

**error_signature**: `I-RED`

#### [action] Add burst detection monitoring (backlog)

**Problem**: 7 identical requests in 50 seconds went undetected for 12+ hours.

**Fix**: Add real-time monitoring rule: if the same agent posts ≥ 3
near-identical comments on the same issue within 60 seconds, trigger a
moderator alert.

**Acceptance Criteria**: A 7-request burst produces an alert within 60 seconds
of the 3rd request.

**error_signature**: `I-RED`
**Placement**: Backlog (depends on monitoring infrastructure not yet available).
**Unblock condition**: Monitoring infrastructure deployed.

---

### What A Shallower Approach Would Miss

| Shallower Approach | Why It Fails |
|-------------------|--------------|
| "Tell dispatcher not to repeat itself" | Too vague to be enforceable. The agent needs a specific check step, not a behavioral admonition. |
| "Replace the dispatcher" | The dispatcher isn't broken — its instructions lack a guard step. Any replacement without the guard would have the same failure. |
| "Rate-limit agent actions" | Masks the symptom (fewer duplicates) without fixing the cause (no pre-action check). A rate-limited dispatcher still re-evaluates incorrectly; it just produces fewer errors. |
| Skip Q4 (detection chain) | Without Q4, the 12.75h detection delay is invisible. The monitoring gap would persist, and the next incident would have the same delay. |
| Skip substitution test in Q5 | Without it, the principle might stay bound to "dispatcher" / "routing" — missing that this applies to any stateless agent in any event-driven system. |

### Chapter Cross-References

Each step in this example maps to a specific section of this skill:

| Step | References |
|------|-----------|
| Trigger identification | [When to Use](#when-to-use-this-protocol) — S3, S6 |
| Q1: Facts | [Question 1](#question-1-facts-事实) — timeline constraint |
| Q2: Direct Cause | [Question 2](#question-2-direct-cause-直接原因) — pinpoint constraint + transient check |
| Q3: Why It Survived | [Question 3](#question-3-why-it-survived-为什么存活) — gap types |
| Q4: Detection Chain | [Question 4](#question-4-why-it-wasnt-detected-为什么没发现) — detection table |
| Q5: Root Pattern | [Question 5](#question-5-root-pattern-根本模式) — substitution test |
| Classification | [RCA Output Template](#rca-output-template) — classification vs error_signature |
| Error codes | [Error Signature Classification](#error-signature-classification) — I-RED, P-REP |
| Action items | [Action Item Decomposition](#action-item-decomposition) — format + rules + backlog placement |

---



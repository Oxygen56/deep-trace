# Real Five-Question RCA Transcripts

Each example below is a real five-question analysis from a production multi-agent
system. The annotation explains what each question revealed, which SKILL.md
section it maps to, and what a shallower approach would have missed.

## Case Selection Rationale

Three cases were selected as full transcripts (out of 6 production cases
available) based on the following criteria:

| Case | Selection Rationale |
|------|-------------------|
| [OXY-412](#example-1-repeated-routing-requests-oxy-412) (Repeated Routing) | **Cleanest teaching case**: single root cause, clear error signature pair (`I-RED`/`P-REP`), action items generalize to any event-driven agent system. Ideal first example for new readers. |
| [OXY-571](#example-2-high-frequency-corrections-self-reflection-oxy-571) (Self-Reflection) | **Demonstrates recursive application**: the investigator ran the protocol on itself. This case produced the Correction Response Protocol and proves the methodology's credibility through equal application. |
| [OXY-177](#example-3-template-bound-rules-oxy-177) (Template-Bound Rules) | **Demonstrates meta-principle generation**: the RCA directly produced a reusable design principle ("template-embedded rules don't generalize") that now lives in the skill itself as Principle B. |

Two additional cases are summarized in the [Appendix](#appendix-additional-case-summaries):
- OXY-414 (File Delivery Unreachable) — demonstrates the `D-INC` pattern
- OXY-574 (Blocked Issues Timeout) — demonstrates multi-pattern RCA and "alert ≠ action"

---

## Example 1: Repeated Routing Requests (OXY-412)

> **Source**: [OXY-412](mention://issue/05abf3c1-4781-41fc-b1c8-f9d6ead1d894)
> **Trigger**: S3 (Circular mentions) + S6 (Timing anomaly)
> **SKILL.md ref**: This is the primary worked example in SKILL.md

**Scenario**: Dispatcher agent sent 7 identical routing requests in 50 seconds,
triggering the router agent to repeatedly respond "already routed."

### Q1: Facts [→ SKILL.md: Question 1]
```
10:28:14 — Dispatcher sends routing request #1
10:28:18 — Dispatcher sends routing request #2 (4s later, identical)
10:28:24 — Dispatcher: "test"
10:28:30 — Dispatcher sends routing request #3
... (4 more in next 34 seconds)
10:30:37 — Router responds with routing result
10:31:07 — Dispatcher acknowledges
... (continues @mentioning router and executor for 40 more minutes)
```

### Q2: Direct Cause [→ SKILL.md: Question 2]
Dispatcher instructions lacked:
- "Before routing, check if this task was already routed" (no dedup)
- "When replying to a completed agent, use issue link not agent @mention"
  (agent mention triggers re-execution)

Each trigger event caused dispatcher to run from scratch, re-evaluate
"needs routing" → true, and route again. No state across runs.

### Q3: Why It Survived [→ SKILL.md: Question 3]
- Routing workflow had no "check existing results" step
- CLAUDE.md mention rules covered silence-but-not-re-trigger prevention
- No concrete rule: "If executor said 'done', don't @mention them again"

### Q4: Why Not Detected [→ SKILL.md: Question 4]
**Detection delay**: ~12.75 hours (events at 10:28, detected at 00:00 next
cycle). No real-time anomaly detection for "same agent, same issue, ≥3
near-identical comments in <60 seconds."

### Q5: Root Pattern [→ SKILL.md: Question 5]
> When a stateless agent is re-triggered by events on the same context, and
> has no cross-run state awareness, it re-evaluates task needs from scratch
> and may re-execute already-completed actions. Instructions must include a
> mandatory "check existing results before acting" step.

**Substitution test**: ✅ Passes. Replace "stateless agent" → "[stateless
processing unit]", "re-triggered" → "[re-invoked]".

### RCA Output [→ SKILL.md: RCA Output Template]

| Field | Value |
|-------|-------|
| **Reflection Trigger** | S3 (Circular mentions) + S6 (Timing anomaly) |
| **Classification** | 流程缺失 |
| **error_signature** | `I-RED` (primary), `P-REP` (secondary) |
| **Solution** | Add pre-routing dedup check + @mention guard rule |

### Action Items [→ SKILL.md: Action Item Decomposition]

1. **[action] Add pre-routing dedup check** — Dispatcher checks for existing
   routing result before routing. AC: re-trigger produces ≤ 1 routing action.
   `error_signature`: `P-REP`
2. **[action] Add @mention guard for completed agents** — CLAUDE.md rule: use
   issue links, not agent @mentions, when referencing completed work.
   `error_signature`: `I-RED`
3. **[action] Add burst detection monitoring** (backlog) — Alert on ≥ 3
   near-identical comments in <60s. Unblock: monitoring infra deployed.

### Closure Result
Dispatcher dedup check and @mention guard deployed. No recurrence of multi-request
bursts in subsequent review cycles.

### What A Shallower Approach Would Miss
- Would add "don't repeat yourself" → too vague to be enforceable
- Q5's "stateless agent re-trigger" pattern is universal — applies to any
  agent in any event-driven system, not just this dispatcher
- The distinction between "agent is broken" and "instructions lack state
  awareness" changes the fix from "replace agent" to "add guard step"

---

## Example 2: High-Frequency Corrections / Self-Reflection (OXY-571)

> **Source**: [OXY-571](mention://issue/e8ec4a5f-2c2e-4cdc-bd83-078e1abd9576)
> **Trigger**: S7b (≥ 3 corrections on same topic within 24h) + self-reflection
> **SKILL.md ref**: Self-Reflection Protocol, Correction Response Protocol

**Scenario**: Inspector was corrected 4 times in 40 minutes on the same task.
The inspector ran the protocol on itself — demonstrating recursive self-application.

### Q1: Facts [→ SKILL.md: Question 1]
```
17:08 — User correction #1: "mentioning this project is inappropriate"
17:10 — Inspector adjusts, replies with ~3000-word analysis
17:18 — User correction #2: "too long-winded", 4 new requirements
17:19 — Inspector compresses but still uses internal jargon, hard numbers
17:44 — User correction #3: "don't use numbers, don't use jargon"
17:45 — Inspector adjusts, but writes "scanned OXY-526 output"
17:48 — User correction #4: "wrong! Not OXY-526 — scan the ENTIRE workspace"
17:50 — Inspector finally does full scan, gets it right
```

### Q2: Direct Cause [→ SKILL.md: Question 2]
Each correction fixed **only the specific point raised**, without re-aligning
to the original request ("scan entire workspace → extract resume highlights").
By correction #3, the reference point was "the last reply" not "the original
request" — causing scope to drift until even the scan target was wrong.

### Q3: Why It Survived [→ SKILL.md: Question 3]
- No "alignment confirmation" step between correction receipt and modification
- Inspector's instructions: respond quickly, no requirement to restate
  understanding before making changes
- No mechanism to re-anchor to original request after each correction

### Q4: Why Not Detected [→ SKILL.md: Question 4]
- Between corrections #1-#2: no self-check of "is this what user wanted?"
- Between #3-#4: the most critical break — reply explicitly said "scanned
  OXY-526" but never asked "user said full workspace — is OXY-526 the right
  scope?"
- The self-check that would have caught this: "What did the user originally
  ask for, and does my current scope match?"

### Q5: Root Pattern [→ SKILL.md: Question 5]
> When an executor receives sequential local corrections and each time only
> patches the specific point raised without re-aligning to the original
> objective, the correction sequence converges superficially but diverges
> substantively — because each correction's input is the previous incomplete
> understanding, not the original requirement.

**Substitution test**: ✅ Passes. Replace "executor" → "[agent]", "corrections"
→ "[feedback items]", "original objective" → "[original requirement]".

### RCA Output [→ SKILL.md: RCA Output Template]

| Field | Value |
|-------|-------|
| **Reflection Trigger** | S7b (4 corrections in 40 min) + self-reflection |
| **Classification** | 流程缺失 |
| **error_signature** | `U-UND` (primary), `E-FMT` (secondary) |
| **Solution** | Add alignment confirmation step between correction and modification |

### Action Items [→ SKILL.md: Action Item Decomposition]

1. **[action] Add Correction Response Protocol to instructions** — Four-step
   protocol: Align → Confirm → Execute → Verify. AC: After any correction,
   inspector restates understanding before making changes.
   `error_signature`: `U-UND`
2. **[action] Add 1-minute re-anchoring rule** — After receiving a correction,
   spend ≥ 1 minute re-reading the original request before responding.
   AC: Correction response time floor is 1 minute (not 30 seconds).
   `error_signature`: `U-UND`

### Closure Result
Correction Response Protocol deployed to inspector instructions. The protocol
was later extracted as a standalone section in SKILL.md (now applicable to all
agents, not just the inspector). No S7b recurrence from inspector since deployment.

### What A Shallower Approach Would Miss
- Would blame the agent's "understanding ability"
- Q3's "alignment confirmation step" is the fix — a process addition, not an
  ability upgrade
- This analysis directly produced the "Correction Response Protocol" now
  embedded in SKILL.md as an independent module
- The recursive self-application of the protocol is what makes it credible:
  the tool that finds root causes was itself subjected to the same tool

---

## Example 3: Template-Bound Rules (OXY-177)

> **Source**: [OXY-177](mention://issue/found)
> **Trigger**: User correction — missing issue mention link format
> **SKILL.md ref**: Rule Design Meta-Principles (Principle B), Question 5

**Scenario**: Manager referenced a child issue by text name only (no clickable
link), violating workspace convention. The RCA discovered a meta-principle about
how template-embedded rules fail to generalize.

### Q1: Facts [→ SKILL.md: Question 1]
```
12:52:44 — Manager creates child issue OXY-628. References it as **OXY-628**
           (bold text, no mention link).
           Required format: [OXY-628](mention://issue/<id>)
12:55:56 — User: "I have a global rule that child issue references must
           include jump links. Why did the manager only give text?"
```

### Q2: Direct Cause [→ SKILL.md: Question 2]
Manager instructions had mention link rules **only in template-specific
contexts** (briefing templates, decision tables). The comment was a "routing
notification" — a scenario not covered by any template, so the agent treated
the rule as non-applicable.

### Q3: Why It Survived [→ SKILL.md: Question 3]
The global rule (workspace CLAUDE.md) existed but was not replicated in the
manager's own instructions. Agents prioritize their own instructions over
global rules. The global rule was a standalone statement; the agent's local
rules embedded the requirement in templates. Template-bound rules don't
generalize to non-template scenarios.

### Q4: Why Not Detected [→ SKILL.md: Question 4]
- Daily patrol checks covered interaction signals, closure breaks, stage
  continuity — not issue reference format compliance
- Quality review dimensions: accuracy/completeness/usability/efficiency — not
  format compliance
- No automated lint / pre-comment format check
- Only detected by manual human observation

### Q5: Root Pattern [→ SKILL.md: Question 5]
> When a rule is expressed as a template or example within instructions, the
> system tends to treat it as specific to that template scenario. Universal
> rules must be declared independently of any template, with templates serving
> only as instantiations, not as the sole definition.

**Substitution test**: ✅ Passes. Replace "a rule" → "[a behavioral constraint]",
"template" → "[usage example]", "instructions" → "[system configuration]".

### RCA Output [→ SKILL.md: RCA Output Template]

| Field | Value |
|-------|-------|
| **Reflection Trigger** | User correction — format violation |
| **Classification** | 设计缺陷 (template-bound rules are a design-level pattern) |
| **error_signature** | `E-SKP` (process skip — the format check was skipped in non-template context) |
| **Solution** | Declare universal rules independently of templates; templates serve as instantiations only |

### Action Items [→ SKILL.md: Action Item Decomposition]

1. **[action] Extract mention-link rule from templates to standalone** — Move
   mention link requirement from template-specific sections to a standalone
   global rule in CLAUDE.md. AC: Agent produces mention links in all comment
   types (routing notifications, briefings, status updates), not just
   template-governed ones. `error_signature`: `E-SKP`

### Closure Result
Mention link rule extracted to standalone CLAUDE.md entry. The meta-principle
("template-embedded rules don't generalize") was added to SKILL.md as Principle B
of the Rule Design Meta-Principles section.

### What A Shallower Approach Would Miss
- Would just add "use mention links" to the routing template → same class of
  error would recur in the next non-template scenario
- Q5's principle about template-bound vs universal rules is the real fix
- This meta-principle is itself a reusable finding for any instruction design,
  not just mention links — it prevents an entire class of future errors
- The principle now lives in the skill itself, making the skill self-improving

---

## Appendix: Additional Case Summaries

The following cases are included as condensed summaries to demonstrate
additional error patterns without full five-question transcripts.

### Case A: File Delivery Unreachable (OXY-414)

> **Source**: [OXY-414](mention://issue/found)
> **Trigger**: S8 (Closure break) — agent claimed delivery, consumer found nothing
> **error_signature**: `D-INC` (primary)

**TL;DR**: Content agent wrote files to agent runtime isolated filesystem, not
shared workspace. Self-verification (`find`/`grep`) produced correct results
in the wrong domain. Three occurrences in 24 hours before formal RCA.

**Key insight**: The system cannot distinguish "execution failure" from
"execution success with unreachable output." Both look identical to the
consumer (0 files found), but require completely different fixes. Q4's
detection chain analysis revealed every feedback loop stayed at "agent didn't
write" — never penetrating to "why does the agent see the files but we don't?"

**Root pattern**: Self-verification in an isolated environment produces
correct-but-wrong-domain results. This is a distributed systems isolation
problem, not an agent-specific issue.

**Fix**: Expose write domain to execution units; add cross-environment
verification step.

**Why not a full transcript**: This case is inherently about infrastructure
(isolation between runtime and shared filesystem), making it less universally
applicable than the three main cases. The pattern is valuable but the fix is
platform-specific.

---

### Case B: Blocked Issues Timeout (OXY-574)

> **Source**: [OXY-574](mention://issue/found)
> **Trigger**: S5 (Blocked timeout) — 6 issues blocked 57–263 hours
> **error_signature**: `P-RTE` + `E-SKP`

**TL;DR**: 6 issues blocked for 57–263 hours, all waiting on human input.
Inspections ran every 1–3 hours but could only @remind — no escalation
authority.

**Key insight**: Detection chain was intact (signals fired, inspections ran,
comments posted). Response chain was broken (@mentions → human → no response →
repeat). The self-correcting nature of the protocol was demonstrated: Q1's
data contradicted the initial signal ("no inspections"), leading to a corrected
premise — inspections were present, response was absent.

**Three root patterns found**:
1. Alert without action → timeout action needed
2. Misroute without self-heal → automatic re-routing needed
3. Completion signal unconsumed → auto-advance on upstream completion

**Fix**: Add escalation timeout (N reminders → auto-escalate), cross-domain
capability check at routing, auto-promotion on child completion.

**Why not a full transcript**: This case found three distinct root patterns,
each requiring a different fix. As a teaching example, it's too complex for a
first-time reader. The three main cases are single-root-pattern examples that
build understanding incrementally before tackling multi-pattern analyses.

---

## Pattern Cross-Reference

Looking across all five cases (3 full + 2 appendix), recurring meta-patterns
emerge:

| Meta-Pattern | Cases | General Form |
|-------------|-------|--------------|
| Template-bound rules don't generalize | #3, #2 | Rules embedded in templates are treated as template-scoped; universal rules must be standalone |
| Stateless re-trigger without guard | #1 | Agents restarting from scratch on every event need explicit "check before acting" steps |
| Alert ≠ Action | B | Detection without escalation authority creates infinite notification loops |
| Closed verification loop | A, #2 | Self-verification in an isolated environment produces correct-but-wrong-domain results |
| Symptom patching without re-alignment | #2 | Sequential local fixes diverge from original intent without re-anchoring step |

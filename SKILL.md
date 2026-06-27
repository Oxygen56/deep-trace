---
name: deep-trace
description: "Agent Root Cause Analysis Protocol — A systematic five-question methodology adapted from Toyota's 5 Whys for diagnosing failures in AI agent systems. Covers trigger scenarios, the five-question framework, error signature classification, action decomposition, rule design meta-principles, and self-reflection. When an agent makes a mistake, this protocol traces back through the instruction and process layers to find the true root cause instead of patching symptoms."
user-invocable: true
allowed-tools: Bash(multica *)
---

# deep-trace: Agent Root Cause Analysis Protocol (五问审查)

When an AI agent produces incorrect output, the instinct is to fix the symptom —
tell it "don't do that" or add a one-line rule. These patches accumulate, inflate
instructions, and the same class of error recurs under different names.

This protocol breaks that cycle. It is a **systematic five-question drill** that
forces the investigator to trace every error back to its root cause in the
**instruction/process/design layer**, not the symptom layer.

It was developed and battle-tested over 60+ agent collaboration scenarios in
production, where it reduced instruction patch inflation and made errors
preventable at the architectural level.

---

## Scope Declaration

This skill covers **post-mortem Root Cause Analysis** using the five-question
drill. It is one component of a broader quality assurance system:

| Component | Scope | Relationship |
|-----------|-------|-------------|
| **deep-trace** (this skill) | Post-mortem RCA — five-question drill after an incident occurs | This skill |
| **agent-pregating-check** (future) | Pre-output self-check — "产出前三问" before every reply | Upstream: prevents errors before they ship |
| **agent-review-standards** (future) | Daily quality review — four-dimension standards + three iron rules | Parallel: handles routine quality, not deep RCA |
| **patrol-signal-detection** (future) | Real-time signal monitoring — S1-S9 detection system | Upstream: detects when RCA should be triggered |

**When to use RCA vs. routine review**: If the error is a one-off that can be
fixed with a single, simple instruction edit → use routine review standards.
If the error recurs, involves multiple agents or system layers, or reveals a
design-level gap → use this RCA protocol.

---

## When to Use This Protocol

Run this protocol when any of these signals appear. The protocol works on **any
agent in any system** — it does not require specific platforms or tooling.

### Signal Detection Triggers (Inline Definitions)

These nine signal types are defined here so this skill is self-contained
(no external dependency on patrol-signal-detection):

| Code | Signal | Definition | Example |
|------|--------|------------|---------|
| **S1** | Correction | User or downstream agent explicitly negates output | "wrong", "not what I asked", "redo" |
| **S2** | Repeated explanation | Same point explained ≥ 2 times on the same issue | "I already said…" appearing across comments |
| **S3** | Circular mentions | Two agents @mention each other ≥ 3 rounds without progress | Dispatcher ↔ Router ping-pong on the same task |
| **S4** | Output rejected | Delivery explicitly rejected as unusable | Task pulled from review back to in-progress |
| **S5** | Blocked timeout | Task blocked > 48h with no resolution | Issue stuck on "waiting for human" for days |
| **S6** | Timing anomaly | Unusual delay or burst pattern in agent activity | 7 identical operations in 50 seconds |
| **S7** | Repeated correction (intra-topic) | ≥ 3 corrections from the same person on the same topic within 24h | User corrects format, then scope, then target |
| **S8** | Closure break | Closed-loop action chain is broken | Agent claims "done" but downstream finds nothing |
| **S9** | Flow stagnation | Task stays in the same stage beyond expected duration | Issue in "in-progress" for 5× the normal cycle time |

### Non-Signal Triggers

In addition to the signal-detection triggers above, run this protocol when:

- **User explicitly requests RCA**: "Why did this happen?" / "Do a five-whys on this"
- **Agent self-triggers reflection**: The agent notices a pattern in its own errors
  and decides to investigate before being asked

### When NOT to Use This Protocol

Do **not** start the five-question drill when:

- The error is a **single, isolated typo or formatting glitch** that the agent
  fixed immediately and no systemic pattern is suspected
- The error is **already covered** by an existing RCA whose action items are
  still in progress (link to the existing analysis instead)
- The error is a **known transient** (e.g., a one-time network timeout that
  resolved on retry, with no configuration gap)
- The situation is better handled by **routine review standards** (accuracy/
  completeness/usability/efficiency checks) rather than deep RCA

### Self-Reflection Triggers

The protocol applies **recursively to the investigator itself**. When the
investigator is corrected or makes an error, it runs the protocol on itself:

- "Why did I miss X in my previous analysis?"
- "Why did I give relative/hedged quality judgments?"
- "Why did I fix symptoms instead of the root cause?"

---

## The Five Questions

Each question has a specific methodological constraint. Skipping or softening a
constraint invalidates the analysis.

### Question 1: Facts (事实)

> What happened? Describe the timeline without causal judgment.

**Constraint**: No causal words. If the answer contains "because", "due to",
"caused by" — rewrite. This question is pure observation.

**Format**: Timestamped sequence of events. Who did what at what time. Include
exact quotes and identifiers.

**Example (good)**:
```
15:21:34 — Agent Alice claimed "files written to shared workspace"
15:22:36 — Agent Bob ran find — 0 files found. Commented "second time"
15:23:21 — Alice re-claimed "18 files written", included find output showing 20 files
15:24:15 — Bob found 0 files again, switched to workaround
```

**Example (bad — contains causal judgment)**:
```
Alice kept claiming files were written because she was checking in the wrong
environment, causing Bob to waste time verifying.
```

### Question 2: Direct Cause (直接原因)

> What specific statement, operation, or decision directly caused this?

**Constraint**: Must pinpoint to a concrete, verifiable element — a specific
line in instructions, a specific tool call, a specific missing check. Cannot be
a vague category like "communication issue" or "quality problem".

#### ⚠️ Transient Termination Branch

If the suspected cause is "context truncation" or another runtime glitch:

1. **Note the evidence** — what specifically makes you suspect truncation?
2. **Re-run verification** — reproduce the scenario with full context
3. **If the problem disappears** → mark as `transient` and **terminate the
   analysis immediately**. Do not proceed to Q3-Q5. Do not produce action items.
   This is critical: treating a transient as a genuine defect creates phantom
   "fixes" that solve nothing and add instructions bloat.
4. **If the problem persists** → it is not transient; continue to Q3.

**Why this matters**: Traditional 5 Whys has no "transient termination" concept.
Agent systems have ephemeral runtime states (context windows, token limits,
model sampling) that can cause one-off failures. Misclassifying these as
systematic defects is a common failure mode that this protocol explicitly guards
against.

**Example (good)**:
```
Direct cause: Alice's Write tool placed files in the agent runtime's isolated
filesystem, not the shared workspace. Alice's verification (find/grep) ran
in the same isolated environment, so it correctly found the files — but in the
wrong domain. Alice's instructions contain no cross-environment reachability check.
```

**Example (bad)**:
```
Direct cause: file system problems between agents.
```

### Question 3: Why It Survived (为什么存活)

> Which specific step in the existing process failed to catch this?

**Constraint**: Name the exact process gap with its location. "The process was
incomplete" is not specific enough. Instead: "The routing step (step 3 in the
workflow) has no cross-domain capability check. Once a task is misrouted, there
is no automated re-routing mechanism — it stays on the wrong entity forever."

Look for these gap types:
- **Missing check**: A step that should exist but doesn't
- **Wrong scope**: A check exists but covers the wrong domain
- **Closed loop**: A check exists in the wrong environment (e.g., agent verified
  in its own sandbox, not the shared environment)

**Example (good)**:
```
Why it survived:
1. Missing check: `actions/validate.sh` line 12 checks `status=0` but never
   inspects stdout content. A failing operation that returns exit code 0 with
   error text passes validation.
2. Wrong scope: The linter covers agent output format but not issue reference
   link format — so missing `mention://` links are never flagged.
3. Closed loop: Post-write verification runs in the agent's isolated sandbox,
   confirming files exist locally but never checking the shared workspace.
```

**Example (bad)**:
```
Why it survived: the process was incomplete and quality checks were insufficient.
```

### Question 4: Why It Wasn't Detected (为什么没发现)

> Where exactly did the detection chain break? What was the delay from
> occurrence to detection?

**Constraint**: Must identify the specific detection node that failed. Map the
full detection chain from occurrence to eventual discovery, and mark the break
point.

**Format**: Table of detection nodes → expected behavior → actual behavior →
break reason.

| Detection Node | Should Have | Actually Did | Break Reason |
|---------------|-------------|--------------|--------------|
| First occurrence feedback | Escalate to infrastructure investigation | Re-filed as agent execution quality issue | No escalation mechanism |
| Second occurrence (same pattern) | Auto-trigger platform investigation | Manual repeat of same feedback | No pattern recognition |
| Third occurrence | Emergency escalation | Forced workaround | Escalation path doesn't exist |

**Key insight**: Always distinguish between "the agent didn't do it" and "the
agent did it but the result is unreachable". They look identical in symptoms but
require completely different fixes.

### Question 5: Root Pattern (根本模式)

> What is the universal principle behind this failure?

**Constraint**: Express the finding as a general principle. Then run the
**proper noun substitution test**: replace all specific names (agent names,
platform names, squad names, file paths) with generic placeholders. If the
statement still holds, it is a principle. If it collapses, it is a patch.

#### Self-Check Procedure (Mandatory)

Before accepting a Q5 answer, execute these verification steps:

1. **Replace all proper nouns** in the principle with placeholders
   (e.g., "Dispatcher" → "[a routing agent]", "MUL-123" → "[a task]",
   "multica" → "[the platform]").
2. **Re-read the substituted statement.** Does it still express a coherent,
   universally applicable rule?
3. **If YES** → the principle is valid. Proceed.
4. **If NO** → the statement is a patch bound to specific names. Go back and
   rewrite Q5 at a higher level of abstraction. Repeat until the test passes.
5. **Document the substitution** in the analysis so reviewers can verify
   the test was applied honestly.

**Why this matters**: Without this self-check, investigators naturally produce
statements that feel general but are actually specific patches in disguise.
The substitution test is an objective gate — it removes the investigator's
own blind spots about whether they discovered a principle or dressed up a patch.

**Example (passes the test)**:
```
Original: "When a dispatcher routes a task without checking the executor's
capability declaration, the task may land on an executor that cannot execute it."

Substituted: "When [a dispatcher] routes a task without checking [the
executor]'s [capability declaration], the task may land on [an executor] that
cannot execute it."

→ Holds. This is a principle: dispatchers must verify executor capability
before routing.
```

**Example (fails the test)**:
```
Original: "Dispatcher should always check [issue] format before posting."

Substituted: "[Dispatcher] should always check [specific format] before posting."

→ Does not hold as a universal principle. This is a patch for a specific format.
```

---

## Error Signature Classification

After the five questions, tag the root cause with one of these classification
codes. This enables cross-incident pattern analysis.

The eight codes are organized into **five dimensions** by error type:

### Cognitive Errors (Understanding)

| Code | Name | Definition | Example | Non-Agent Distinction |
|------|------|------------|---------|----------------------|
| `U-UND` | Understanding Deviation | Agent misunderstood the core intent of the request | "I asked for resume highlights, you gave me a system audit report" | **Agent-specific**: Unlike human misunderstanding, this often stems from instruction ambiguity or prompt structure, not cognitive limitation |
| `U-CTX` | Context Truncation | Agent lost context that was explicitly provided earlier | "I told you the scan scope was 'entire workspace' but you only scanned one issue" | **Agent-specific**: Context window limits and token management have no equivalent in traditional software RCA |

### Execution Errors (Process Adherence)

| Code | Name | Definition | Example | Non-Agent Distinction |
|------|------|------------|---------|----------------------|
| `E-FMT` | Output Format Violation | Agent ignored required output format, fields, or structure | "I asked for 3 lines — you gave 3000 words" | **Agent-specific**: LLM output is probabilistic; format adherence requires explicit structural constraints, unlike deterministic software |
| `E-SKP` | Process Skip | Agent omitted a required check or verification step | "You routed the task without checking if the squad can handle it" | **Shared pattern** with human process compliance, but the fix mechanism differs: agent fixes are instruction edits, not training |

### Interaction Errors (Multi-Agent)

| Code | Name | Definition | Example | Non-Agent Distinction |
|------|------|------------|---------|----------------------|
| `I-RED` | Interaction Redundancy | Agents looping, re-confirming, or @mentioning without information gain | "Three rounds of 'task completed' / 'no it's not' between two agents" | **Agent-specific**: Stateless re-triggering is a unique failure mode of event-driven agent systems; traditional distributed systems have circuit breakers |

### Delivery Errors (Output Completeness)

| Code | Name | Definition | Example | Non-Agent Distinction |
|------|------|------------|---------|----------------------|
| `D-INC` | Delivery Incomplete | Deliverable missing required components | "Claimed 18 files delivered but 0 found in shared workspace" | **Shared pattern** but the root cause is often environment isolation (agent-specific) rather than partial execution (traditional) |

### Process Errors (Routing & Workflow)

| Code | Name | Definition | Example | Non-Agent Distinction |
|------|------|------------|---------|----------------------|
| `P-RTE` | Routing Error | Task assigned to an entity without the required capability | "OSS contribution task assigned to inspection squad" | **Shared pattern** with any workflow system, but agent routing is instruction-based (no type system) making detection harder |
| `P-REP` | Repeated Execution | Agent re-executed an already-completed operation | "Sent 7 identical routing requests in 50 seconds" | **Agent-specific**: Stateless re-triggering produces duplicate work that a stateful workflow engine would prevent |

### Classification Rules

1. **Primary vs. Secondary**: An incident can carry one primary classification
   (the root cause layer) and optional secondary classifications (symptom layers).
2. **Choose closest to root cause**: If an incident matches multiple codes,
   select the one closest to the root cause. For example, `P-REP` (repeated
   execution) may be caused by `U-UND` (misunderstood the completion signal) —
   in that case, `U-UND` is primary and `P-REP` is secondary.
3. **Err toward the fixable layer**: If the root cause is ambiguous between two
   codes, prefer the one that points to the most actionable fix layer
   (instructions → process → design, in that order).

---

## RCA Output Template

Every RCA analysis must produce output in this standard format. The template
ensures consistency across incidents and enables cross-incident comparison.

### Required Output Fields (6 Items)

| # | Field | Description | Source |
|---|-------|-------------|--------|
| 1 | **Reflection Trigger** | What signal or event initiated this RCA | When to Use section |
| 2 | **Root Cause** | The five-question analysis output (Q1–Q5) | The Five Questions |
| 3 | **Classification** | Root cause category: one or more of `instructions盲区` \| `流程缺失` \| `检测断层` \| `设计缺陷` | Q3/Q4 analysis |
| 4 | **error_signature** | Machine-readable classification code (e.g., `U-UND`, `P-REP`) | Error Signature Classification |
| 5 | **Solution** | Concrete fix addressing the root cause (not the symptom) | Q5 principle → action |
| 6 | **Action Items** | Decomposed, trackable fixes with acceptance criteria | Action Item Decomposition |

### ⚠️ Critical Distinction: Classification ≠ error_signature

These are **two independent dimensions** and must not be merged:

| Dimension | Purpose | Values | Example |
|-----------|---------|--------|---------|
| **Classification** | Categorizes *where in the system* the root cause lies | `instructions盲区`, `流程缺失`, `检测断层`, `设计缺陷` | "This is a 检测断层 — the verification ran in the wrong environment" |
| **error_signature** | Tags *what type of error* manifested | `U-UND`, `U-CTX`, `E-FMT`, `E-SKP`, `I-RED`, `D-INC`, `P-RTE`, `P-REP` | "error_signature: D-INC (symptom), root: 设计缺陷" |

**Example**: An incident can be classified as `设计缺陷` (the root cause category
is a design-level gap) while carrying error_signature `D-INC` (the observable
symptom was incomplete delivery). The classification tells you *where to fix*;
the error_signature tells you *what broke*.

### Output Format Example

```
## RCA: [Short Incident Title]

**Reflection Trigger**: S8 (Closure Break) — Agent claimed delivery, consumer found nothing

**Root Cause (Q1–Q5)**:
Q1: [Timeline…]
Q2: Direct cause: files written to isolated filesystem, not shared workspace
Q3: Survived because: no cross-environment check after write
Q4: Undetected because: every feedback loop stayed at "didn't write" vs "wrote but unreachable"
Q5: Root pattern: self-verification in isolated environment is meaningless to target environment

**Classification**: 设计缺陷 (primary), 检测断层 (secondary)
**error_signature**: D-INC (primary), E-SKP (secondary)

**Solution**: Expose write domain to execution units; add cross-environment verification step

**Action Items**: [See Action Item Decomposition section below]
```

---

## Action Item Decomposition

The protocol's output is not a report — it is a set of **actionable items**
with clear ownership and acceptance criteria.

### Format

```
### [action] <Short title describing what to fix>

**Problem**: <One sentence linking to the root cause>

**Fix**: <Concrete change — edit instructions, add rule, create process>

**Acceptance Criteria**: <Observable, verifiable condition that proves the fix works>

**error_signature**: <Classification code>
```

### Rules for Action Items

1. **Fix the root, not the symptom**. If Q5 found a principle violation, the
   action must address the principle, not the specific instance.
2. **One action = one change**. If a fix requires editing three different
   agent instructions, that's three actions.
3. **Acceptance criteria must be observable**. "Agent quality improved" is not
   observable. "Agent no longer sends duplicate routing requests when
   re-triggered on a completed issue" is observable.
4. **Track with metadata**. Each action carries `error_signature` so
   cross-incident pattern analysis is possible.
5. **Closed-loop failure** (recurrence after closure): If the same class of
   error recurs after an action item was marked closed, do **not** simply
   reopen the old action. Instead, investigate *why the previous fix failed to
   prevent recurrence* — this is a new RCA trigger. The question is: "Why did
   the fix we deployed not prevent this?" — not "let's try the same fix again."
6. **Backlog vs. todo placement**: If the action item cannot be executed
   immediately (e.g., depends on a platform change, requires input from another
   squad), place it in the **backlog** with a clear unblock condition. Only
   place directly executable items in the **todo** list. A backlog item without
   an unblock condition is an abandoned item.

---

## Rule Design Meta-Principles

The protocol itself generated two meta-principles for designing agent rules.
Apply these when writing rules based on RCA findings.

### Principle A: Extract Principles, Not Patches

After identifying the root pattern (Q5), write the rule as a **principle**,
not a ban on the specific symptom. Deploy only after the proper noun
substitution test passes.

**Positive example** (principle):
```
"Quality judgments use absolute standards aligned to the requester's
expectations. Remove all qualifying words and re-read: if the statement
collapses, rewrite it."
→ Applies to any quality review by any agent in any domain.
```

**Negative example** (patch):
```
"Don't use relative language in OXY reviews."
→ Bound to a specific squad. The same error will recur under a different name.
```

### Principle B: One Global Rule > N Instance Patches

When the same class of error appears across multiple agents or contexts,
write **one global rule** in the shared instructions layer (e.g., CLAUDE.md),
not N separate rules in each agent's local instructions. Instance-level rules
fragment, drift, and create maintenance burden.

**Positive example** (global rule):
```
Add to CLAUDE.md: "Before routing any task, verify the target entity has
declared the required capability."
→ One rule, protects all routing agents.
```

**Negative example** (instance patches):
```
Add to dispatcher instructions: "Check [task] capability before routing."
Add to manager instructions: "Verify team can handle tasks before assigning."
Add to autopilot instructions: "Don't route to squad without checking skills."
→ Three rules, same intent, inconsistent wording, two will eventually drift.
```

**Why this matters**: Instance patches accumulate silently. Six months later,
no one remembers why three different agents have three slightly different
versions of "the routing check rule." When one drifts, the error returns —
and the investigator must rediscover the principle that was already found.

---

## Termination Conditions

Stop the analysis (do not continue to Q5) when any of these conditions are met:

| Condition | Signal | Action |
|-----------|--------|--------|
| **Transient** | Problem disappears on re-verification after restoring full context; was likely a temporary runtime glitch (context truncation, model sampling variation, network hiccup) | Mark analysis as `transient`, produce **no action items** |
| **Cancelled (active)** | The investigator or requester explicitly cancels the analysis (e.g., "never mind, this isn't worth an RCA") | Mark as `cancelled`, archive |
| **Cancelled (passive)** | The triggering issue, comment, or context has been deleted or withdrawn by an external actor | Mark as `cancelled`, archive |
| **Duplicate** | Another reflection issue already covers the same root pattern and its action items are still valid | Link to existing analysis, mark as `duplicate` |

### Transient Determination Flow

```
1. Suspect transient (e.g., context truncation indicators)
2. Re-run the same scenario with full context
3. ┌─ Problem disappears → Confirm no residual impact → Mark transient → STOP
4. └─ Problem persists → Not transient → Continue to Q3
```

**Key rule**: A transient termination produces **zero action items**. The most
common RCA anti-pattern is treating a transient as a defect and creating
"phantom fixes" that add instructions bloat without preventing anything.

---

## Self-Reflection Protocol

The protocol applies **recursively to the investigator itself**. When the
investigator makes errors, it runs the same five-question drill on its own
failures — no shortcuts, no softened constraints.

### When to Trigger Self-Reflection

Trigger self-reflection RCA when:

- You have been corrected **≥ 2 times** on the same topic within a short window
- Your output was **rejected or pulled back** from review
- The **same class of error** appears in your work within days of a previous fix
- You notice yourself **patching symptoms** (quick local fixes) instead of
  investigating the pattern

### Self-Reflection Process

1. **Run the standard five questions** on yourself — Q1 through Q5, all
   constraints enforced. The subject is now your own output and decision process.
2. **Acknowledge the outcome** honestly: if you find `U-UND` (understanding
   deviation) in your own work, own it. The protocol's credibility depends on
   equal application to self and others.
3. **Produce action items for yourself**: what instruction or process change
   will prevent this class of error in your future work?
4. **Publish the self-RCA** in the relevant issue thread so others can see
   that the investigator applies the same standard to itself.

### Self-Reflection vs. Correction Response

Self-reflection is a **post-mortem analysis** — you run it after the fact,
using the full five-question drill. It answers "why did I fail?"

**Correction Response** (next section) is a **real-time protocol** — you
execute it *while* being corrected, before making changes. It answers
"how do I respond to this correction without making things worse?"

The two are related: a self-reflection RCA may *produce* a Correction Response
Protocol rule as one of its action items.

---

## Correction Response Protocol

When you receive a correction (from a user, another agent, or a reviewer),
execute these four steps **before** making any changes. This protocol prevents
the most common correction failure mode: patching only the specific point
raised while drifting further from the original intent.

### The Four Steps

| Step | Action | Output |
|------|--------|--------|
| **1. Align** | Restate your understanding of the core need | "I understand your core need is X. Is that correct?" |
| **2. Confirm scope** | List exactly what you plan to change | "Based on this, I need to change: A, B, C. Does this cover everything?" |
| **3. Execute** | Make changes only after explicit confirmation | Apply edits |
| **4. Verify** | Check alignment against original request | "Is the result aligned with the original request, not just the last correction?" |

### The 1-Minute Principle

After receiving a correction, spend **at least 1 minute** re-reading the
original request before responding. The instinct is to reply in 30 seconds
fixing only the surface issue — this is the root mechanism behind high-frequency
correction cascades (S7b).

> **1 minute vs. 30 seconds**: 30 seconds is enough to patch the symptom.
> 1 minute is enough to re-anchor to the original intent. The extra 30 seconds
> prevents the next 3 correction rounds.

### Relationship to "产出前三问" (Pre-Output Check)

The Correction Response Protocol and the pre-output three-question check
(需求对齐 / 假设核实 / 纠正闭环) serve different roles:

| Protocol | Trigger | Direction |
|----------|---------|-----------|
| **Pre-output three questions** | Before every reply | Proactive — prevents errors from shipping |
| **Correction Response Protocol** | After receiving a correction | Reactive — prevents correction cascades |

They are complementary, not redundant. The pre-output check is a **gate**
(prevent); the Correction Response Protocol is a **guardrail** (contain).

---

## Common Mistakes & Anti-Patterns

The following is a non-exhaustive list of common errors when applying the
five-question protocol. Each entry shows the **incorrect** pattern, the
**correct** approach, and **why** the distinction matters.

### Anti-Pattern 1: Causal Language in Q1

| | Incorrect ❌ | Correct ✅ |
|---|-------------|-----------|
| **Q1 output** | "Alice kept claiming files were written **because** she was checking in the wrong environment, **causing** Bob to waste time." | "15:21 — Alice claimed '6 files written'. 15:22 — Bob found 0 files. 15:23 — Alice re-claimed '18 files written' with local verification." |
| **Why it matters** | Causal words in Q1 pre-load the conclusion. The investigator has already decided "Alice was wrong" before Q2 even begins. Q1 is pure evidence-gathering — causality belongs in Q2–Q5. | |

**Rule**: If a Q1 sentence contains "because", "due to", "caused by", "resulted in",
"led to" — **reject it immediately**. The Q1 gate is absolute.

### Anti-Pattern 2: Vague Direct Cause

| | Incorrect ❌ | Correct ✅ |
|---|-------------|-----------|
| **Q2 output** | "Direct cause: communication breakdown between agents." | "Direct cause: Dispatcher instructions lack 'check if this task was already routed' step (no dedup guard). Each trigger event causes dispatcher to re-evaluate from scratch." |
| **Why it matters** | "Communication breakdown" is unfalsifiable and unactionable. You cannot edit instructions to fix "communication." The correct version names a specific missing instruction that can be added and verified. | |

**Rule**: If you could give the Q2 answer to an engineer who has never seen the
system and they couldn't find the exact line to edit — it's too vague.

### Anti-Pattern 3: Symptom Patch Disguised as Principle

| | Incorrect ❌ | Correct ✅ |
|---|-------------|-----------|
| **Q5 output** | "Dispatcher should always check [issue] format before posting." | "When a rule is expressed as a template or example within a system's instructions, the system treats it as specific to that template scenario, not as a universal behavior. Universal rules must be declared independently of any template." |
| **Substitution test** | "[Dispatcher] should always check [specific format] before posting." → Does **not** hold as a universal. | "When [a rule] is expressed as [a template] within [instructions], [the system] treats it as [template-scoped]. [Universal rules] must be declared independently." → **Holds**. |
| **Why it matters** | The incorrect version is a patch: it fixes one format in one agent. The correct version is a principle: it prevents the entire class of "template-bound rules that don't generalize" — which applies to every agent, every template, every platform. | |

**Rule**: If the Q5 statement still contains a proper noun (agent name, format
name, squad name, file path), the substitution test has not been honestly applied.

### Anti-Pattern 4: Merging Classification and Error Signature

| | Incorrect ❌ | Correct ✅ |
|---|-------------|-----------|
| **Output** | "Classification: D-INC" | "Classification: 设计缺陷. error_signature: D-INC (primary), E-SKP (secondary)." |
| **Why it matters** | `D-INC` is a symptom tag — it tells you *what* (incomplete delivery). `设计缺陷` tells you *where* (the design layer). Without the classification dimension, you know what broke but not where to fix it. An action item addressing `D-INC` at the agent level would miss the design-level root cause. | |

**Rule**: If your RCA output has only one classification field, you're missing
a dimension. Always produce both: the root cause category **and** the error
signature code.

### Anti-Pattern 5: Action Items Without Acceptance Criteria

| | Incorrect ❌ | Correct ✅ |
|---|-------------|-----------|
| **Action item** | "Fix: Improve agent communication quality." | "[action] Add pre-routing dedup check. **Acceptance Criteria**: Given a task routed within 24h, re-triggering produces ≤ 1 routing action." |
| **Why it matters** | Without observable acceptance criteria, you can never verify the fix worked. Three months later, no one knows whether "improve communication quality" was achieved. The correct version gives a testable, binary condition. | |

**Rule**: If a reviewer cannot look at agent behavior and answer "did this fix
work? yes/no" in under 10 seconds, the acceptance criteria are not specific enough.

---

## Worked Example

This example walks through a complete five-question RCA of a real production
incident. It was chosen as the primary worked example because it is the
cleanest case: single root cause, clear error signature pair, and action items
that generalize to any event-driven agent system.

> **Source**: Case: Duplicate Routing Request Burst (originally traced from production issue #412) —
> Dispatcher agent sent 7 identical routing requests in 50 seconds.

### Trigger
Signal **S3** (Circular mentions) + **S6** (Timing anomaly): Dispatcher and
Router @mention each other repeatedly, and 7 nearly-identical routing requests
appear within 50 seconds. [→ When to Use: Signal Detection Triggers]

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

## Integration with Existing Systems

This protocol is designed to be **platform-agnostic**:

- **No specific API calls required.** The five questions work with any
  observable data: comments, logs, timestamps, status changes.
- **No specific agent framework assumed.** Works with any LLM-based agent
  system — the protocol is about the methodology, not the runtime.
- **Metadata-based tracking is recommended but optional.** Classifying errors
  with codes like `U-UND` / `E-SKP` enables cross-incident analysis, but the
  protocol works without it.
- **The protocol can be embedded in an agent's instructions** (as a quality
  assurance agent's core methodology) or **used by human operators** reviewing
  agent performance.

### Embedding in Agent Instructions

Two integration levels are provided. Choose based on how deeply you want to
embed the protocol.

#### Minimal Embed (~15 lines)

For agents that need the core methodology without the full classification
system. Suitable for general-purpose QA agents.

```
## Root Cause Analysis (Five-Question Protocol)

When analyzing an error, follow these five questions in order.
Never skip a question or soften a constraint.

Q1: Facts — Timeline only. No causal words ("because", "due to").
    If you find yourself writing "because", rewrite.
Q2: Direct cause — Pinpoint the specific instruction / operation / decision.
    Cannot be vague ("communication issue"). Must be concrete.
    If suspected context truncation: re-run and if problem disappears → mark
    transient and STOP (no action items).
Q3: Why it survived — Which exact process step failed to catch this?
    Name the gap with its location.
Q4: Why it wasn't detected — Map the full detection chain, mark the break
    point, report the delay.
Q5: Root pattern — Express as a universal principle. Replace all proper nouns
    with placeholders. If the statement still holds → it's a principle.
    If it collapses → it's a patch; go back and rewrite.

Output: Root cause + action items with observable acceptance criteria.
Terminate on: transient, cancelled (active/passive), or duplicate.
```

**Self-test**: Replace all platform-specific terms in this template with
generic placeholders. Does it still make sense for a LangChain agent? For an
AutoGPT agent? For a human operator? If yes → the template is sufficiently
platform-agnostic.

#### Full Embed (~50 lines)

For dedicated quality assurance / inspection agents. Includes the error
signature classification system and action item format.

```
## Root Cause Analysis Protocol (Full)

When analyzing an agent error or quality incident, follow the five-question
drill from deep-trace.

### Questions
Q1: Facts — timeline only, no causal words.
Q2: Direct cause — pinpoint the specific instruction/operation/decision.
    Transient check: re-run → problem disappears → mark transient, STOP.
Q3: Why it survived — which process step failed, with exact location.
Q4: Why it wasn't detected — full detection chain table, mark break, report delay.
Q5: Root pattern — principle + proper noun substitution test.

### Output Format
1. Reflection trigger (which signal)
2. Root cause (Q1–Q5)
3. Classification: instructions盲区 | 流程缺失 | 检测断层 | 设计缺陷
4. error_signature: U-UND | U-CTX | E-FMT | E-SKP | I-RED | D-INC | P-RTE | P-REP
5. Solution (addresses root, not symptom)
6. Action items: [action] title + problem + fix + acceptance criteria + error_signature

### Classification Codes
U-UND: Understanding deviation    E-FMT: Format violation      I-RED: Interaction redundancy
U-CTX: Context truncation         E-SKP: Process skip          D-INC: Delivery incomplete
                                                            P-RTE: Routing error    P-REP: Repeated execution

### Termination
Stop at Q2 if: transient (re-run → disappears), cancelled (active/passive), duplicate.

### Meta-Principles for Rules
- Extract principles, not patches → proper noun substitution test before deployment
- One global rule > N instance patches → write to shared instructions layer
```

### Integration Verification Checklist

After embedding, verify with these 3 checks:

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | **Trigger**: Does the agent recognize when to start RCA? | Give it a scenario with S3 or S7b signals — does it initiate the five-question drill unprompted? |
| 2 | **Classification**: Does error_signature get assigned correctly? | Feed a known incident (e.g., the worked example) — does the agent produce `I-RED` / `P-REP`? |
| 3 | **Action format**: Are action items correctly formatted? | Check output for `[action]` prefix + `**Problem**` / `**Fix**` / `**Acceptance Criteria**` / `**error_signature**` fields |

---

## Version & Provenance

### Source Material

This skill is based on:

- **CLAUDE.md 压缩版** (lines 48–67): The compressed five-question protocol from
  the agent instructions. This is the primary and only available source for the
  five-question framework.
- **6 production cases**: Case: Duplicate Routing Request Burst (traced from #412), Case: Cross-Agent File Delivery Failure (#414), Case: High-Frequency Correction Spiral (#571), Case: Blocked-Issue Timeout Cluster (#574), Case: Quality Standard Relativization (#177), Case: Template-Bound Rule Gap (#627)
  — real RCA analyses executed in a multi-agent system. Each case has been
  cross-validated against the original issue threads.

### Known Gaps

| Gap | Impact | Mitigation |
|-----|--------|------------|
| **完整版 3405 char 五问协议未纳入** | The original CLAUDE.md references a "complete version" (3405 characters) that has not been located on disk. This skill is based solely on the compressed version. | If the complete version is recovered, diff it against this skill to identify and incorporate missing constraints. |
| **patrol-signal-detection skill 未创建** | Signal definitions (S1–S9) are inlined in this skill but lack the full detection logic planned for the standalone patrol skill. | The inlined definitions are sufficient for RCA triggering. Create the patrol skill separately for real-time monitoring. |

### Version Dependencies

| Dependency | Status | Notes |
|-----------|--------|-------|
| CLAUDE.md (compressed 5-questions) | ✅ Available | Lines 48–67 |
| Complete 3405 char version | ❌ Not located | Referenced but not found on disk |
| Agent runtime | Any | Protocol is platform-agnostic |
| Error signature codes | ✅ Self-contained | Defined in this skill |

### Companion Skills (Planned)

| Skill | Relationship |
|-------|-------------|
| `agent-pregating-check` | Pre-output three-question self-check — prevents errors before they ship |
| `agent-review-standards` | Daily quality review (four dimensions + three iron rules) — handles routine quality |
| `patrol-signal-detection` | Real-time S1–S9 signal monitoring — detects when RCA should be triggered |

---

## References

- `references/rca-examples.md` — Annotated transcripts of three production RCA
  analyses (Case: Duplicate Routing Request Burst, Case: High-Frequency Correction Spiral, Case: Quality Standard Relativization) with two additional case summaries
  (Case: Cross-Agent File Delivery Failure, Case: Blocked-Issue Timeout Cluster). Each example shows the full five-question output and
  identifies what a shallower approach would have missed.

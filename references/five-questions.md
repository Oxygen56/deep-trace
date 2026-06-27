# The Five Questions — Full Methodology

> **Parent skill**: [deep-trace SKILL.md](../SKILL.md)
> 
> This file contains the complete five-question methodology with all constraints, 
> examples, and termination conditions. Read the parent SKILL.md first for a 
> quick overview.

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



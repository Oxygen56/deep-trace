# Action Items & Rule Design Principles

> **Parent skill**: [deep-trace SKILL.md](../SKILL.md)

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



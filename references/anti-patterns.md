# Common Mistakes & Anti-Patterns

> **Parent skill**: [deep-trace SKILL.md](../SKILL.md)
>
> Non-exhaustive list of common errors when applying the five-question protocol.
> Each entry shows the incorrect pattern, the correct approach, and why it matters.

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



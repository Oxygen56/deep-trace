# Error Signatures & Output Template

> **Parent skill**: [deep-trace SKILL.md](../SKILL.md)

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



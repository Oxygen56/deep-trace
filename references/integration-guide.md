# Integration Guide

> **Parent skill**: [deep-trace SKILL.md](../SKILL.md)
>
> How to embed the five-question protocol into agent instructions. Two integration 
> levels are provided — choose based on how deeply you want to embed the protocol.

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



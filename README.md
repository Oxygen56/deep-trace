# deep-trace

**AI Agent Root Cause Analysis Protocol — Five-Question Drill Adapted from Toyota's 5 Whys**

When an AI agent makes a mistake, the instinct is to fix the symptom — add a one-line rule, tell it "don't do that." These patches accumulate, inflate instructions, and the same class of error recurs under different names.

**deep-trace breaks that cycle.** It's a systematic five-question drill that forces the investigator to trace every error back to its root cause in the instruction/process/design layer — not the symptom layer.

---

## Before vs. After: Why deep-trace Exists

A real production incident. **Dispatcher agent sent 7 identical routing requests in 50 seconds** to a router that had already responded.

### 🔴 Without deep-trace (Conventional Approach)

| Step | Action | Outcome |
|------|--------|---------|
| **See symptom** | "The dispatcher keeps repeating itself" | Filed as "agent misbehavior" |
| **Apply fix** | Add rule: "Don't send duplicate routing requests" | Rule deployed to dispatcher instructions |
| **Verify** | Wait. No duplicate burst in the next hour. | ✅ Marked resolved |
| **Reality** | Two weeks later, a different agent re-executes a completed deployment 5 times. Same root cause, different name. | 🔴 Problem returned |

**Why the fix failed**: "Don't send duplicate routing requests" is a symptom patch. It silences one symptom in one agent. The root cause — stateless agents re-evaluating from scratch on every trigger event — is untouched. It surfaces elsewhere, wearing a different mask.

### 🟢 With deep-trace (Five-Question Root Cause Drill)

| Question | Finding |
|----------|---------|
| **Q1: Facts** | Timeline: 7 identical routing requests in 50 seconds, router already responded after #1 |
| **Q2: Direct Cause** | Dispatcher instructions lacked "before routing, check if this task was already routed." Each trigger event causes dispatcher to restart from scratch — no cross-run state awareness. |
| **Q3: Why It Survived** | Three specific process gaps: (1) no dedup check step in routing workflow, (2) @mention rules covered silence-prevention but not re-trigger-prevention, (3) no rule "if executor said 'done', don't @mention them again" |
| **Q4: Detection Chain** | 7-request burst went undetected for **12.75 hours**. Detection existed (humans review) but had no real-time component. Every feedback loop treated each request as independent. |
| **Q5: Root Pattern** | **Principle**: When a stateless agent is re-triggered by events on the same context and has no cross-run state awareness, it re-evaluates from scratch and may re-execute already-completed actions. Instructions must include a mandatory "check existing results before acting" step. |

**Substitution test**: Replace "stateless agent" → "[stateless processing unit]", "re-triggered" → "[re-invoked]" — still holds. This is a **universal principle**, not a dispatcher-specific patch.

**Three action items deployed**:
1. **Pre-routing dedup check** — check before routing, not after
2. **@mention guard for completed agents** — use issue links, not agent mentions
3. **Burst detection monitoring** (backlog) — alert on ≥3 near-identical comments in <60s

**Result**: The same class of error **never recurred** — across any agent, any trigger type, any task domain. Because the fix addressed the pattern, not the symptom.

### The Difference, In One Sentence

| Without deep-trace | With deep-trace |
|---|---|
| "The dispatcher shouldn't repeat itself." | "Stateless agents need mandatory pre-action state checks." |
| Fixes **one agent** for **one symptom**. | Fixes **every agent** for **an entire class of failures**. |
| Problem returns in two weeks under a different name. | Problem doesn't return. |

---

## Scope: What deep-trace Is (and Isn't)

deep-trace is **post-mortem Root Cause Analysis** — you run it after an incident occurs. It's one component of a broader quality assurance system:

```
                ┌──────────────────────────┐
                │  patrol-signal-detection │  ← "When should we investigate?"
                │  (real-time S1–S9 signal │
                │   monitoring)            │
                └────────────┬─────────────┘
                             │ triggers
                             ▼
                ┌──────────────────────────┐
                │  deep-trace (this repo)  │  ← "What's the root cause?"
                │  (post-mortem five-      │
                │   question RCA drill)    │
                └────────────┬─────────────┘
                             │ produces action items
                             ▼
                ┌──────────────────────────┐
                │  agent-pregating-check   │  ← "How do we prevent recurrence?"
                │  (pre-output three-      │
                │   question self-check)   │
                └──────────────────────────┘
```

| Skill | Role | When |
|-------|------|------|
| **deep-trace** (this repo) | Post-mortem RCA — five-question drill | After an incident |
| agent-pregating-check (planned) | Pre-output gate — "产出前三问" | Before every reply |
| agent-review-standards (planned) | Daily quality review — 4D standards | Routine quality checks |
| patrol-signal-detection (planned) | Real-time S1–S9 signal monitoring | Continuously |

---

## What Makes deep-trace Different

There are many "5 Whys" tutorials. deep-trace is not a general methodology explainer — it's an **AI agent-specific RCA protocol** built from 60+ production incidents.

| Generic 5 Whys Tutorial | deep-trace |
|---|---|
| "Ask why five times" | Each question has a **hard constraint** with gate checks (Q1: no causal words; Q2: must be a specific line/operation; Q5: substitution test) |
| No agent-specific concepts | **Transient termination** (context truncation → re-run → disappears → stop), **8 error signature codes** (U-UND, U-CTX, E-FMT, E-SKP, I-RED, D-INC, P-RTE, P-REP) covering LLM-agent-specific failure modes |
| Ends with a report | Ends with **actionable items**: `[action]` prefix + acceptance criteria + error_signature tags for cross-incident tracking |
| Rules are added ad-hoc | **Meta-principles**: extract principles, not patches; one global rule > N instance patches |
| No self-application | **Recursive self-reflection**: the protocol requires the investigator to apply the same five questions to its own failures |

---

## Quick Start

### Install as a Claude Code Skill

```bash
# Copy to your skills directory
cp -r deep-trace ~/.claude/skills/deep-trace/
```

The skill auto-activates when you invoke `/deep-trace` or when agent error signals are detected.

### Embed in Agent Instructions (Minimal, ~15 lines)

```
## Root Cause Analysis (Five-Question Protocol)

When analyzing an error, follow these five questions in order:

Q1: Facts — Timeline only. No causal words ("because", "due to").
Q2: Direct cause — Pinpoint the specific instruction/operation/decision.
    If suspected context truncation: re-run → disappears → mark transient, STOP.
Q3: Why it survived — Which exact process step failed to catch this?
Q4: Why it wasn't detected — Map full detection chain, mark break point, report delay.
Q5: Root pattern — Express as universal principle. Replace proper nouns with
    placeholders. Still holds → principle. Collapses → patch; rewrite.

Output: Root cause + action items with observable acceptance criteria.
```

### Embed in Agent Instructions (Full, ~50 lines)

See [SKILL.md](./SKILL.md#integration-with-existing-systems) for the complete embedding template including error signature classification codes.

---

## Repository Structure

```
deep-trace/
├── README.md                    # You are here
├── SKILL.md                     # Complete protocol (~980 lines)
│   ├── Scope Declaration
│   ├── When to Use (9 signal types + non-signal triggers)
│   ├── The Five Questions (with constraints, gates, examples)
│   ├── Error Signature Classification (8 codes × 5 dimensions)
│   ├── RCA Output Template (6 required fields)
│   ├── Action Item Decomposition (format + 6 rules)
│   ├── Rule Design Meta-Principles (2 principles with substitution test)
│   ├── Termination Conditions (4 types)
│   ├── Self-Reflection Protocol
│   ├── Correction Response Protocol (4-step)
│   ├── Common Mistakes & Anti-Patterns (5 patterns)
│   ├── Worked Example (OXY-412: full five-question walkthrough)
│   ├── Integration Guide (minimal + full embed templates)
│   └── Version & Provenance
└── references/
    └── rca-examples.md           # 3 full transcripts + 2 appendix cases
```

---

## Production Track Record

Developed and battle-tested over 60+ agent collaboration scenarios in a multi-agent system. Key results:

- **Eliminated duplicate routing bursts** (OXY-412): 7 identical requests in 50s → 0 recurrences after deploy
- **Ended high-frequency correction cascades** (OXY-571): 4 corrections in 40 min → 0 S7b recurrence
- **Extracted reusable design principle** (OXY-177): Template-bound rules don't generalize — prevented an entire class of future errors
- **Reduced instruction patch inflation**: Rules added via RCA are principles, not one-off symptom bans

---

## License

MIT

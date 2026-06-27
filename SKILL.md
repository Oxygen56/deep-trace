---
name: deep-trace
description: "Agent Root Cause Analysis Protocol — A systematic five-question methodology adapted from Toyota's 5 Whys for diagnosing failures in AI agent systems. Covers trigger scenarios, the five-question framework, error signature classification, action decomposition, rule design meta-principles, and self-reflection."
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

Battle-tested over 60+ agent collaboration scenarios in production.

---

## When to Use

### Signal Detection Triggers

Run the protocol when any of these signals appear:

| Code | Signal | One-line definition |
|------|--------|---------------------|
| **S1** | Correction | User or downstream agent explicitly negates output |
| **S2** | Repeated explanation | Same point explained ≥ 2 times on the same issue |
| **S3** | Circular mentions | Two agents @mention each other ≥ 3 rounds without progress |
| **S4** | Output rejected | Delivery explicitly rejected as unusable |
| **S5** | Blocked timeout | Task blocked > 48h with no resolution |
| **S6** | Timing anomaly | Unusual delay or burst pattern in agent activity |
| **S7** | Repeated correction | ≥ 3 corrections from the same person on the same topic within 24h |
| **S8** | Closure break | Closed-loop action chain is broken |
| **S9** | Flow stagnation | Task stays in the same stage beyond expected duration |

Also trigger on: user explicitly requests RCA ("why did this happen?"), or agent
self-triggers on noticing a pattern in its own errors.

### When NOT to Use

- Single isolated typo or formatting glitch, fixed immediately
- Error already covered by an existing RCA with active action items
- Known transient (one-time network timeout, resolved on retry)
- Better handled by routine review (accuracy/completeness checks) rather than deep RCA

---

## Quick Reference: The Five Questions

| Q | Name | Question | Core Constraint |
|---|------|----------|-----------------|
| **Q1** | Facts (事实) | What happened? | **No causal words.** Pure timeline. If "because"/"due to" appears, rewrite. |
| **Q2** | Direct Cause (直接原因) | What specifically caused this? | **Pinpoint a concrete element** — a line in instructions, a tool call, a missing check. Cannot be vague. |
| **Q3** | Why It Survived (为什么存活) | Which process step failed to catch this? | **Name the exact gap with its location.** "Process incomplete" is not specific enough. |
| **Q4** | Why It Wasn't Detected (为什么没发现) | Where did the detection chain break? | **Map the full detection chain**, mark the break point, report the delay. |
| **Q5** | Root Pattern (根本模式) | What's the universal principle? | **Proper noun substitution test**: replace all names with placeholders. Still holds → principle. Collapses → patch. |

---

## Reference Index

All detailed content lives in `references/`. Read the files relevant to your task:

| Reference File | Content | Use When |
|---------------|---------|----------|
| [`references/five-questions.md`](references/five-questions.md) | Full Q1-Q5 methodology with constraints, examples, good/bad samples, and termination conditions | You need to actually run an RCA |
| [`references/classification.md`](references/classification.md) | 8 error signature codes (`U-UND` through `P-REP`), output template, classification vs error_signature distinction | You need to tag or categorize a finding |
| [`references/rules-and-actions.md`](references/rules-and-actions.md) | Action item decomposition format, acceptance criteria rules, and rule design meta-principles (extract principles not patches; one global rule > N instance patches) | You're writing fixes from RCA findings |
| [`references/anti-patterns.md`](references/anti-patterns.md) | 5 common mistakes: causal language in Q1, vague direct cause, symptom patches disguised as principles, merging classification and error_signature, action items without acceptance criteria | You want to avoid common pitfalls |
| [`references/worked-example.md`](references/worked-example.md) | Complete walkthrough of a real incident (7 duplicate routing requests in 50 seconds), with all Q1-Q5 outputs and analysis | You're learning the protocol or want a template |
| [`references/self-protocols.md`](references/self-protocols.md) | Self-Reflection Protocol (running RCA on yourself) and Correction Response Protocol (how to respond to corrections without making things worse) | You've been corrected or are correcting yourself |
| [`references/integration-guide.md`](references/integration-guide.md) | Minimal (~15 line) and Full (~50 line) embed templates for embedding the protocol in agent instructions, plus verification checklist | You're adding RCA capability to an agent |
| [`references/rca-examples.md`](references/rca-examples.md) | Annotated transcripts of 3 production RCA analyses with 2 additional case summaries | You want to see real-world applications |

---

## Version & Provenance

Based on CLAUDE.md compressed five-question protocol (lines 48-67) and 6 production
RCA cases. Platform-agnostic — works with any LLM-based agent system. Companion
skills planned: `agent-pregating-check` (pre-output self-check), `agent-review-standards`
(daily quality review), `patrol-signal-detection` (real-time S1-S9 monitoring).

Known gaps: the original 3405-character complete five-question protocol has not
been located on disk. If recovered, diff it against `references/five-questions.md`.

---
name: ud-jev-decision-workflow
license: MIT
description: >
  Architectural decision workflow for choosing among deterministic code,
  Jev/System One bounded semantic judgments, reasoning LLMs, and human review.
  Provides dual-provider routing for Jev across native TypeSafe and OpenRouter,
  enforces confidence vs. consequence governance, and preserves application-owned
  workflow authority. Complements and sits one layer above the official typesafe-ai skill.
---

# Decision Workflow with Jev & Deterministic Code

> **Core philosophy:** Code owns the workflow. AI supplies bounded semantic judgment where deterministic code is insufficient.

The application or workflow engine (e.g., backend service, worker, n8n, event pipeline) must retain ownership of state, permissions, deterministic business rules, thresholds, execution, irreversible actions, audit trails, and escalation policy. AI models should never act as the autonomous central brain of a mission-critical workflow unless explicitly designed with verified boundaries.

---

## Relationship with the Official `typesafe-ai` Skill

This skill sits **one architectural layer above** the official `typesafe-ai` skill:

```text
typesafe-ai (Official)
= TypeSafe/Jev semantics + native TypeSafe usage & primitives
  (Authoritative for Choice, Noul, Score, native SDK contracts, live docs)

ud-jev-decision-workflow (This Skill)
= Architecture + provider choice + escalation policy + governance
  (Authoritative for execution mode selection, transport abstraction, uncertainty vs consequence)
```

- **Do not modify or fork** the official `typesafe-ai` skill.
- The official `typesafe-ai` skill remains the **sole authoritative reference** for current TypeSafe/System One primitives, request schemas, and native SDK methods.
- Both skills are designed to be installed together and complement each other.

---

## The Decision Hierarchy

When designing any workflow step, evaluate options from most deterministic to most open-ended:

```text
Task / Incoming Event
  ↓
Can deterministic code or rules solve it reliably?
  ├── Yes → DETERMINISTIC CODE / RULES (Regex, lookup, schema validation, calculations)
  └── No
        ↓
Is this a bounded, narrow semantic judgment?
  ├── Yes → JEV / SYSTEM ONE
  │           ↓
  │     Evaluate Confidence + Consequence
  │           ├── High confidence + low consequence → AUTOMATED ACTION
  │           ├── Ambiguous / low confidence → REASONING LLM
  │           └── High consequence → HUMAN REVIEW
  │
  └── No
        ↓
Does it require synthesis, generation, planning, multi-step analysis, or open-ended prose?
        → REASONING / FRONTIER LLM
```

---

## The 4 Execution Modes

| Mode | When to Use | Typical Operations |
| :--- | :--- | :--- |
| **A. Deterministic Code** | Rules, exact lookups, math, static logic can solve the task reliably. | Regex matching, schema validation, database lookup, permission check, numeric calculation, blacklist/whitelist, status transitions. |
| **B. Jev / System One** | Bounded, atomic semantic judgment where ordinary code lacks common sense. | Category classification (`Choice`), boolean probability (`Noul`), rubric rating (`Score`), intent routing, relevance triage. |
| **C. Reasoning LLM** | Deep synthesis, multi-document aggregation, open-ended reasoning, or generation. | Investigative synthesis, dispute resolution reasoning, policy drafting, open-ended explanation, out-of-distribution analysis. |
| **D. Human Review** | High consequence, irreversible actions, regulatory mandates, or persistent ambiguity. | Account suspension, payment blocking, security escalation, medical/legal impact, compliance exceptions. |

See detailed breakdown in [decision-architecture.md](references/decision-architecture.md).

---

## Confidence Semantics & Governance

Two immutable rules govern AI decision-making:

```text
typed output != truth
confidence != permission to act
```

1. **Syntactic validity is not semantic truth:** A valid `Choice` or `Score` guarantees only that the output format matched the contract. It does not prove the factuality or correctness of the decision.
2. **Consequence gates automation:** A model confidence score of `0.98` may authorize routing a harmless low-priority email, but it must **never** solely authorize freezing an account, deleting data, or rejecting a high-value transaction.
3. **Thresholds are not universal constants:** Never hardcode generic thresholds like `0.80` or `0.95` without calibrating against representative historical data and factoring in false-positive versus false-negative costs.

See detailed rules in [governance-and-escalation.md](references/governance-and-escalation.md).

---

## Provider Routing: Native TypeSafe vs. OpenRouter

Jev can be consumed via either **Native TypeSafe** or **OpenRouter**. Keep decision semantics isolated from provider transport:

```text
Business Decision Specification (Questions & State)
                     ↓
          Provider Adapter Layer
          ├── Native TypeSafe API  (POST /v1/systemone)
          └── OpenRouter API       (POST /api/alpha/decisions)
```

### Provider Selection Policy

```text
1. If the user or project config explicitly specifies a provider:
   → Use that provider.

2. Else if TYPESAFE_API_KEY is present:
   → Use Native TypeSafe (endpoint: api.typesafe.ai/v1/systemone, model: jev-latest).

3. Else if OPENROUTER_API_KEY is present:
   → Use OpenRouter (endpoint: openrouter.ai/api/alpha/decisions, model: typesafe/jev-latest).

4. Else:
   → Halt and instruct user to configure either TYPESAFE_API_KEY or OPENROUTER_API_KEY.
```

> [!WARNING]
> **Strict Transport Isolation:** Never simply point the TypeSafe Native SDK at the OpenRouter base URL. OpenRouter uses distinct transport endpoints, model naming conventions (`typesafe/jev-latest`), and request envelopes. Keep provider adapters decoupled.

See complete implementation details in [provider-routing.md](references/provider-routing.md).

---

## Reusable Architecture Pattern

```text
[ Incoming Request / State ]
            ↓
1. Deterministic Preprocessing (Sanitize, extract metadata, normalize)
            ↓
2. Deterministic Gate (Rules, blocklists, schema checks, exact cache hit)
            ↓
3. Jev Atomic Judgments (Parallel Choice / Noul / Score evaluations)
            ↓
4. Policy Evaluation (Uncertainty + Consequence Matrix)
      ├── Sufficient evidence + low consequence  → 5A. Deterministic Workflow Execution
      ├── Ambiguous evidence / high uncertainty   → 5B. Reasoning LLM Escalation
      └── Material risk / high consequence       → 5C. Human Queue & Verification
            ↓
6. Audit Logging & Evidence Storage (State, questions, answers, probabilities, reasoning)
```

---

## References & Examples

- **Architectural Reference:** [references/decision-architecture.md](references/decision-architecture.md) — 4-tier decision hierarchy, workflow boundary definitions.
- **Provider Routing Reference:** [references/provider-routing.md](references/provider-routing.md) — Native vs. OpenRouter adapter patterns, configuration, and transport contracts.
- **Governance Reference:** [references/governance-and-escalation.md](references/governance-and-escalation.md) — Uncertainty vs consequence matrix, threshold calibration, and 10 principles.
- **Design Patterns & Anti-Patterns:** [references/patterns.md](references/patterns.md) — Speculative fan-out, verification cascades, and anti-patterns to avoid.
- **Example 1: IT Support Ticket Routing:** [examples/support-ticket-routing.md](examples/support-ticket-routing.md) — Ticket triage with n8n/Python, deterministic filtering, and atomic Jev questions.
- **Example 2: Financial Fraud Triage:** [examples/fraud-triage.md](examples/fraud-triage.md) — Suspicious event screening with strict consequence guardrails and human escalation.

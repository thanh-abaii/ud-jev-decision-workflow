# Governance, Confidence Semantics, and Escalation

## 1. Core Confidence Semantics

When integrating AI judgments into operational pipelines, developers often make two fatal assumptions:
1. *If the model returns a schema-valid response, the judgment is factually correct.*
2. *If the model's confidence is above 0.90, the workflow is safe to automate.*

This skill enforces two non-negotiable axioms:

```text
typed output != truth
confidence != permission to act
```

### Typed Output vs. Semantic Truth
A schema-valid `Choice`, `Noul`, or `Score` guarantees only syntactic conformance—that the output matches your requested data types. It does **not** guarantee:
- That the source context contained complete facts.
- That the model did not misinterpret nuanced domain jargon.
- That the underlying reality matches the model's probabilistic assessment.

### Confidence vs. Permission to Act
A confidence score (e.g., `0.95`) reflects the concentration of probability across the output distribution under the model's internal priors. It does **not** equal a business authorization to trigger side effects. 

Automation permission must be computed as a function of **both Uncertainty and Consequence**:

$$\text{Permission to Automate} = f(\text{Uncertainty}, \text{Consequence}, \text{Reversibility})$$

---

## 2. The 10 Governance Principles

1. **Retain Business Authority in Application Code:** Business logic, thresholds, permissions, and policy rules must reside in code, not in prompts or model weights.
2. **Decouple Judgment from Execution:** The AI model produces a structured observation or classification. The workflow engine determines what action to execute based on that observation.
3. **Decouple Confidence from Permission:** Never equate a high numerical score with an automatic green light for high-impact actions.
4. **Evaluate Uncertainty Jointly with Consequence:** A high-uncertainty / low-consequence task can often be automated or routed to a lightweight fallback; a low-uncertainty / high-consequence task often still requires human oversight.
5. **Preserve Meaningful Human Control:** For high-consequence decisions (financial, security, legal, employment, safety), ensure a human operator has the context, authority, and time to intervene meaningfully.
6. **Favor Reversible Actions over Irreversible Actions:** If an automated step cannot be trivially undone (e.g., permanently deleting data vs. soft-flagging for review), require stronger verification gates.
7. **Maintain Complete Auditability & Provenance:** Store the snapshot of input `state`, exact question schemas, model ID, raw probability distributions, chosen answers, and downstream execution traces.
8. **Calibrate Thresholds Empirically:** Never adopt generic constants (e.g., `0.85`, `0.95`) without testing on representative domain data and measuring precision/recall trade-offs.
9. **Treat Structured Output as Evidence, Not Proof:** Verify claims against ground-truth lookups or cross-source checks whenever consequence is non-trivial.
10. **Design for Graceful Escalation:** A workflow should never fail catastrophically when an answer is ambiguous; it should fall back to a reasoning LLM or human operator queue.

---

## 3. Threshold Calibration & Cost Matrix

Different applications carry asymmetrical error costs. Setting thresholds requires analyzing the cost of False Positives ($C_{FP}$) versus False Negatives ($C_{FN}$):

| Domain | False Positive (FP) | False Negative (FN) | Calibration Strategy |
| :--- | :--- | :--- | :--- |
| **Spam Filter** | Legitimate email marked spam (High user friction) | Spam slips into inbox (Minor nuisance) | Calibrate for high precision ($P \ge 0.98$); only auto-archive with overwhelming confidence. |
| **Security Breach Triage** | Normal activity flagged for inspection (Low cost) | Real intruder ignored (Catastrophic breach) | Calibrate for high recall ($R \ge 0.99$); escalate to human even on moderate suspicion ($> 0.30$). |
| **IT Ticket Category** | Ticket sent to Software instead of Network (5 min reroute) | Ticket sits in unassigned queue (SLA breach) | Balance precision/recall; moderate threshold ($0.75$) suffices. |

---

## 4. The Uncertainty vs. Consequence Matrix

Workflows must evaluate tasks against the two-dimensional matrix:

```text
               CONSEQUENCE
          Low               High
       ┌─────────────────┬─────────────────┐
  Low  │  Zone 1: AUTO   │  Zone 2: HUMAN  │
U      │  Full automated │  Mandatory human│
N      │  execution      │  sign-off       │
C      ├─────────────────┼─────────────────┤
E High │  Zone 3: LLM    │  Zone 4: TRIAGE │
R      │  Escalate to    │  Escalate to    │
T      │  Reasoning LLM  │  Human + LLM    │
       └─────────────────┴─────────────────┘
```

- **Zone 1 (Low Consequence, Low Uncertainty):** Auto-execute directly in code. Example: Tagging support ticket topic as "Billing Question".
- **Zone 2 (High Consequence, Low Uncertainty):** Even if the model is 99% confident, route to human verification. Example: Permanently deleting an enterprise organization account.
- **Zone 3 (Low Consequence, High Uncertainty):** Escalate to a reasoning LLM to synthesize context, or route to default general queue. Example: Complex customer feedback without obvious product categorization.
- **Zone 4 (High Consequence, High Uncertainty):** Immediate alert to human specialists; augment with reasoning LLM investigative dossier. Example: Suspected high-value account takeover with conflicting IP geolocation signals.

---

## 5. Escalation Implementation Pattern

```python
class EscalationPolicy:
    def __init__(self, high_consequence_threshold: float, calibrated_confidence: float):
        self.high_consequence_threshold = high_consequence_threshold
        self.calibrated_confidence = calibrated_confidence

    def evaluate_decision(self, judgment, consequence_level: str):
        # Rule 1: High consequence always requires human authority
        if consequence_level == "HIGH":
            return {
                "route": "HUMAN_OPERATOR_QUEUE",
                "reason": "Action carries high consequence; human sign-off mandatory.",
                "evidence": judgment
            }

        # Rule 2: Low confidence requires deeper reasoning
        if judgment.confidence < self.calibrated_confidence:
            return {
                "route": "REASONING_LLM_ESCALATION",
                "reason": f"Confidence {judgment.confidence:.2f} below calibrated threshold {self.calibrated_confidence}.",
                "evidence": judgment
            }

        # Rule 3: Low consequence + high confidence -> Safe for automation
        return {
            "route": "AUTOMATED_EXECUTION",
            "action": judgment.selected_option,
            "evidence": judgment
        }
```

# Example 1: IT Support Ticket Routing & Triage

This worked example demonstrates how to build an enterprise IT support ticket triage pipeline using **n8n / Node.js backend logic**, **deterministic code**, **Jev atomic judgments**, and **governed escalation**.

---

## 1. System Topology

```mermaid
sequenceDiagram
    autonumber
    actor User as Employee / Customer
    participant Workflow as Workflow Engine (n8n / Node.js)
    participant Rules as Deterministic Rules
    participant Jev as Jev / System One
    participant LLM as Reasoning LLM
    participant Agent as IT Specialist / SOC Queue

    User->>Workflow: Submit Support Ticket
    Workflow->>Rules: Sanitize & Check VIP / Domain Blacklist
    alt Deterministic Match (e.g., Spam Domain)
        Rules-->>Workflow: Drop / Auto-close
    else Valid Ticket
        Workflow->>Jev: Evaluate State with Atomic Questions
        Jev-->>Workflow: Typed Decisions + Probabilities
        Workflow->>Workflow: Evaluate Confidence + Consequence Policy
        alt Security Incident Detected (High Consequence)
            Workflow->>Agent: Route to SOC Urgent Human Review
        alt Routine & High Confidence (Low Consequence)
            Workflow->>Workflow: Auto-assign Department & Set SLA
        else Ambiguous / Low Confidence (Routine)
            Workflow->>LLM: Escalate for Contextual Disambiguation
            LLM-->>Workflow: Synthesized Routing Recommendation
            Workflow->>Workflow: Route with AI Explanation Note
        end
    end
```

---

## 2. Step 1: Deterministic Preprocessing & Business Rules

Before invoking any AI, deterministic code scrubs the input and applies hard invariants:

```typescript
// workflow/preprocessor.ts
export interface RawTicket {
  id: string;
  sender_email: string;
  subject: string;
  body: string;
  created_at: string;
}

export function preprocessTicket(raw: RawTicket) {
  // 1. Spam / blocklist check (Deterministic)
  const SPAM_DOMAINS = ["@spamcorp.xyz", "@phish-test.net"];
  if (SPAM_DOMAINS.some(d => raw.sender_email.endsWith(d))) {
    return { action: "DROP", reason: "Known spam domain" };
  }

  // 2. VIP Employee lookup (Deterministic database query)
  const isVip = raw.sender_email.endsWith("@c-suite.enterprise.com");

  // 3. Extract ticket tokens (Deterministic regex)
  const serverMatch = raw.body.match(/SRV-[0-9]{4}/);

  return {
    action: "PROCEED",
    state: {
      ticket_id: raw.id,
      sender: raw.sender_email,
      is_vip: isVip,
      server_ref: serverMatch ? serverMatch[0] : null,
      text: `${raw.subject}\n\n${raw.body}`.trim(),
    }
  };
}
```

---

## 3. Step 2: Bounded Jev Semantic Judgments

We formulate three **atomic, independent questions** over the extracted `state`:

```typescript
// workflow/triage.ts
import { judge } from "./provider-adapter";

export async function evaluateTicketSemantics(state: any) {
  return await judge(state, {
    department: {
      type: "choice",
      instructions: "Assign the primary IT department responsible for this ticket.",
      criteria: {
        network: "VPN issues, Wi-Fi connectivity, DNS, firewall, routers",
        software: "Application crashes, license activation, email client bugs",
        security: "Phishing report, stolen laptop, credential leak, ransomware warning",
        hardware: "Broken screen, docking station failure, keyboard replacement",
        other: "General inquiries or ambiguous requests"
      }
    },
    security_risk: {
      type: "noul",
      instructions: "Does this ticket describe an active security breach or compromised credentials?"
    },
    urgency_level: {
      type: "score",
      instructions: "Assess user-perceived operational urgency based on business impact.",
      criteria: {
        1: "Low: Question or minor cosmetic glitch; work unaffected.",
        2: "Medium: Significant friction; user can perform workaround.",
        3: "High: Entire team or critical system offline; blocking production."
      }
    }
  });
}
```

---

## 4. Step 3: Application-Owned Policy & Escalation

The application engine evaluates the resulting probabilities against clear governance rules:

```typescript
// workflow/policy.ts
export async function executeRoutingPolicy(state: any, jevResult: any) {
  const { department, security_risk, urgency_level } = jevResult.answers;

  // RULE 1: High Consequence Gate (Security)
  // Even if confidence is only moderate, any security risk >= 0.35 goes directly to human SOC
  if (security_risk.probability >= 0.35 || department.selected === "security") {
    return {
      destination: "QUEUE_SOC_HUMAN_TIER_1",
      priority: "CRITICAL",
      reason: `Potential security incident detected (prob: ${security_risk.probability.toFixed(2)})`,
      audit: { state, answers: jevResult.answers }
    };
  }

  // RULE 2: Low Confidence / High Uncertainty Gate
  // If the department choice has low confidence (< 0.70), escalate to a Reasoning LLM
  if (department.confidence < 0.70 || department.selected === "other") {
    const reasoningExplanation = await askReasoningLLM(
      `Analyze this ambiguous IT ticket and suggest an appropriate routing:\n${state.text}`
    );
    return {
      destination: "QUEUE_IT_TRIAGE_DISPATCH",
      priority: urgency_level.score >= 2 ? "HIGH" : "NORMAL",
      reason: `Ambiguous department (confidence: ${department.confidence.toFixed(2)}). LLM Suggestion: ${reasoningExplanation}`,
      audit: { state, answers: jevResult.answers }
    };
  }

  // RULE 3: Low Consequence, High Confidence -> Automated Action
  return {
    destination: `QUEUE_${department.selected.toUpperCase()}`,
    priority: state.is_vip ? "HIGH" : (urgency_level.score >= 2 ? "HIGH" : "NORMAL"),
    reason: `Automated route: ${department.selected} (confidence: ${department.confidence.toFixed(2)})`,
    audit: { state, answers: jevResult.answers }
  };
}
```

---

## 5. Architectural Takeaways

1. **Jev is not the orchestrator:** Jev did not assign tickets in the database, calculate SLA timers, or send notifications. It answered three distinct semantic questions.
2. **Deterministic rules protected the model:** The spam check was performed with zero LLM/AI tokens.
3. **Escalation is multi-tier:** Security risk triggered a human review queue; ambiguous classification triggered a reasoning model; routine tickets routed automatically in milliseconds.

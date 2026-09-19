# Provider Routing: Native TypeSafe & OpenRouter

## 1. Principles of Provider Abstraction

When integrating Jev / System One into your codebase or automated workflows, **strictly separate semantic decision definitions from provider transport mechanisms**:

```text
Decision Specification (Application State + Questions)
                      │
                      ▼
           Provider Adapter Interface
           ├── Native TypeSafe Adapter  (api.typesafe.ai)
           └── OpenRouter Adapter       (openrouter.ai)
```

Business workflow code must only express:
1. The **state** (the context being evaluated).
2. The **questions** (the typed `Choice`, `Noul`, or `Score` definitions).

The underlying HTTP transport, authentication headers, endpoint URLs, and model aliases must be encapsulated in a lightweight adapter function.

---

## 2. Provider Selection Policy

Applications should implement a standard resolution hierarchy to determine which provider to use at runtime:

```python
def resolve_jev_provider(configured_provider: str | None = None) -> str:
    # 1. Explicit application or user configuration overrides defaults
    if configured_provider in ("typesafe", "openrouter"):
        return configured_provider

    # 2. Check for Native TypeSafe credentials
    if os.getenv("TYPESAFE_API_KEY"):
        return "typesafe"

    # 3. Check for OpenRouter credentials
    if os.getenv("OPENROUTER_API_KEY"):
        return "openrouter"

    # 4. Fail fast with actionable remediation
    raise RuntimeError(
        "No Jev credentials found. Please set either TYPESAFE_API_KEY or OPENROUTER_API_KEY."
    )
```

---

## 3. Provider A: Native TypeSafe Integration

Native TypeSafe provides direct, first-party access to Jev models with lowest possible latency and full support for native SDK features.

### Configuration
- `TYPESAFE_API_KEY`: Required. Bearer token provided by TypeSafe AI.
- `TYPESAFE_BASE_URL`: Optional (default: `https://api.typesafe.ai/v1`).
- `TYPESAFE_DEFAULT_MODEL`: Optional (default: `jev-latest`).

### Direct HTTP Contract
- **Endpoint:** `POST https://api.typesafe.ai/v1/systemone`
- **Headers:**
  ```http
  Authorization: Bearer <TYPESAFE_API_KEY>
  Content-Type: application/json
  ```
- **Request Payload:**
  ```json
  {
    "state": {
      "ticket_id": "TCK-1092",
      "text": "Cannot connect to the VPN gateway after password reset."
    },
    "model": "jev-latest",
    "questions": {
      "department": {
        "type": "choice",
        "instructions": "Select the responsible IT department.",
        "criteria": {
          "network": "VPN, Wi-Fi, routers, switches",
          "software": "App bugs, license issues",
          "security": "Breach, malware, compromised credentials"
        }
      },
      "is_urgent": {
        "type": "noul",
        "instructions": "Does this indicate an active production outage?"
      }
    }
  }
  ```
- **Official Skill Reference:** Consult the official `typesafe-ai` skill for detailed Python/TypeScript SDK usage and native primitives documentation.

---

## 4. Provider B: OpenRouter Integration

OpenRouter provides access to Jev through its multi-model router, useful when consolidating AI billing or when TypeSafe direct keys are not provisioned.

### Configuration
- `OPENROUTER_API_KEY`: Required. OpenRouter API key (`sk-or-v1-...`).
- `OPENROUTER_BASE_URL`: Optional (default: `https://openrouter.ai/api/v1`).
- **Verified Model Identifier:** `typesafe/jev-latest` (or specific version `typesafe/jev-1.13`).

### Direct HTTP Contract
Jev is a non-generative System One model. It **does not use the `/v1/chat/completions` endpoint** or `messages` arrays. It is accessed via the dedicated Decisions API:

- **Endpoint:** `POST https://openrouter.ai/api/alpha/decisions`
- **Headers:**
  ```http
  Authorization: Bearer <OPENROUTER_API_KEY>
  Content-Type: application/json
  HTTP-Referer: <YOUR_APP_URL>
  X-Title: <YOUR_APP_NAME>
  ```
- **Request Payload:**
  ```json
  {
    "model": "typesafe/jev-latest",
    "state": {
      "ticket_id": "TCK-1092",
      "text": "Cannot connect to the VPN gateway after password reset."
    },
    "questions": {
      "department": {
        "type": "choice",
        "instructions": "Select the responsible IT department.",
        "criteria": {
          "network": "VPN, Wi-Fi, routers, switches",
          "software": "App bugs, license issues",
          "security": "Breach, malware, compromised credentials"
        }
      },
      "is_urgent": {
        "type": "noul",
        "instructions": "Does this indicate an active production outage?"
      }
    }
  }
  ```

---

## 5. Critical Distinction: No Direct SDK Swapping

> [!CAUTION]
> **DO NOT simply point the TypeSafe Native SDK at the OpenRouter Base URL.**
>
> The underlying model is the same, but the transport layers differ:
> 1. **Model Aliases:** Native uses `jev-latest`. OpenRouter requires the vendor namespace: `typesafe/jev-latest`.
> 2. **Endpoints:** Native uses `/v1/systemone`. OpenRouter routes decisions via `/api/alpha/decisions`.
> 3. **Headers & Metadata:** OpenRouter expects optional ranking headers (`HTTP-Referer`, `X-Title`) and standard OpenRouter authorization tokens.

---

## 6. Recommended Lightweight Adapter Pattern (TypeScript / JavaScript)

Keep the abstraction minimal. Do not create an overengineered framework:

```typescript
// types.ts
export type Question = 
  | { type: "choice"; instructions: string; criteria: Record<string, string> }
  | { type: "noul"; instructions: string }
  | { type: "score"; instructions: string; criteria?: Record<string, string> };

export interface JudgeResult {
  answers: Record<string, any>;
  usage?: { inputTokens: number; outputTokens: number };
  provider: "typesafe" | "openrouter";
}

// judge.ts
export async function judge(
  state: unknown,
  questions: Record<string, Question>,
  overrideProvider?: "typesafe" | "openrouter"
): Promise<JudgeResult> {
  const typesafeKey = process.env.TYPESAFE_API_KEY;
  const openrouterKey = process.env.OPENROUTER_API_KEY;

  const provider =
    overrideProvider ||
    (typesafeKey ? "typesafe" : openrouterKey ? "openrouter" : null);

  if (!provider) {
    throw new Error(
      "No Jev credentials found. Please set TYPESAFE_API_KEY or OPENROUTER_API_KEY."
    );
  }

  if (provider === "typesafe") {
    const res = await fetch("https://api.typesafe.ai/v1/systemone", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${typesafeKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        state,
        model: "jev-latest",
        questions,
      }),
    });
    if (!res.ok) throw new Error(`TypeSafe API error: ${res.statusText}`);
    const data = await res.json();
    return { answers: data.answers, usage: data.usage, provider: "typesafe" };
  } else {
    const res = await fetch("https://openrouter.ai/api/alpha/decisions", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${openrouterKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "typesafe/jev-latest",
        state,
        questions,
      }),
    });
    if (!res.ok) throw new Error(`OpenRouter Decisions API error: ${res.statusText}`);
    const data = await res.json();
    return { answers: data.answers, usage: data.usage, provider: "openrouter" };
  }
}
```

---

## 7. Security Rules for Provider Integration

1. **Zero Secret Leakage:** Never print, log, serialize, or commit `TYPESAFE_API_KEY` or `OPENROUTER_API_KEY`.
2. **Server-Side Only:** In web architectures, Jev API requests must execute exclusively in trusted backend services or workers. Never invoke Jev directly from client browsers.
3. **Audit Redaction:** Ensure customer PII or sensitive secrets are masked in `state` before logging payload traces for debugging.

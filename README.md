<p align="center">
  <img src="./audit_trailmix.png" alt="Audit TrailMix" width="200"/>
  <img src="./audit_trailmix_black.png" alt="Audit TrailMix Black" width="200"/>
  <img src="./audit_trailmix_cordovan.png" alt="Audit TrailMix Cordovan" width="200"/>
  <img src="./audit_trailmix_white.png" alt="Audit TrailMix White" width="200"/>
</p>

# *Receipts for every AI decision.*
##### Audit TrailMix is a drop-in SDK and control plane that records each agent and LLM call, enforces policy at runtime, and exports signed evidence packs for audits, incidents, and regulators.

## Contents

* [Why TrailMix](#why-trailmix)
* [What you get](#what-you-get)
* [Architecture](#architecture)
* [Install](#install)

  * [Quickstart: Python](#quickstart-python)
  * [Quickstart: TypeScript](#quickstart-typescript)
* [SDK Reference](#sdk-reference)
* [Policies](#policies)
* [Evidence Packs](#evidence-packs)
* [Integrations](#integrations)
* [Security](#security)
* [Repo Layout](#repo-layout)
* [Examples](#examples)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [License](#license)

---

## Why TrailMix

Teams ship AI into customer workflows, but security and compliance need proof:

* What happened, when, and why
* Which model and tools were used
* Which policies were enforced
* Who approved the risky action
* Can we replay it and export artifacts for auditors

TrailMix answers these questions and gives you a signed paper trail without slowing developers down.

## What you get

* **Tamper-evident traces**
  Hash-chained events, per-tenant keys, WORM storage classes.

* **Runtime policy enforcement**
  OPA/Rego policies that gate tools and routes before actions occur.

* **Explainable decisions**
  Plans, retrieval snippets, tool calls, model params, costs, eval scores.

* **One-click evidence packs**
  JSONL + PDF summary with signatures, policy versions, checksums, and external timestamps.

* **SIEM and ticketing hooks**
  Splunk, Sentinel, Datadog, Jira, Slack or Teams.

* **Replay**
  Deterministic re-execution with fixed seeds and versioned policies.

---

## Architecture

```mermaid
flowchart LR
  subgraph App_Agent
    A1[Your Service] --> A2[TrailMix SDK]
    A2 --> A3[Policy Check]
    A2 --> A4[Event Emit]
  end

  A3 -->|allow / deny / approve| A1
  A4 --> Q[Event Bus]
  Q  --> I[Ingest API]
  I  --> S[Append Store + Hash Chain]
  S  --> CP[Control Plane UI]
  CP --> EP[Evidence Packager]
  CP --> SIEM[SIEM and Alerts]
  CP --> APR[Approvals: Slack and Teams]


```

**Key parts**

* Client SDKs: `@trailmix/ts` and `trailmix` (Python)
* Ingest API: authenticated append with tenant scoping
* Policy engine: OPA in WASM with hot-reload bundles
* Storage: Postgres for metadata, S3/GCS for blobs, periodic external timestamping
* Control Plane: Next.js app for search, diff, replay, export

---

## Install

### Requirements

* Node 18+ or Python 3.10+
* Docker and Docker Compose (for local stack)
* PostgreSQL 14+, S3-compatible object store (local: MinIO)

### Local stack

```bash
git clone https://github.com/your-org/audit-trailmix.git
cd audit-trailmix
docker compose up -d
```

Environment defaults are in `.env.example`. Copy and edit as needed.

```bash
cp .env.example .env
```

Important variables:

```
TRAILMIX_SIGNING_SECRET=replace-me
TRAILMIX_DATABASE_URL=postgresql://trailmix:trailmix@localhost:5432/trailmix
TRAILMIX_BLOB_BUCKET=trailmix
TRAILMIX_S3_ENDPOINT=http://localhost:9000
TRAILMIX_S3_ACCESS_KEY=minio
TRAILMIX_S3_SECRET_KEY=minio123
```

---

## Quickstart: Python

Install SDK:

```bash
pip install trailmix-sdk
```

Wrap an LLM call:

```python
from trailmix import TrailMix, span, policy_check
from llm_client import llm

tm = TrailMix(api_key="TRAILMIX_API_KEY", app_id="support-bot")

@policy_check(action="issue_refund", risk="high", amount_field="amount")
def issue_refund(amount: float, customer_id: str):
    # your sensitive tool
    return {"ok": True, "amount": amount, "customer_id": customer_id}

with span(name="draft_refund_reply", risk="medium") as s:
    prompt = f"Draft a polite email approving refund for customer 42."
    out = llm.chat(prompt)
    s.record_input({"prompt": prompt})
    s.record_output({"model_output": out})
    s.attach_eval(name="toxicity", score=0.01, threshold=0.1)

# gated tool call with policy and approval capture
res = issue_refund(amount=750.0, customer_id="42")
print(res)
```

Run and open the Control Plane at `http://localhost:3000` to see traces.

---

## Quickstart: TypeScript

Install SDK:

```bash
npm i @trailmix/ts
```

Wrap a call and gate a tool:

```ts
import { tr, span, policyCheck } from "@trailmix/ts";
import { sendWire } from "./bank";

tr.init({ apiKey: process.env.TRAILMIX_API_KEY!, appId: "finance-bot" });

const draft = await span("draft_wire_email", { risk: "medium" }, async s => {
  const prompt = "Draft a confirmation for a $12,000 wire to EU account.";
  s.recordInput({ prompt });
  const out = await llm.chat({ prompt });
  s.recordOutput({ model_output: out });
  s.attachEval({ name: "groundedness", score: 0.92, threshold: 0.8 });
  return out;
});

const guardedWire = policyCheck(sendWire, {
  action: "send_wire",
  risk: "high",
  metadata: { currency: "EUR" }
});

await guardedWire({ amount: 12000, iban: "DE00..." });
```

---

## SDK Reference

### Core types

```ts
type SpanOptions = {
  risk?: "low" | "medium" | "high";
  labels?: Record<string, string>;
};

type Event = {
  event_id: string;
  parent_id?: string;
  session_id: string;
  ts: string;
  actor_type: "user" | "agent" | "service";
  app_id: string;
  input?: any;
  output?: any;
  tools_invoked?: Array<{ name: string; args: any; result: any; latency_ms: number }>;
  model?: { name: string; version?: string; params?: any; provider?: string; cost?: number };
  policy_eval?: { policy_id: string; decision: "allow" | "deny" | "approve"; reasons: string[]; approvals?: any[] };
  evals?: Array<{ name: string; score: number; threshold?: number; details?: any }>;
  hash_prev?: string;
  hash_self?: string;
  signature?: string;
}
```

### API (TS)

```ts
tr.init({ apiKey, appId, endpoint? })
span(name, options, fn) // transactional wrapper that emits a span event
recordInput(obj)
recordOutput(obj)
attachEval({ name, score, threshold?, details? })
policyCheck(fn, { action, risk, metadata? }) // wraps a function with a preflight policy evaluation
```

### API (Python)

```python
TrailMix(api_key, app_id, endpoint=None)
@span(name: str, risk: str = "low")
def your_fn(...): ...
span(name, risk) as s: ...
s.record_input(obj); s.record_output(obj); s.attach_eval(name, score, threshold=None, details=None)

@policy_check(action: str, risk: str = "low", **kwargs)
def tool(...): ...
```

---

## Policies

Policies are written in Rego and evaluated in a WASM bundle. The engine sees a normalized input that includes actor, action, risk, payload, and current policy version.

**Example: require approval for refunds above 500**

```rego
package trailmix

default decision := {"result": "allow", "reasons": []}

high_value_refund {
  input.action == "issue_refund"
  input.payload.amount > 500
}

decision := {"result": "approve", "reasons": ["refund > 500"]} { high_value_refund }
```

**Example: block PII leaving region**

```rego
package trailmix

deny_pii_eu {
  input.action == "llm_call"
  input.context.region == "EU"
  some p
  p := input.payload.contains_pii
}

decision := {"result": "deny", "reasons": ["PII cannot exit EU"]} { deny_pii_eu }
```

Bundle policies:

```bash
opa build -t wasm -e trailmix/decision policies/
```

Place the `.wasm` and `data.json` in `apps/policy-engine/bundles/`.

---

## Evidence Packs

Click **Export Evidence** in the Control Plane or call the API:

```
POST /api/evidence/export
{
  "app_id": "support-bot",
  "time_from": "2025-09-01T00:00:00Z",
  "time_to": "2025-09-30T23:59:59Z",
  "include": ["traces","policies","evals","approvals"]
}
```

Output:

* `events.jsonl` with signed, hash-chained events
* `policies.json` with versions and checksums
* `summary.pdf` with Merkle root, counts, decision ratios
* `checksums.txt` and an external timestamp receipt

---

## Integrations

**Inbound**

* OpenAI, Azure OpenAI, Anthropic, Vertex, Bedrock
* Tool wrappers via SDKs

**Outbound**

* SIEM: Splunk HEC, Microsoft Sentinel, Datadog
* Chat: Slack, Microsoft Teams
* Tickets: Jira, ServiceNow
* Storage: S3, GCS
* CI: GitHub Actions comment bot for coverage and policy status

---

## Security

* Per-tenant keys and signatures
* Field-level encryption for sensitive payloads
* Deterministic tokenization for PII with reversible reveal by role and purpose
* SSO (SAML/OIDC), SCIM provisioning, RBAC with purpose binding
* IP allowlists and private networking
* Data residency controls and retention classes, legal hold

**Hash chain**

* Each event includes `hash_prev` and `hash_self`
* Periodic Merkle root checkpoints with external timestamps

---

## Repo Layout

```
audit-trailmix/
  apps/
    control-plane/          # Next.js UI (traces, diff, replay, export, approvals)
    policy-engine/          # OPA bundles, Rego tests, WASM build
    event-ingestor/         # FastAPI/Express, signature checks, queue
    evidence-packager/      # Exporter for JSONL + PDF, checksum and timestamp
  sdk/
    python/trailmix/        # Python client SDK
    typescript/trailmix/    # TS client SDK
  connectors/
    slack/
    teams/
    jira/
    splunk/
    sentinel/
  infra/
    helm/
    terraform/
  examples/
    fastapi_bot/
    node_agent/
  docs/
    quickstart.md
    policies.md
    evidence_packs.md
    security.md
```

---

## Examples

### Support bot with refund approvals (Python)

```python
with span("support_flow", risk="medium") as s:
    resp = llm.chat("Customer asks for refund of $750 due to defect.")
    s.record_output({"assistant": resp})
    if "refund" in resp.lower():
        s.attach_eval(name="hallucination", score=0.02, threshold=0.2)
        result = issue_refund(amount=750, customer_id="42")  # blocked until approved
```

### Finance tool gating with Teams approvals (TS)

```ts
const pay = policyCheck(payVendor, { action: "pay_vendor", risk: "high" });
await pay({ vendorId: "acme", amount: 5400 }); // Approval Butler posts to Teams
```

---

## Roadmap

* Multi-env policy versioning with diff and revert
* Snowflake and BigQuery event sinks
* Redaction templates for PHI and PCI variants
* Automated groundedness checks with retriever snippets
* On-prem and air-gapped build profile
* Org-wide incident drill pack (scripted export for tabletop exercises)

---

## Contributing

1. Fork and branch from `main`.
2. Add or update tests.
3. Run `make ci` locally.
4. Open a PR with a clear problem statement and screenshots for UI changes.

Please avoid adding new dependencies unless there is a clear benefit. Keep the SDKs tiny and boring.

---

## License

MIT. See [LICENSE](LICENSE).

---

### Appendix: Event schema (JSON Schema excerpt)

```json
{
  "$id": "https://trailmix.ai/schema/event.json",
  "type": "object",
  "required": ["event_id", "session_id", "ts", "actor_type", "app_id", "hash_self"],
  "properties": {
    "event_id": { "type": "string" },
    "parent_id": { "type": "string" },
    "session_id": { "type": "string" },
    "ts": { "type": "string", "format": "date-time" },
    "actor_type": { "enum": ["user","agent","service"] },
    "app_id": { "type": "string" },
    "input": {},
    "output": {},
    "tools_invoked": { "type": "array" },
    "model": { "type": "object" },
    "policy_eval": { "type": "object" },
    "evals": { "type": "array" },
    "hash_prev": { "type": "string" },
    "hash_self": { "type": "string" },
    "signature": { "type": "string" },
    "retention_class": { "type": "string" }
  }
}
```

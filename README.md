# AgenticRail

**AI agent sequence enforcement infrastructure.**

Not logging. Not alerting. Blocking.

---

## Live

**[agenticrail.nz](https://agenticrail.nz)** — landing page  
**[agenticrail.nz/docs](https://agenticrail.nz/docs)** — full API documentation  
**[agenticrail.nz/evaluate](https://agenticrail.nz/evaluate)** — gate endpoint  

Demo key (public, live): `DEMO-AGENTICRAIL-PUBLIC-2026`

No login. No setup. Hit the gate right now.

---

## The Problem

AI agents fail silently.

They retry completed steps. They replay requests with identical nonces. They skip required validation steps and jump ahead. They re-enter sequences that have already been sealed and closed.

Most AI safety tooling watches this happen and logs it.

AgenticRail sits beneath your agent and blocks it before execution.

---

## What It Enforces

| Law | What it means |
|-----|--------------|
| **Order** | Steps must execute in sequence. No exceptions. |
| **Replay** | A nonce can only be used once. Identical requests are blocked permanently. |
| **Skip** | No step can be bypassed. Every step must execute in order. |
| **Seal** | Once a sequence reaches its terminal step, it is closed. No re-entry. Ever. |

Every enforcement decision produces a **cryptographic receipt** — a `pack_id` signed at the edge. The audit trail is structural, not procedural.

---

## Architecture

```
Internet
    │
    ▼
┌─────────────────────────────┐
│  Gate Worker (PUBLIC)        │
│  slp8-gate                  │
│  · API key validation        │
│  · Request hardening         │
│  · CORS enforcement          │
└──────────────┬──────────────┘
               │ Service Binding
               │ (air gap — no public route to Core)
               ▼
┌─────────────────────────────┐
│  Core Worker (PRIVATE)       │
│  slp8-core                  │
│  · Sequence state            │
│  · Nonce registry            │
│  · Seal enforcement          │
│  · Receipt generation        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Durable Object              │
│  · Per-key, per-sequence     │
│  · Globally consistent       │
│  · Serialised access         │
└─────────────────────────────┘
```

**The enforcement core has no public route.**

`workers_dev = false` — Core cannot be called directly from the internet. It only accepts requests through the Gate's internal service binding, and only when those requests carry the shared internal secret it validates before processing anything.

This is not a software rule. It is an architectural guarantee. Even a compromised endpoint cannot reach the sequence enforcement core.

---

## API

**Endpoint:** `POST https://agenticrail.nz/evaluate`

**Headers:**
```
Content-Type: application/json
x-slp8-key: YOUR_API_KEY
```

**Request payload:**
```json
{
  "schema_version": "1.0",
  "model_id": "MSMD",
  "sequence_id": "unique-sequence-identifier",
  "step": "intake",
  "function": "intake",
  "action_type": "CHECK_STATE",
  "action": "human_readable_label",
  "inputs": { "signal": "..." },
  "nonce": "unique-per-request-uuid",
  "ts_ms": 1234567890000
}
```

**Response — ALLOW:**
```json
{
  "status": "OK",
  "pack": {
    "pack_version": "slp8_pack_1.0",
    "decision": "ALLOW",
    "reasons": [],
    "executed": true
  },
  "pack_id": "sha256-receipt-hash"
}
```

**Response — DENY:**
```json
{
  "status": "OK",
  "pack": {
    "pack_version": "slp8_pack_1.0",
    "decision": "DENY",
    "reasons": ["SEQUENCE_VIOLATION"],
    "executed": false
  },
  "pack_id": "sha256-receipt-hash"
}
```

**Denial reasons:**
| Reason | Meaning |
|--------|---------|
| `SEQUENCE_VIOLATION` | Step sent out of order |
| `REPLAY_NONCE` | Nonce already used |
| `SEALED_SEQUENCE` | Sequence already completed and closed |
| `ACTION_NOT_ALLOWED` | action_type not valid for this function |

Full field reference and quickstart examples at **[agenticrail.nz/docs](https://agenticrail.nz/docs)**.

---

## Enforcement Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| Correct sequence | intake → disruption → ... → settle in order | ✓ ALLOW all |
| Replay attack | intake sent twice with identical nonce | ✗ DENY second |
| Sequence skip | intake → instability (disruption never sent) | ✗ DENY skip |
| Sealed breach | Full sequence completed → intake again | ✗ DENY re-entry |

---

## Pressure Test Results (2026-04-04)

100,002 requests. Concurrency 150. Sustained for 13 minutes.

| Enforcement property | Result |
|----------------------|--------|
| Valid transition → ALLOW | 99.97% |
| Skip transition → DENY | **100%** |
| Replay transition → DENY | **100%** |
| Race condition (200 concurrent bursts) | **0 leaks** |
| Nonce replay (300 sequences) | **100% blocked** |
| Seal (100 full spine walks) | **100% sealed, 0 leaked** |
| Errors | 0.017% (Cloudflare platform ceiling at sustained concurrency-150) |

0 enforcement failures across all scenarios.

---

## Performance

- **Throughput:** 128 RPS sustained at concurrency-150
- **CPU per decision:** 0.87ms (enforcement logic only)
- **End-to-end latency p50:** 890ms (including cryptographic receipt write to R2)
- **Error rate:** 0% enforcement failures
- **Infrastructure:** Cloudflare Workers + Durable Objects + R2 + KV
- **Availability:** Global edge — no single point of failure

---

## EU AI Act

High-risk AI provisions are **fully enforceable August 2026.**

The Act requires demonstrable control over AI decision sequences and verifiable audit trails. Most teams are relying on logging and procedural controls. Regulators asking *"how do you demonstrate your AI cannot skip required steps?"* need a structural answer, not a procedural one.

AgenticRail provides:
- **Architectural enforcement** — the sequence cannot be bypassed at the infrastructure level
- **Cryptographic receipts** — every decision is signed and verifiable
- **Air-gapped core** — enforcement layer is physically isolated from the public internet

---

## Origin

AgenticRail didn't start with a product brief.

It started with a carving framework — [MSMD](https://msmdnwortopknotthinking.substack.com) — that taught sequence order as law, not suggestion. That a chisel removes ambiguity. That skipping a stage destabilises the whole system.

The same law runs the [Rātā Gate](https://mokoseemokodo.com/rata_gate?skin=rata_spectacle_triplehelix_v12) — a generative visual engine that withholds expression until rhythm is achieved. Same invariant law, different expression.

AgenticRail is that law enforced at the infrastructure layer.

---

## Build Journal

Follow the build at **[AgenticRail on Substack](https://msmdnwortopknotthinking.substack.com)**

- [AI agents don't fail loudly. That's the problem.](https://msmdnwortopknotthinking.substack.com/p/ai-agents-dont-fail-loudly-thats)
- [The air gap is the product.](https://msmdnwortopknotthinking.substack.com/p/the-air-gap-is-the-product)
- [Before the code, there was the gate.](https://msmdnwortopknotthinking.substack.com/p/before-the-code-there-was-the-gate)

---

## Status

| Component | Status |
|-----------|--------|
| Gate Worker | ✅ Live |
| Core Worker | ✅ Live (private) |
| Demo UI | ✅ Live |
| Developer Docs | ✅ Live |
| Per-key sequence isolation | ✅ Live |
| Cryptographic receipt log (R2) | ✅ Live |
| UNAUNAHI Observability Dashboard | ✅ Live |
| Ledger export (CSV/JSON) | 🔧 Planned |

---

## Contact

**hello@agenticrail.nz**

If you're building agentic AI in production — especially in a regulated environment — I'd like to know what your current enforcement story looks like.

---

*Built by Kade. Sequence is law.*

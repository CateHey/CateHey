# MineOps Dispatch

> WhatsApp-to-control-room automation for mining operations — real-time field reporting without forms or radio calls.

A production-grade system that lets mining operators (drillers, blasters, transport drivers) report operation status by sending a single WhatsApp message. An AI agent classifies the report, updates the operations database, and confirms back to the operator — end to end in under 3 seconds.

**Built as a consulting deliverable for a mining operations company in Chile.**

https://mining-operations-n8n-automation-2d.vercel.app/

---

## The problem

Mining operations lose critical time to reporting friction. Operators finish a drilling run, a blast, or a transport haul, then need to radio the control room, wait for the dispatcher, and relay details that get manually entered into the system. Status updates are delayed, shift handoffs lose context, and suspended operations sit unreported until someone asks. In remote mine sites with limited connectivity, this lag affects safety decisions and production targets.

## The solution

A single-channel reporting handler. Operators send one WhatsApp message in plain language — *"Voladura completada zona norte, 3 detonaciones sin incidente"* — and the system handles the rest:

1. Identifies the operator by phone number
2. Looks up their active operation for the current shift
3. Classifies the message into one of three outcomes using Claude
4. Updates the operations database, logs variations, or finds the next available shift slot
5. Sends a structured confirmation back via WhatsApp
6. Surfaces every event on a real-time control room dashboard

---

## Architecture

```
┌─────────────┐    ┌─────────────┐    ┌──────────────┐
│  WhatsApp   │───▶│   Twilio    │───▶│     n8n      │
│  (field)    │    │   webhook   │    │  orchestrator│
└─────────────┘    └─────────────┘    └──────┬───────┘
       ▲                                     │
       │                                     ▼
       │                            ┌──────────────┐
       │                            │  Claude API  │
       │                            │ intent class.│
       │                            └──────┬───────┘
       │                                   │
       │                                   ▼
       │                            ┌──────────────┐
       │  confirmation              │   Supabase   │
       └────────────────────────────│  (Postgres)  │
                                    └──────┬───────┘
                                           │ realtime
                                           ▼
                                    ┌──────────────┐
                                    │  Dashboard   │
                                    │  (Vercel)    │
                                    └──────────────┘
```

---

## Key technical decisions

### Intent classification with Claude instead of keyword matching
Operators in mining sites communicate informally, mixing technical jargon with colloquial Spanish. The system needs to extract structured data (tonnage, number of trips, equipment IDs, suspension reasons) from free-form text. Claude returns a confidence score; messages below 60% trigger a clarification reply instead of a wrong action.

### Supabase Realtime for the control room dashboard
The dashboard subscribes to Postgres change events via WebSockets — no polling, no custom backend. When n8n writes a status update, the control room sees it within ~200ms. This is critical for shift supervisors monitoring multiple active operations.

### Confidence gating before destructive actions
Every classification result passes through a confidence check before any database mutation. Low-confidence messages are bounced back to the operator with structured options (`LISTO / VARIACION / SUSPENDER`) — preventing wrong-state writes when the model is uncertain.

### Idempotent webhook handler
The webhook returns `200 OK` immediately, then processes asynchronously. Twilio retries on timeout, so the handler tolerates duplicate inbound messages safely.

---

## Stack

| Layer | Technology |
|---|---|
| Messaging | Twilio WhatsApp API |
| Orchestration | n8n (self-hosted) |
| AI / NLU | Claude (Anthropic API) |
| Database | Supabase (Postgres + Realtime) |
| ERP Integration | Mining ERP (REST API) |
| Frontend | Vanilla JS + Tailwind CDN |
| Hosting | Vercel |

---

## Outcomes handled

| # | Example message from operator | Action chain |
|---|---|---|
| **1** | *"Voladura completada zona norte, sin novedad"* | Mark operation complete → update Supabase → log event → confirm via WhatsApp |
| **2** | *"Transporte listo, 3 viajes extra, 45 toneladas adicionales por desvio en ruta"* | Extract variations → log additional tonnage/trips → mark complete → confirm |
| **3** | *"Perforacion suspendida, broca rota, necesitamos repuesto"* | Find next available shift → suspend and reschedule → notify control room → confirm new slot |

Plus four error paths: unknown phone number, no active operation for current shift, multiple operations requiring disambiguation, and ambiguous report (low confidence).

---

## What's in this repo

- `index.html` — The control room dashboard (single-file SPA, deploys directly to Vercel)
- `n8n_workflow.json` — The complete n8n workflow (importable)
- `supabase-schema.sql` — Database schema with sample mining data

---

## Live demo

**Dashboard:** [mining-operations-n8n-automation-2d.vercel.app](https://mining-operations-n8n-automation-2d.vercel.app)
**Try it:** Send a WhatsApp message to `+1 415 523 8886` (Twilio sandbox) — watch the dashboard update in real time.

---

## Why this project matters in a portfolio

This isn't a CRUD app. It's a system that integrates four different APIs across messaging, AI, database, and ERP layers, and ships a real-time control room dashboard on top — all glued together with thoughtful error handling and confidence-based safety rails. It demonstrates:

- **API integration depth** across heterogeneous services
- **LLM productionization** with structured output, confidence gating, and fallbacks
- **Realtime UI patterns** without standing up a custom backend
- **Domain-specific NLU** handling informal, multilingual mining jargon
- **Safety-first design** — audit trail for regulatory compliance, confidence gating before state mutations

---

## About

Built and shipped by **Catherine Varas** as consulting work for a mining operations company.

[LinkedIn](https://www.linkedin.com/in/catherine-varas)
[GitHub](https://github.com/CateHey)

Open to senior backend, full-stack, and AI integration roles.

# Proply Research — Persistent Memory for GTM AI Agents

> The go-to-market stack is being rebuilt around autonomous agents. The missing piece is memory.

This repo documents Proply's research into **persistent memory architecture for GTM agents** — how AI agents can maintain complete, structured relationship context across every channel, signal, and interaction over time.

---

## The Problem

Modern GTM teams are deploying AI agents everywhere: outbound sequences, LinkedIn automation, meeting assistants, CRM enrichment, proposal generation. Each agent operates in isolation. None of them share memory.

An agent sending a cold sequence via Instantly doesn't know the prospect already visited your pricing page three times. A meeting assistant in Fireflies doesn't know they've been in your outbound sequence for six weeks. A proposal agent doesn't know they're already a warm lead.

**The result:** agents that act on incomplete context, sending sequences to people who already met with you, treating hot leads like cold ones, and failing to maintain relationship continuity across the stack.

The GTM agent stack needs a shared, persistent memory layer — the same way every serious application has a database. We call this the **CRM for GTM Agents**.

---

## The Thesis

> Proply is not a CRM for humans. It is the persistent memory layer for AI agents.

The CRM as we know it was designed for human salespeople who need a place to log notes, track pipeline, and run reports. AI agents don't need a dashboard. They need a structured memory they can query before acting, and write to after every interaction.

The right abstraction is:

```
Every GTM signal → normalized → canonical contact → pipeline stage → queryable memory
```

An agent should be able to ask: *"What do I know about this person, across every channel, right now?"* and get a structured, complete answer in milliseconds.

---

## Core Architecture

### 1. The Signal Taxonomy

Not all signals are equal. We classify every GTM signal by intent level:

| Level | Name | Window | Example Signals |
|---|---|---|---|
| 1 | **Aware** | 30 days | website visit, email opened, LinkedIn view |
| 2 | **Interested** | 30 days | email reply, LinkedIn message, content download |
| 3 | **Evaluating** | 60 days | meeting held, pricing page visit, positive reply |
| 4 | **Client** | Permanent | signed, deal won, payment received |

Each signal has a canonical `activity_type`, a source integration, and a recency window. The pipeline stage is computed automatically as the **highest tier with a qualifying signal within its window**.

This means stage is never manually set by a human — it is derived from evidence.

```
website_visit         → aware      (30-day window)
email_reply           → interested (30-day window)
meeting_held          → evaluating (60-day window)
proposal_signed       → client     (permanent, never decays)
```

Full taxonomy: [`signal-taxonomy.md`](./signal-taxonomy.md)

---

### 2. Identity Resolution Waterfall

Before any signal can be stored, it must be resolved to a canonical contact identity. Signals arrive from a dozen platforms, each with their own IDs and schemas. The same person might be `john@acme.com` in HubSpot, `linkedin.com/in/johndoe` in Unipile, and an anonymous visitor ID in RB2B.

Identity resolution runs this waterfall on every incoming signal:

```
1. Match by external platform ID   (hubspot_id, apollo_id, rb2b_id, ...)
2. Match by email                  (normalized: lowercase + trim)
3. Match by LinkedIn URL           (normalized URL)
4. No match → create new contact (Level 1 sources only) or reject (Level 2/3)
```

The **create vs. reject** distinction is critical. Not every signal source should be allowed to create new contacts:

| Level | Sources | Creates contacts? | Rationale |
|---|---|---|---|
| **Level 1** | Instantly, RB2B, LinkedIn, Apollo | Yes | Intentional prospecting signals |
| **Level 2** | Gmail, inbox | No | Any stranger who emails you would become a contact |
| **Level 3** | Fireflies, Calendly | No | Meeting participants ≠ pipeline contacts |

When a contact is matched, field-level merging fills in any blanks from the incoming payload — it never overwrites existing values.

The result: one canonical contact record with all platform IDs, all activity history, and all signals from every tool in the stack.

Full spec: [`identity-resolution.md`](./identity-resolution.md)

---

### 3. The Activity Ledger

Every signal lands in an append-only activity ledger. This is the ground truth of all relationship history.

```
activity_log {
  contact_id      -- canonical contact
  workspace_id    -- tenant isolation
  activity_type   -- canonical type (from taxonomy)
  source          -- originating platform
  external_id     -- platform's own event ID (deduplication key)
  occurred_at     -- when it happened (from platform clock, not ingestion time)
  title           -- human-readable headline
  summary         -- full text (meeting summary, email subject, etc.)
  raw_data        -- complete original payload (for future agent retrieval)
  metadata        -- structured extracted fields (participants, duration, etc.)
}
```

**Deduplication:** unique constraint on `(workspace_id, source, external_id)`. The same event replayed 100 times writes one row.

**Trigger:** every insert fires a database trigger that recomputes the contact's pipeline stage automatically. No application-layer orchestration needed.

---

### 4. Pipeline Stage Compute

Stage is computed by a single function that scans the activity ledger and returns the highest qualifying tier:

```
compute_pipeline_stage(contact_id):
  IF any CLIENT signal ever → return 'client'
  IF any EVALUATING signal in last 60 days → return 'evaluating'
  IF any INTERESTED signal in last 30 days → return 'interested'
  IF any AWARE signal in last 30 days → return 'aware'
  RETURN 'identified'
```

Stages also **decay** when windows expire. A daily cron runs:

```
evaluating → interested   (no EVALUATING signal in 60 days)
interested → aware        (no INTERESTED signal in 30 days)
aware      → identified   (no AWARE signal in 30 days)
client     → never decays
```

Manual overrides are respected but superseded by any upward-moving signal. A human can mark someone as "Evaluating" — but a pricing page visit will confirm it, and a new website visit alone won't override it back down.

Full spec: [`pipeline-stages.md`](./pipeline-stages.md)

---

### 5. Integration Signal Architecture

Integrations are split into three levels based on trust and intent:

**Level 1 — Lead Acquisition (can create new contacts)**
- RB2B — anonymous website visitor identification
- Instantly / Apollo — cold email sequences
- LinkedIn / Unipile — connection requests, profile views, DMs
- HubSpot / Pipedrive — one-time bootstrap (seeds Proply from existing CRM)

**Level 2 — Engagement (update only, never create)**
- Gmail — email threads with existing contacts

**Level 3 — Meetings & Closing (update only, never create)**
- Fireflies / Fathom — meeting transcripts
- Calendly — booking events
- Stripe — payment events

Each integration normalizes its events to canonical activity types before storage. Proply never stores raw CRM records — it stores structured activity signals.

Full spec: [`integrations.md`](./integrations.md)

---

### 6. ICP Scoring

After enrichment (via Apollo), each contact is scored against the workspace's ICP definition using an LLM:

```
ICP score (0–100) = f(
  workspace ICP memory (job titles, industries, company sizes, signals),
  contact enriched data (title, seniority, department, company)
)
```

The score is used by agents to prioritize actions — only contacts above a minimum ICP score trigger certain automations.

---

### 7. Connection Score

Separate from pipeline stage, each contact has a `connection_score` (0–100) reflecting relationship warmth:

| Component | Max | Condition |
|---|---|---|
| Recency | 40 | ≤7 days = 40, ≤30 = 25, ≤90 = 10, older = 0 |
| Touchpoints | 35 | 5pts per activity, capped at 7 |
| Platform diversity | 25 | 8pts per unique source |

A contact you've met once on a call scores differently from one you've emailed 10 times across three channels.

---

### 8. Automation Rules Engine

After every activity write, the rules engine evaluates automation triggers:

```
trigger: contact_activity
conditions:
  activity_type: meeting_held
  min_icp_score: 70
action:
  create_task: "Send follow-up proposal"
```

This gives agents a way to subscribe to relationship state changes without polling.

---

## Why This Matters for AI Agents

The architecture above enables a specific pattern that doesn't exist in any current CRM:

**Agents can query relationship context before acting:**

```
GET /memory/{contact_id}
→ {
    stage: "evaluating",
    connection_score: 72,
    icp_score: 85,
    last_activity: { type: "meeting_held", occurred_at: "2026-04-17", summary: "..." },
    activities_30d: [...],
    signals: { website_visits: 3, emails_replied: 1, meetings: 1 }
  }
```

An outbound agent can check this before sending a sequence and skip contacts already in `evaluating`. A proposal agent can read the meeting transcript from Fireflies and personalize the proposal. A follow-up agent can know exactly how warm a lead is before choosing a message tone.

**This is not what HubSpot is for.** HubSpot stores records for humans to read. Proply stores structured memory for agents to query.

---

## What's Open vs. Closed

This repo documents the **architecture and schema** — the opinion on how GTM memory should work.

The hosted Proply service (closed) handles:
- The ingestion pipeline (webhook handlers, normalization, deduplication)
- Multi-tenant isolation and data security
- ICP scoring prompt engineering
- Identity resolution edge cases at scale
- Integration credential management

The reference implementation [`proply/lite`](https://github.com/bennetglinder1/proply-lite) shows the core pattern running on SQLite — educational, not production-ready.

---

## Papers & Prior Art

The architecture draws from:

- **MemGPT** — paging memory in and out of context
- **Reflexion** — agents that write and update their own memories after acting
- **LangMem** — structured memory for long-horizon agent tasks
- **GraphRAG** — relationship graphs as retrieval context

GTM-specific prior art is sparse. The closest analogues are CRM systems designed for humans, not agents. This is the gap Proply is designed to fill.

---

## License

Research and schema: MIT. Referenced implementation: see [`proply/lite`](https://github.com/bennetglinder1/proply-lite).

# Entity Resolution in Real-Time GTM Data Streams

*Original research — Proply*

---

## The Problem

A contact visits your pricing page. RB2B fires a webhook: `{ visitor_id: "rb2b_8821", email: "ben@acme.com", name: "Ben Carter" }`.

Three hours later, Instantly fires: `{ email: "b.carter@acme.com", event: "email_replied", name: "Benjamin Carter" }`.

Next week, Fireflies fires: participant `"Ben Carter, Acme Corp"` in a meeting transcript.

Are these the same person? Almost certainly. But no system told you that. Three different platforms, three different identifiers, three different name formats for what is — in the real world — one relationship with one human being.

This is entity resolution in a GTM data stream: the continuous, real-time problem of matching incoming signal records to canonical contact identities, across platforms that have never heard of each other, under latency constraints that don't allow for batch processing or LLM inference on every record.

Getting it wrong in either direction is expensive. Miss a match (false negative) and you create a duplicate contact — your agent treats a warm, active lead as a cold unknown. Merge incorrectly (false positive) and you corrupt two separate relationships into one — your agent acts on the wrong person's history.

This document is our current thinking on how to solve it.

---

## Why GTM Streams Are Different From Standard ER

Entity resolution is a well-studied problem in databases and NLP. Most of the literature assumes a batch context: you have a dataset, you run ER, you get a resolved output. GTM data streams break nearly every assumption that literature makes.

**Records arrive continuously, not in batches.** A webhook fires when a contact visits your site, opens an email, or books a meeting. You have milliseconds to resolve their identity before the signal is lost or stored incorrectly. You cannot queue it for a nightly batch run.

**Records arrive from systems that don't share identifiers.** RB2B identifies visitors by IP and email. Instantly tracks by sequence contact ID. LinkedIn uses profile URL. HubSpot uses its own internal ID. None of these are compatible. The only universal identifier is email — but email is only present in roughly 60–70% of incoming signals.

**Records carry different levels of identifying information.** A Fireflies meeting transcript gives you participant names but often no emails. An RB2B visit gives you email but a different name format. A LinkedIn connection event gives you a URL but no email at all. Every source is missing something.

**The volume is asymmetric.** Most incoming signals (80–90%) resolve cleanly — they carry an email or external ID that matches exactly. A small fraction (10–20%) don't, and those require more expensive resolution logic. The system has to be designed so the fast path is very fast and the slow path is accurate.

**Resolution errors compound.** In a static dataset, an ER error affects one record. In a live GTM stream, an ER error means every subsequent signal from that source gets attributed to the wrong contact. A wrong merge in January corrupts the entire activity history for the life of that contact.

---

## The Architecture: A Tiered Waterfall

The right architecture for real-time GTM ER is a **tiered waterfall**: a sequence of resolution strategies ordered from most discriminative to least, stopping at the first successful match. Each tier is faster and more reliable than the one below it.

```
Signal arrives
  ↓
Tier 1: External platform ID match    ← fastest, deterministic
  ↓ (no match)
Tier 2: Email match (normalized)      ← ground truth, near-deterministic
  ↓ (no match)
Tier 3: LinkedIn URL match            ← normalized, reliable
  ↓ (no match)
Tier 4: Fuzzy name + company match   ← probabilistic, needs threshold
  ↓ (no match or below threshold)
Tier 5: Create new contact            ← Level 1 sources only
         or reject                    ← Level 2/3 sources
```

### Tier 1: External ID Match

Every integration that creates contacts assigns its own identifier. RB2B assigns `rb2b_id`. HubSpot assigns `hubspot_id`. Instantly assigns a sequence contact ID. These are stable, unique within their platform, and arrive in the webhook payload.

On first encounter, the external ID is stored alongside the canonical contact. On subsequent encounters, it matches immediately. This tier covers all returning contacts from platforms they've been seen on before — roughly 70–80% of all incoming signals from active integrations.

**Latency:** single index lookup, sub-millisecond.

### Tier 2: Email Match (Normalized)

Email is the closest thing to a universal identifier in GTM. When it's present, it's reliable.

The normalization step matters more than it seems:
- Lowercase the entire string
- Trim whitespace
- Strip display name portions (`"Ben Carter <ben@acme.com>"` → `"ben@acme.com"`)
- Handle `+` aliases consistently (`ben+sales@acme.com` → treat as same domain identity or strip suffix — depends on policy)

Un-normalized email matching causes silent misses. `Ben@Acme.com` and `ben@acme.com` are the same address; a case-sensitive lookup creates a duplicate.

**Latency:** single indexed lookup after normalization, sub-millisecond.

### Tier 3: LinkedIn URL Match

LinkedIn URL is a stable, unique identifier per person. The normalization challenge: URLs arrive in multiple formats.

- `https://linkedin.com/in/bencarter`
- `https://www.linkedin.com/in/bencarter/`
- `linkedin.com/in/bencarter`

All three should normalize to a canonical form (`bencarter`) and match against a normalized index. Case differences, trailing slashes, and protocol prefixes are all noise.

**Latency:** single indexed lookup after normalization, sub-millisecond.

### Tier 4: Fuzzy Name + Company Match

This is the probabilistic tier. It exists for the case where a signal carries a name and company but no email or external ID. Meeting transcripts from Fireflies frequently hit this path — a participant listed as `"Ben Carter, Acme"` with no email in the payload.

The approach: compute a composite similarity score from multiple signals:

```
score = w₁ × name_similarity(incoming, candidate)
      + w₂ × company_similarity(incoming, candidate)
      + w₃ × domain_match(incoming_email_domain, candidate_email_domain)
```

Where `name_similarity` combines:
- **Token sort ratio** — handles `"Carter Ben"` vs `"Ben Carter"`
- **Phonetic encoding** — handles `"Bennet"` vs `"Bennett"` vs `"Ben"`
- **Edit distance** — handles typos and abbreviations

And `company_similarity` handles:
- `"Acme"` vs `"Acme Corp"` vs `"Acme Corporation"` — token overlap
- `"IBM"` vs `"International Business Machines"` — abbreviation expansion (needs a lookup or LLM)

If the composite score exceeds a high threshold (≥0.90): auto-merge.  
If score is between 0.70–0.90: flag for human review, don't auto-merge.  
If score is below 0.70: treat as new contact (if Level 1 source) or reject (Level 2/3).

**The threshold matters.** Too high and you miss real matches. Too low and you corrupt data. For GTM, the asymmetry in error costs (false negative = duplicate contact, false positive = corrupted identity) suggests a high threshold with a human-review queue for the uncertain middle.

**Latency:** 10–50ms depending on candidate pool size and similarity computation.

### Tier 5: Create or Reject

After all tiers fail to find a match, the system makes a binary decision based on signal source:

**Level 1 sources** (Instantly, RB2B, LinkedIn, Apollo, HubSpot) — create a new contact. These represent intentional prospecting activity. If someone visited your site and no match exists, they're a new contact.

**Level 2 sources** (Gmail) — reject silently. Anyone who has ever emailed you would become a contact. The noise ratio is too high.

**Level 3 sources** (Fireflies, Calendly, Fathom) — reject silently. Meeting participants are not necessarily prospects. The `createIfMissing` flag is false; if no match exists, the signal is discarded without creating a record.

This two-class treatment of sources is critical. Without it, a single meeting transcript from a 20-person all-hands call creates 20 spurious contacts, most of them irrelevant.

---

## The Hard Cases

The waterfall handles the majority of signals correctly. The hard cases are worth naming explicitly because they represent the edge cases where the system will fail until explicitly addressed.

### Abbreviations and Expansions

`IBM` and `International Business Machines` are the same company. `SF` and `San Francisco` are the same city. A standard string similarity metric will not catch these — the strings share almost no characters.

Two approaches:
1. **Lookup table** — maintain a dictionary of known abbreviation → expansion mappings for common company names and locations. Simple, fast, limited coverage.
2. **Embedding similarity** — compare vector representations. `IBM` and `International Business Machines` will have high cosine similarity if the embeddings are trained on text that uses both forms. More general, higher latency.

At current scale (thousands of contacts), a curated lookup table is sufficient. At millions of contacts, embedding-based matching becomes the right investment.

### Nicknames

`Ben` and `Bennet`. `Mike` and `Michael`. `Liz` and `Elizabeth`. These are different strings that almost certainly refer to the same person when co-occurring with the same company domain.

The signal for handling this: if company domain matches (or email domain matches) and phonetic encoding of first name matches, boost the composite similarity score significantly. The combination of `same domain + phonetically similar name` is a strong match signal even when the strings don't overlap.

### Contextual Disambiguation

`"Apple"` in a conversation about consumer electronics vs. a conversation about a catering company. `"Mercury"` as a SaaS travel platform vs. a plumbing company. Same string, different entity — and the difference is invisible to any string similarity metric.

This is the hardest class of problem and the one where bottom-up approaches (Graphiti, LightRAG) have an inherent advantage: they use the surrounding context to disambiguate, not just the entity string itself.

For GTM at current scale, contextual disambiguation rarely matters — most incoming signals include company domain or location data that resolves the ambiguity. When it does matter (matching against a large company database, for example), company industry and size data from enrichment provides the context that disambiguates.

### Conflict Resolution

Entity resolution solves the matching problem. Conflict resolution is the sequel: once you know two records refer to the same entity, which field values do you keep?

Two records for the same person might carry:
- Different job titles (one from six months ago, one from today)
- Different company names (one using the legal entity, one using the trading name)
- Different phone numbers (one mobile, one office)

The policy question: who wins?

**By recency:** the most recently received value wins. Simple but not always correct — a stale platform might send an older version of a field.

**By source trust ranking:** assign each integration a trust rank per field type. Apollo is trusted for `job_title` and `company`. LinkedIn is trusted for `linkedin_url`. Gmail is trusted for `email`. When sources conflict, the highest-ranked source for that field wins.

**Fill-only (never overwrite):** only fill blank fields from incoming signals, never update a field that already has a value. Simpler, more conservative, protects against stale data overwriting fresh data.

Proply currently uses fill-only. It's the right default at early scale — it protects against data corruption and requires no configuration. The trust ranking approach becomes valuable when you have both reliable and unreliable sources contributing to the same fields and need the reliable one to win.

---

## Top-Down vs. Bottom-Up in GTM

The academic literature and the systems we've studied frame two architectural philosophies for entity resolution:

**Top-down:** define your entity schema upfront. All incoming data maps to predefined fields. Resolution is deterministic and fast. You're betting that you understand the domain well enough to define the schema before data arrives.

**Bottom-up:** let entity structure emerge from the data. An LLM or ML model extracts entities and relationships from raw text. More flexible, handles unexpected structures, but slower and less consistent.

For GTM, **top-down is the right choice** — and the reasoning is worth being explicit about:

The entities in GTM are known in advance. A contact is a person with an email, a company, a job title, a LinkedIn URL. A signal is a typed event with a source, a timestamp, and intent metadata. These structures don't surprise you. The domain is well-understood.

The signals are structured at source. Webhook payloads from Instantly, RB2B, and HubSpot are JSON with defined schemas. There's nothing to "extract" — the data is already structured. An LLM at ingestion time adds latency and cost without adding information.

The resolution rules are deterministic. Email is ground truth. External IDs are stable. The waterfall is a set of ordered lookups, not a probabilistic model. This determinism is a feature — the same input always produces the same output, which means bugs are reproducible and the system is debuggable.

Bottom-up approaches (Graphiti, LightRAG) shine when you're ingesting unstructured text from diverse sources and can't predict what entities will look like. For GTM contact resolution from known webhook sources, that flexibility comes at unnecessary cost.

The one place bottom-up reasoning enters the GTM stack: **meeting transcript processing**. When Fireflies sends a 10,000-word transcript, extracting participant names, company names, discussed topics, and action items does benefit from LLM extraction. But this is entity extraction from a single document — not real-time streaming entity resolution. It happens asynchronously, after the signal is already stored, not in the resolution path.

---

## The Incremental Problem: Why Streaming ER Is Harder

Standard ER assumes a static dataset. Run the pipeline once, get a resolved output. GTM streaming ER has to handle continuous, incremental updates — and this is where most standard approaches break down.

Three specific challenges that don't exist in batch ER:

**Out-of-order arrival.** A signal from last week might arrive today because a webhook retried. The resolution system has to handle signals that arrive out of chronological order without corrupting the activity history.

**Retroactive merges.** Two contacts were created separately in January. In March, a new signal arrives that matches both — they're the same person. The correct behavior is to merge both contact records and re-attribute all historical activities. This is a complex operation that most systems don't support gracefully.

**Deduplication across replays.** Webhook providers retry failed deliveries. The same signal might arrive 2–5 times. Resolution must be idempotent: processing the same signal twice should produce the same result as processing it once. This is handled by the unique constraint on `(source, external_id)` in the activity log — the same event always maps to the same row.

---

## Where This Goes Next

The waterfall described here handles the majority of cases correctly at current scale. The gaps worth addressing as the system grows:

**Fuzzy name + company matching (Tier 4)** is not yet implemented. Currently, if a Fireflies participant has no email match, they're either created as a new contact or dropped. Adding a similarity-scored fuzzy tier would catch the meeting participants who are already in the system under a slightly different name.

**Abbreviation expansion** for company names matters more as the company database grows. The top 1,000 companies by revenue all have common abbreviation patterns worth cataloging.

**Active learning for the uncertain middle.** When Tier 4 produces a 0.70–0.90 score, the system flags for human review. Each human label can be fed back into the similarity weights — the Dedupe pattern applied to GTM contact resolution. Over time, the threshold can be tuned per industry or company size.

**Company-level resolution.** Currently, resolution is contact-first. Company deduplication is secondary. As account-based GTM motion becomes more important, resolving `"Acme Corp"` → `"Acme Corporation"` → `"ACME"` to a single canonical account becomes as important as contact resolution.

---

## Summary

Entity resolution in real-time GTM streams is the problem of matching continuous, incomplete, cross-platform signal records to canonical contact identities under millisecond latency constraints. The solution is a tiered waterfall: deterministic exact matching in the fast path (tiers 1–3), probabilistic fuzzy matching for the hard cases (tier 4), and a strict create/reject policy based on signal source trust level (tier 5).

The hard cases — abbreviations, nicknames, contextual disambiguation — are known and addressable with the right tooling. The conflict resolution policy (fill-only, source trust ranking, recency) is a configuration decision with explicit tradeoffs, not an algorithmic one.

Top-down entity schema is the right choice for GTM because the domain is well-understood, signals are already structured, and resolution determinism is a feature. Bottom-up approaches are valuable for unstructured text processing (meeting transcripts, email bodies) but not in the real-time signal resolution path.

Getting entity resolution right is the foundational requirement for every other capability in a GTM memory layer. Pipeline stage, ICP score, connection score, agent memory queries — all of these are only as good as the contact identity they're computed on.

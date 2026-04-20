# Graphiti: Temporal Context Graphs for AI Agents

**Source:** Zep — [github.com/getzep/graphiti](https://github.com/getzep/graphiti)  
**Built by:** Zep (the memory infrastructure company behind [getzep.com](https://getzep.com))  
**Type:** Open-source framework / engine  
**License:** Apache 2.0

---

## Why We're Studying This

Graphiti is the most mature open-source framework for persistent agent memory. It's built by Zep, a company that's further along the "memory for agents" problem than almost anyone else. Understanding how they architected it — what choices they made, what tradeoffs they accepted — is directly relevant to how we think about Proply's memory layer.

The central idea we're taking from this: **new data should replace old data based on timestamps, not overwrite it.** Facts have validity windows. When something changes, the old fact is marked invalid — not deleted. You always know what's true now and what was true at any point in the past. That's the core pattern.

---

## 1. The Problem Graphiti Solves

Traditional RAG has a staleness problem. You embed documents, store vectors, retrieve top-k at query time. This works when knowledge is static. It breaks when knowledge changes — because the embedded representation of an outdated fact looks identical to an updated one. There's no mechanism for saying "this was true then, this is true now."

The standard fix — periodically re-embed and re-index everything — is expensive and still doesn't give you historical queryability. You lose the ability to ask "what did we know about this on a specific date?"

Graphiti's answer: stop treating memory as a snapshot and start treating it as a **timeline of facts with explicit validity windows**.

---

## 2. Core Architecture

### The Four Building Blocks

**Episodes**
The raw ground truth. Every piece of data ingested — a message, a document, a webhook payload — becomes an episode. Episodes are never modified. They're the source of record that all derived facts trace back to.

**Entities (Nodes)**
People, companies, products, concepts — anything with an identity. Entities have evolving summaries that update as new episodes arrive, but the entity itself persists across time.

**Facts / Relationships (Edges)**
Triplets connecting entities: `[Subject] → [Relationship] → [Object]`. Every edge has two timestamps:
- `valid_from` — when this fact became true
- `valid_to` — when it was superseded (null if still current)

Example: `"Kendra uses Salesforce (valid: Jan 2025 → Mar 2026)"` becomes invalid when a new episode says she switched to HubSpot. The old edge stays — you can still query it. The new edge gets `valid_from: Mar 2026, valid_to: null`.

**Custom Types (Ontology)**
Developers can define their own entity and edge types via Pydantic models. This lets you bring a prescribed schema to the graph (e.g., always extract `Company` and `Contact` entities) while still allowing emergent structure to form from unstructured data.

---

## 3. Temporal Fact Management: The Core Pattern

This is the most important thing Graphiti gets right and the pattern most worth understanding.

### The problem with deletion

If you overwrite a fact when it changes, you lose history. If you delete old facts, you can't query past state. Both approaches make it impossible to answer questions like "what did we know about this account in Q3?" or "when did this contact's company change size?"

### The solution: validity windows

Every fact has a `valid_from` and `valid_to` timestamp. When a contradicting fact arrives, the system:

1. Detects the contradiction (via LLM comparison or rule)
2. Sets `valid_to` on the old edge to the new episode's timestamp
3. Creates a new edge with `valid_from` set to that same timestamp

The result is **bi-temporal tracking**: you can query by *when something was true in the real world* (valid_from/valid_to) independently of *when Graphiti learned about it* (episode ingestion timestamp). These two timelines are always tracked separately.

### Why this matters

Three classes of questions become answerable that previously weren't:

| Question Type | Without temporal tracking | With temporal tracking |
|---|---|---|
| Current state | Easy | Easy |
| Past state | Impossible | Query by date |
| Change detection | Requires diff logic | Built-in (valid_to set) |

---

## 4. Hybrid Retrieval

Graphiti doesn't choose between semantic search, keyword search, or graph traversal — it runs all three and combines results.

**Semantic embeddings** — vector similarity for conceptually related facts, even when phrasing differs  
**BM25 keyword search** — exact and near-exact term matching, fast and precise  
**Graph traversal** — following relationship edges to find connected facts (e.g., from a person to their company to their company's recent news)

The combination gives sub-second query latency with higher precision than any single method alone. GraphRAG, which relies on LLM summarization at query time, typically takes seconds to tens of seconds. Graphiti achieves the same quality faster by doing the structural work at ingestion time rather than query time.

---

## 5. Incremental Updates (No Batch Reprocessing)

Traditional knowledge graph approaches require periodic full recomputation when new data arrives. Graphiti integrates new episodes immediately and updates only the affected entities and edges.

The practical effect: a new signal arriving at 2pm is queryable at 2pm, not at the next batch run. For agents operating in real time — responding to a webhook, triggering on a calendar event, reacting to a new email — this matters.

---

## 6. The Open/Closed Split

Graphiti is the open-source engine. Zep is the closed hosted service built on top of it.

What Graphiti provides (open): the graph engine, entity extraction, temporal invalidation, hybrid retrieval, the data model.

What Zep adds (closed): user management, conversation threading, pre-configured retrieval pipelines, multi-tenant isolation, enterprise support.

The pattern is identical to what Proply is building: open the engine and the schema, close the hosted execution. Developers adopt through the open layer; companies pay for the hosted layer.

---

## 7. Two Approaches to Agent Memory: Graph vs. Signal Ledger

Graphiti and Proply solve the same underlying problem — persistent memory for AI agents — through different architectural choices. Neither is universally superior; they're optimized for different use cases.

| Dimension | Graphiti (graph-based) | Proply (signal ledger) |
|---|---|---|
| **Domain** | General purpose — any agent, any data | GTM-specific — contacts, signals, pipeline |
| **Schema** | Emergent or prescribed ontology | Pre-defined signal taxonomy |
| **Intelligence location** | LLM extracts entities/relationships at ingestion | Schema encodes domain knowledge upfront |
| **Memory structure** | Knowledge graph (nodes + edges + validity windows) | Append-only activity ledger + computed stage |
| **Retrieval** | Hybrid: embeddings + BM25 + graph traversal | Structured query: stage, score, recency window |
| **Temporal model** | Bi-temporal validity windows on edges | Recency windows + decay compute |
| **Update model** | Incremental, any new data | Webhook-driven, typed signal events |
| **Best for** | Agents operating on unstructured, diverse data | Agents operating on structured GTM signals |

The core tradeoff: Graphiti is flexible because it lets structure emerge from data. Proply is fast and consistent because the structure is defined in advance. For GTM — where you already know that a pricing page visit means high intent, that a meeting beats an email open, that `client` is permanent — encoding that knowledge in the schema produces more reliable, lower-latency results than discovering it at query time.

For domains where you don't know the structure in advance, Graphiti's approach is better. For domains where the structure is well-understood and stable, a typed ledger with pre-computed stage is more efficient.

---

## 8. What We're Taking From This

**Bi-temporal tracking is the right pattern.** Graphiti's separation of "when something was true" vs. "when we learned about it" is exactly what Proply's `occurred_at` vs. `received_at` fields implement. This isn't an accident — it's the correct model for any system that ingests data from external platforms with their own clocks. We should document this explicitly in our architecture.

**Validity windows over deletion.** Graphiti never deletes superseded facts — it marks them invalid. Proply's stage decay is the same principle applied to computed state: stages don't reset to zero, they decay through defined transitions with history preserved in the activity ledger. The activity log is append-only for the same reason episodes are immutable in Graphiti.

**Incremental ingestion as a first-class requirement.** Graphiti's emphasis on immediate integration (no batch recomputation) validates our webhook-first architecture. Memory that can't be updated in real time isn't memory for agents — it's a cache.

**The episode as ground truth.** Graphiti stores raw ingested data as immutable episodes, and all derived facts trace back to them. Proply stores `raw_data` on every activity log entry for the same reason: the processed signal might be wrong, but the source payload is ground truth.

**Open the engine, close the execution.** Zep's split between Graphiti (open) and Zep (closed) is proof the playbook works. Developers adopt through the open layer because it's useful on its own. They pay for the hosted layer because scaling it is hard.

---

## 9. What Graphiti Doesn't Do (That We Need)

Graphiti is general-purpose by design. For GTM-specific memory, several things need to be added on top:

- **Intent classification** — knowing that a pricing page visit means something different than a homepage visit requires GTM domain knowledge that a general graph can't infer
- **Pipeline stage compute** — there's no concept of a contact moving through behavioral stages based on signal recency
- **Signal taxonomy** — the classification of signals by intent level and recency window is domain knowledge that has to be encoded, not discovered
- **ICP scoring** — scoring a contact's fit against a workspace's customer profile requires GTM context that lives outside the graph
- **Identity resolution for GTM** — resolving the same person across Instantly, RB2B, HubSpot, and LinkedIn requires platform-specific normalization logic

These aren't gaps in Graphiti — they're outside its scope by design. Graphiti solves the general memory problem. The GTM layer is what Proply adds on top.

---

## Key Takeaway

The most transferable idea from Graphiti: **treat memory as a timeline, not a state.** Don't overwrite facts — invalidate them. Don't delete history — timestamp it. Don't batch-process updates — integrate incrementally. These principles apply regardless of whether your memory structure is a knowledge graph or a typed signal ledger.

---

## Links

- GitHub: [github.com/getzep/graphiti](https://github.com/getzep/graphiti)
- Zep (hosted): [getzep.com](https://getzep.com)
- Docs: [help.getzep.com/graphiti](https://help.getzep.com/graphiti)

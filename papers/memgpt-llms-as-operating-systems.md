# MemGPT: Towards LLMs as Operating Systems

**Source:** UC Berkeley — [arxiv.org/abs/2310.08560](https://arxiv.org/abs/2310.08560)  
**Authors:** Charles Packer, Vivian Fang, Shishir G. Patil, Kevin Lin, Sarah Wooders, Joseph E. Gonzalez (2023)  
**Evolved into:** Letta — [letta.com](https://letta.com)  
**Code:** [github.com/cpacker/MemGPT](https://github.com/cpacker/MemGPT)

---

## Why This Is the Foundation Paper

Every system we've studied in this research repo — Graphiti's temporal facts, LightRAG's dual-level retrieval, the entity resolution pipeline, Context Rot's findings — is downstream of a single problem MemGPT named first: **LLM context windows are finite, but tasks are not.**

MemGPT is the foundational paper for agent memory. It's not the most recent, and it doesn't have the most impressive benchmarks. What it has is the right mental model — the one everything else builds on. Before reading anything else about memory for agents, read this.

---

## 1. The Problem: Fixed Context, Unbounded Tasks

Every LLM has a fixed context window. Everything the model can "see" — the conversation history, the task description, the retrieved facts, the tools available — must fit inside that window simultaneously. When it doesn't fit, older content gets dropped.

This creates a hard ceiling. An agent working on a long document analysis hits the ceiling. An agent in a multi-session conversation hits the ceiling. An agent maintaining relationships across months of interactions hits the ceiling.

The naive fixes all fail:
- **Summarize and discard** — you lose precision. Summaries compress detail.
- **Use a bigger context window** — Context Rot shows performance degrades as length increases regardless of window size.
- **Store everything in a vector DB and retrieve top-k** — you lose structure and recency awareness. Unfiltered retrieval surfaces irrelevant information.

MemGPT's contribution: don't try to fit everything in the window. Instead, treat the context window like RAM and external storage like disk — and give the agent the ability to move data between them.

---

## 2. The OS Analogy

Modern operating systems give applications the illusion of unlimited memory through **virtual memory** — data is moved between fast RAM (immediately accessible) and slow disk storage (retrievable on demand). The application doesn't need to manage this explicitly; the OS handles paging.

MemGPT applies this principle to LLMs:

- **Context window** = RAM (fast, immediately visible to the model)
- **External storage** = disk (slower, requires explicit retrieval)
- **Function calls** = the paging mechanism (the agent decides what to load and what to store)

The key shift: instead of the system passively managing what's in context, **the agent actively decides**. It uses tool calls to read from and write to external storage. It decides what's important enough to keep in the context window and what can be moved out.

---

## 3. The Four Memory Tiers

### Tier 1: Message Buffer (in-context)
The most recent conversation messages. This is the immediate short-term memory — what just happened. It's in the context window and visible to the model at all times.

When the buffer fills up, older messages are summarized and evicted into Recall Storage. The summary stays; the detail goes.

### Tier 2: Core Memory (in-context, persistent)
Editable, structured blocks pinned to the context window. Always present, always visible. Contains:
- Key facts about the user (preferences, name, context)
- Agent persona and objectives
- Critical task-specific state

This is the memory that should never be evicted. It's small, curated, and always current. The agent can update core memory using tool calls (`memory_replace`, `memory_rethink`) — it actively maintains what it considers most important.

**This is the minimum viable memory:** a small, structured, always-present block of the most relevant facts.

### Tier 3: Recall Storage (external)
The complete, append-only history of all past messages and interactions. Lives outside the context window in a searchable database. The agent retrieves from it using tool calls:
- `conversation_search(query)` — semantic search over past conversations
- `conversation_search_date(start, end)` — retrieve by time range

Nothing in recall storage is ever deleted. It's the ground truth record of everything that has happened.

### Tier 4: Archival Storage (external)
External knowledge — documents, files, long-form data — processed and stored in a vector database. Searched semantically. Accessed via:
- `archival_memory_insert(content)` — store new knowledge
- `archival_memory_search(query)` — retrieve relevant fragments

This is where you put information that's too large to live in core memory but too important to discard.

---

## 4. How Paging Works

When the message buffer approaches the context limit, MemGPT triggers an eviction cycle:

1. The oldest messages (e.g., 50% of the buffer) are taken out of context
2. The agent generates a recursive summary of the evicted messages, incorporating the existing summary if one exists
3. The evicted messages are stored in Recall Storage (permanent, searchable)
4. The summary stays in context, replacing the detailed messages

The result: the context window always contains recent detail + a compressed summary of older context. The full history is always retrievable, just not always in-context.

**The recursive summary is important.** Each eviction cycle the summary is rebuilt incorporating new evicted content. Over time it becomes denser but never loses the thread. This is how MemGPT maintains coherence across thousands of messages while keeping the context window focused.

---

## 5. Sleep-Time Agents

Letta (MemGPT's production descendant) introduced a concept not in the original paper worth noting: **sleep-time agents** — asynchronous processes that reorganize memory during idle periods.

While the main agent is between tasks, a background process runs over the accumulated memory and improves it:
- Merging redundant memories
- Updating stale facts
- Re-ranking what belongs in core memory vs. archival
- Building higher-level summaries from detailed interaction history

This is the equivalent of offline consolidation — the same process that happens in human memory during sleep. It's not real-time, and it doesn't need to be. Memory quality improves continuously without adding latency to active tasks.

---

## 6. Key Results

**Document analysis:** MemGPT successfully processed documents substantially exceeding the underlying model's context window, with higher accuracy than truncation-based approaches.

**Multi-session chat:** Agents maintained coherent relationships and remembered personal details across sessions that would have been entirely lost without the paging mechanism.

The benchmark numbers are less important than the demonstration: you can build agents that remember things across time without requiring an infinitely large context window.

---

## 7. The Hierarchy in One Diagram

```
┌─────────────────────────────────────────────┐
│            CONTEXT WINDOW (RAM)             │
│                                             │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │   Core Memory   │  │  Message Buffer  │  │
│  │  (always here)  │  │  (recent msgs)   │  │
│  └─────────────────┘  └──────────────────┘  │
│         ↑ agent edits        ↓ eviction      │
└─────────────────────────────────────────────┘
              ↕ tool calls (paging)
┌─────────────────────────────────────────────┐
│           EXTERNAL STORAGE (Disk)           │
│                                             │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │  Recall Storage │  │ Archival Storage │  │
│  │  (full history) │  │ (knowledge base) │  │
│  └─────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────┘
```

---

## 8. What This Changes for GTM Agent Memory

**The core memory block is the right abstraction for contact context.** When a GTM agent is about to act on a contact, it doesn't need the full history — it needs a small, curated, always-present summary of what matters right now. Pipeline stage, connection score, last activity, ICP fit. This is core memory. ~300 tokens. Always in context.

**Recall storage is the activity ledger.** Every signal that has ever arrived for a contact lives here. Not in context — but searchable when the agent needs to go deep. "What did we discuss in the meeting three weeks ago?" → search recall storage. Not injected upfront.

**Archival storage is company and market knowledge.** What we know about the contact's company, industry, funding, competitors — stored externally, retrieved when the agent needs context for a specific task.

**The paging problem is solved by pre-computation, not runtime retrieval.** MemGPT solves paging by having the agent decide at runtime what to load. Proply solves it by pre-computing the answer: stage is already computed, score is already computed, last activity is already structured. The agent doesn't need to page through history — the memory layer has already done that work and surfaced the result as a compact core memory block.

**The minimum viable memory for a GTM agent is a core memory block.** Stage, score, ICP fit, last activity, platform signals in last 30 days. Everything else is in recall storage, available if the agent asks. This is MemGPT's core memory principle applied to GTM: small, curated, always present, actively maintained.

---

## Key Takeaway

MemGPT's lasting contribution is the mental model: **treat agent memory as a hierarchy, not a flat context.** Some things are always in context (core memory). Some things are recent and in context (message buffer). Some things are retrievable (recall storage). Some things are deep background (archival). The agent actively manages what lives where using tool calls.

The corollary for memory layer design: your job is not to inject everything into the agent's context. It's to maintain the hierarchy — keep core memory small and current, recall history searchable and complete, archival knowledge deep and organized. The agent retrieves what it needs. The memory layer ensures it's there when asked.

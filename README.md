# proply-research

Welcome to proply-research. This is a public research log on **persistent memory for go-to-market AI agents**.

This repository serves as part of our process of building [Proply](https://goproply.com) — the persistent memory layer for GTM agents. Our goal is to be transparent about what we find works, what doesn't, and how we think about memory architectures as we build.

We hope this helps developers, builders, and researchers understand the importance of memory for AI agents operating in real go-to-market environments — where context isn't just a conversation, it's a relationship spanning months, dozens of signals, and a dozen different tools.

---

## What You'll Find Here

We continuously update this repository with:

**Signal & Memory Architecture**
- How GTM signals (website visits, email replies, LinkedIn messages, meetings) map to intent levels
- Identity resolution — how the same person across Instantly, RB2B, and LinkedIn resolves to one canonical record
- Pipeline stage compute — how behavioral signals drive stage transitions without human input
- Decay models — how memory goes stale and what to do about it

**Paper Summaries** — [`/papers`](./papers/)**

Concise breakdowns of research we found most relevant, written in the context of what it means for GTM agent memory:

| Paper | Our Take |
|---|---|
| [Context Rot](./papers/context-rot.md) — Chroma Research | Long context ≠ solved memory. Every model degrades as tokens increase — and this is why structured, pre-computed memory exists |
| [Graphiti: Temporal Knowledge Graphs](./papers/graphiti-temporal-knowledge-graphs.md) — Zep | Don't overwrite facts — invalidate them. The bi-temporal pattern and what it means for GTM agent memory |
| [Entity Resolution for Big Data](./papers/entity-resolution-big-data.md) — Academic Survey | The four-stage ER pipeline: blocking → matching → clustering. Why identity resolution is hard at scale and how to think about it |
| [Dedupe: Practical Entity Resolution](./papers/dedupe-practical-entity-resolution.md) — dedupeio | Active learning cuts labeling from thousands of examples to dozens. What production ER looks like in code |

We'll keep adding papers as we find ones that shape how we think.

**Implementation Notes**
Architecture sketches, design decisions, and lessons from building the actual system — what we tried, why it worked or didn't, and what we'd do differently.

---

## Why This Repo Exists

We believe the next generation of autonomous AI agents won't be defined by model size — they will be defined by the quality of their memory.

Agents today are capable of reasoning, planning, and taking action. What limits them is context: they forget conversations, lose track of relationships, and can't maintain continuity across sessions, tools, or time. Every time an agent starts a new task, it starts cold.

This is the problem Proply is built to solve — specifically for go-to-market: the place where relationship memory matters most, the stakes are high, and the signals are scattered across a dozen disconnected tools.

This repository documents our process as we:

- Study foundational research on memory architecture and context engineering
- Extract the core principles that apply to real GTM agent workflows
- Apply those insights to build Proply's persistent memory layer
- Share what we learn openly

We're not just building a product. We're working out what the right model for GTM agent memory even looks like — and sharing the thinking as we go.

---

## Implementation Notes

As we build Proply's memory layer, we'll share:

- **Signal taxonomy** — the canonical activity types, their sources, and their intent weights
- **Identity resolution** — the waterfall logic for matching signals to contacts across platforms
- **Stage compute** — how behavioral signals drive pipeline stages without human input
- **Decay logic** — how and when memory downgrades when signals go stale
- **Architecture sketches** — how the pieces fit together at the system level

For a reference implementation of the core pattern running on SQLite, see [proply-lite](https://github.com/proply-gtm/proply-lite).

---

## Stay Updated

- Product: [goproply.com](https://goproply.com)
- LinkedIn: [linkedin.com/in/documentbennet](https://linkedin.com/in/documentbennet)

---

## Contributing

If you're researching similar topics or want to collaborate:

- Open an issue
- Submit a pull request
- Reach out directly on LinkedIn

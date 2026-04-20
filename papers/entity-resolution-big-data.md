# Entity Resolution for Big Data: The Core Problem

**Source:** "End-to-End Entity Resolution for Big Data: A Survey" — Christophides, Efthymiou, Palpanas, Papadakis, Stefanidis (2020)  
**Paper:** [arxiv.org/abs/1905.06397](https://arxiv.org/abs/1905.06397)  
**Published in:** ACM Computing Surveys  
**Related reading:** [The Morning Paper summary](https://blog.acolyer.org/2020/12/14/entity-resolution/)

---

## Why We're Reading This

Every signal that enters Proply arrives with a name, an email, a LinkedIn URL, a company string — all from different platforms with different formatting conventions. Before any of that can be stored or queried, it has to be matched to a canonical contact. That matching problem has a formal name: **entity resolution**. This paper is the canonical academic treatment of how it works at scale.

---

## 1. What Entity Resolution Is

Entity resolution (ER) is the process of identifying records across data sources that refer to the same real-world entity.

It goes by many names depending on context:
- **Deduplication** — finding duplicates within a single source
- **Record linkage** — matching records across two sources  
- **Entity matching** — the general case, one or many sources

The problem exists because unique, stable identifiers are rare. In the real world:
- The same person is `Bennet Glinder` in HubSpot, `B. Glinder` in a CSV, and `bennet@proply.com` in Instantly
- The same company is `Proply Inc.`, `Proply`, and `PROPLY` depending on who filled out the form
- The same concept (`IBM`, `International Business Machines`, `Big Blue`) lives in different systems with different strings

Without entity resolution, every source produces a different version of the same entity. With it, you get one canonical record with all sources attached.

---

## 2. The ER Pipeline

Entity resolution at scale follows a four-stage pipeline:

### Stage 1: Blocking

**The problem:** Comparing every record to every other record is O(n²). At 100,000 contacts, that's 10 billion comparisons. Impossible in real time.

**The solution:** Blocking. Group records into buckets where matches are likely, then only compare within buckets. A record can belong to multiple buckets.

Common blocking keys:
- First 3 characters of last name
- Soundex/phonetic encoding of name
- Email domain
- Zip code + first name initial

The goal is to reduce comparisons by 99%+ while keeping the true match rate inside the same bucket close to 100%.

**The tradeoff:** Aggressive blocking = fast but misses some matches. Loose blocking = slower but more complete. Most production systems tune this empirically.

### Stage 2: Block Processing

Before running similarity comparisons on every pair in every block, filter out redundant comparisons. If record A was already compared to record B in Block 1, skip that pair in Block 2. This step alone can cut comparison volume in half.

### Stage 3: Matching

For each pair of records in a block, compute a similarity score. This is where the real complexity lives.

**Attribute comparison approaches:**
- **Exact match** — `bennet@proply.com == bennet@proply.com` → trivial but breaks on any variation
- **Edit distance (Levenshtein)** — how many character edits to transform one string into the other
- **Token-based similarity** — compare sets of words rather than characters (handles "Proply Inc" vs "Inc Proply")
- **Phonetic encoding** — `Smith` and `Smyth` map to the same phonetic key
- **Embedding similarity** — semantic closeness via vector distance (handles acronyms, aliases)

**The hard cases** (the ones your notes were describing):
- `IBM` vs `International Business Machines` — abbreviation/expansion
- `Bennet` vs `Ben` — nickname
- `Apple` (tech) vs `Apple` (fruit) — contextual disambiguation — same string, different entity
- `ABC Bakery` in a food context vs. `ABC Bakery` in a tech context — same name, different company

These cases can't be solved by string similarity alone. They require either world knowledge (embeddings trained on large corpora) or domain context (what other entities are mentioned nearby).

### Stage 4: Clustering

Once matching pairs are identified, group them into clusters where each cluster = one real-world entity.

**The inference problem:** If A matches B and B matches C, but A doesn't directly match C — are all three the same entity? Clustering algorithms make this call. Common approaches: connected components (conservative), hierarchical clustering with a similarity threshold.

**The conflict:** Clusters can grow too large (over-merging different entities) or too small (splitting one entity across multiple clusters). Production systems tune precision vs. recall here.

---

## 3. The Incremental Problem

The survey identifies incremental ER as one of the most underdeveloped areas. Most systems are designed for batch processing — you have a dataset, you run ER once, you get results.

Real-world agent systems need **streaming ER**: a new record arrives from a webhook and needs to be matched to an existing entity within milliseconds, not on the next batch run.

The challenge: when a new record arrives, you can't re-run the full pipeline. You need to:
1. Find candidate matches for the new record only
2. Merge or create without disturbing existing resolved entities
3. Handle the case where the new record causes two previously separate entities to merge

This is the hardest operational problem in ER at scale, and it's where most open-source tools fall short.

---

## 4. Top-Down vs. Bottom-Up

The survey frames two fundamental approaches to the schema question:

**Top-down (schema-first)**
Define your entity types and attributes before data arrives. Incoming records are mapped to a known schema. Matching functions are tuned per attribute type.

- Pros: consistent, fast, queryable, tunable
- Cons: misses entities or attributes you didn't anticipate; requires domain knowledge upfront
- Example: Proply's approach — `Contact` schema is defined, all signals map to it

**Bottom-up (data-first)**
Let entity types emerge from the data. Don't predefine what entities look like — discover them.

- Pros: flexible, handles unexpected structures
- Cons: expensive (requires LLM or ML at ingestion time), inconsistent naming, harder to query
- Example: Graphiti's emergent ontology mode — entities and relationships discovered from raw text

**The hybrid:** Define core entity types (top-down) but allow emergent attributes within those types (bottom-up). This is what production systems at scale tend to converge on — Graphiti's custom type support is an example of offering both.

---

## 5. Why ER Is Fundamentally Hard

**Ambiguity is irreducible.** Some cases cannot be resolved without external knowledge. Is `J. Smith at Acme` the same as `John Smith at Acme Corp`? Maybe. Is it the same as `Jane Smith at Acme`? Probably not. No algorithm resolves this with certainty — it's a probabilistic judgment.

**Scale changes everything.** A 99% accurate ER system on 1 million records produces 10,000 errors. At 100 million records, that's 1 million errors. Error rates that are acceptable in small systems become major data quality problems at scale.

**The quadratic baseline.** Without blocking, comparison complexity is O(n²). With blocking, it's approximately O(n log n) if done well. Most of the engineering in large-scale ER systems is in making blocking smarter.

---

## 6. What This Means for Proply

**The identity resolution waterfall is a blocking strategy.** Matching by external ID first, then email, then LinkedIn URL is a form of blocking — you're reducing the comparison space by checking the most discriminative attribute first. Exact-match-first is the right order: it's fast, deterministic, and covers the majority of cases.

**The hard cases need semantic matching.** Your waterfall handles exact matches well. What it doesn't handle: `Ben Glinder` arriving from one source and `Bennet Glinder` from another with no email overlap. At current scale this is fine to handle manually. At 100k+ contacts, you need fuzzy name matching (edit distance + phonetic encoding) as a Stage 4 in the waterfall before the "create new contact" fallback.

**Conflict resolution needs an explicit policy.** When HubSpot says `company = "Proply Inc"` and LinkedIn says `company = "Proply"` — which wins? Right now the answer is "first one in, don't overwrite." That's a valid policy at early scale. Explicitly naming it as a **trust ranking** (which source wins on which field) is the production-grade approach.

**Incremental ER is the real problem.** The survey's framing of incremental ER as underdeveloped is directly relevant — every webhook Proply processes is an incremental ER problem. The current waterfall handles it correctly for the happy path. The edge cases (merging two previously separate contacts when a new signal connects them) are where the hard work lives.

---

## Key Takeaway

Entity resolution is a four-stage pipeline: block → filter → match → cluster. The fundamental challenge is the tension between scale (you can't compare everything to everything) and completeness (you can't miss any match). Every production ER system is a set of tradeoffs along that spectrum. The right tradeoffs depend on the domain — for GTM, false negatives (missing a match = duplicate contact) are more expensive than false positives (merging two different people = data corruption), which means tuning toward higher recall at the cost of some precision.

# Context Rot: How Increasing Input Tokens Impacts LLM Performance

**Source:** Chroma Research — [trychroma.com/research/context-rot](https://www.trychroma.com/research/context-rot)  
**Authors:** Kelly Hong, Anton Troynikov, Jeff Huber (2025)  
**Code:** [github.com/chroma-core/context-rot](https://github.com/chroma-core/context-rot)

---

## Why We're Reading This

Building memory for GTM agents means solving a concrete retrieval problem: given a contact's full signal history, surface the right context at the right moment. The common assumption is that larger context windows make this easier. This paper is systematic evidence that they don't — and it changes how we think about what a memory layer actually needs to do before an agent ever sees data.

---

## 1. Background & Core Finding

### The Problem With the Standard Benchmark

Long-context performance has been measured with **NIAH (Needle in a Haystack)**: hide a fact inside a long document, ask the model to find it. Models score near-perfect, so the consensus became "long context is solved."

The flaw: NIAH depends on **lexical matching** — the question and the answer share the same keywords, so the model finds the answer by pattern-matching rather than understanding. Chroma found that **72.4% of real retrieval tasks require semantic inference** — the question and answer use different phrasing for the same concept. NIAH doesn't test that. It doesn't test filtering similar-but-wrong information either. It doesn't test finding blended information in a noisy context.

So Chroma built a harder benchmark and tested 18 models on it.

### Context Rot

The finding: **performance degrades as input tokens increase — universally, across all 18 models, including on simple tasks.**

They named this **Context Rot**.

- **Universal:** Every model tested showed the same degradation pattern
- **Not complexity-dependent:** Simple replication tasks degrade too, not just complex reasoning
- **Model-specific curves:** Claude, GPT, Gemini, and Qwen each degrade differently — but all degrade

**Models tested:**

| Provider | Models |
|---|---|
| Anthropic | Claude Opus 4, Sonnet 4, Sonnet 3.7, Sonnet 3.5, Haiku 3.5 |
| OpenAI | o3, GPT-4.1, GPT-4.1 mini, GPT-4.1 nano, GPT-4o, GPT-4 Turbo, GPT-3.5 Turbo |
| Google | Gemini 2.5 Pro, Gemini 2.5 Flash, Gemini 2.0 Flash |
| Alibaba | Qwen3-235B-A22B, Qwen3-32B, Qwen3-8B |

---

## 2. Key Concepts

| Term | Definition |
|---|---|
| **Needle** | Target information hidden in a long document — the fact the model needs to retrieve |
| **Haystack** | The surrounding long-form text (10k–100k+ tokens) |
| **Distractor** | Similar-but-incorrect information mixed into the haystack |
| **Semantic similarity** | Meaning-based similarity via embedding cosine score (0–1) |
| **Lexical matching** | Keyword-based matching without semantic understanding |
| **Context Rot** | The universal performance degradation as input token count increases |

---

## 3. Experiments

### 3.1 Needle-Question Similarity

**Question:** Does semantic similarity between the question and the answer affect how well models retrieve it in long context?

**Setup:** Question–answer pairs were collected from Paul Graham essays and arXiv papers and grouped by embedding similarity: High (0.7–0.8) vs Low (0.4–0.5). The answer was hidden inside long documents at 11 different positions and models were asked to retrieve it at increasing context lengths.

**Findings:**
- Short context (~1k tokens): Models retrieve correctly regardless of similarity level
- Long context (10k+ tokens): High-similarity answers still retrieved accurately; Low-similarity answers show sharp accuracy collapse
- Needle position was also tested across 11 locations within the haystack

**Why it matters for GTM memory:** An agent asking "has this contact shown buying intent?" won't find a meeting transcript that says "they seemed ready to move forward" unless semantic bridging happens before retrieval. Keyword-based or embedding-naive retrieval breaks at scale. The memory layer has to do the semantic work — not the agent.

---

### 3.2 Distractor Impact

**Question:** How much does similar-but-wrong information degrade performance — and does it compound?

**Setup:** Four distractors were created for each needle, each varying one dimension while keeping the rest similar:

| Distractor | Change Made |
|---|---|
| #1 | Source change — "from my professor" → "from my classmate" |
| #2 | Sentiment change — "best advice" → "worst advice" |
| #3 | Time change — "in college" → "back in high school" |
| #4 | Certainty change — "I think the best…" → "I thought the best…but not anymore" |

Three conditions were tested: baseline (needle only), single distractor (needle + 1), and multiple (needle + all 4).

**Findings:**
- Even 1 distractor causes measurable performance drop from baseline
- 4 distractors cause compounded degradation
- **Distractor #3 (time change) caused the largest drop** — models consistently confused temporally shifted versions of the same fact
- **Distractor #2 (sentiment change) was second** — models often output the negative version as correct
- **Model divergence:** Claude abstains when uncertain (lowest hallucination); GPT answers confidently even when wrong (highest hallucination)

**Why it matters for GTM memory:** A contact who visited pricing three times and another who visited the homepage once produce structurally similar signals. Without a clear taxonomy and intent weighting, a model retrieves the wrong contact or the wrong signal. Temporal distractor impact is particularly relevant — "they replied last week" vs "they replied last month" are exactly the confusions that break pipeline stage reasoning.

---

### 3.3 Needle-Haystack Similarity

**Question:** Does the answer "blend in" when it matches the surrounding text? Does a different-topic answer "stand out" and become easier to find?

**Setup:** Four combinations were tested using two haystack types (Paul Graham essays vs arXiv papers) and two needle types (writing/startup vs academic/technical). Each combination has a measurable semantic similarity score between the needle and the surrounding text.

| Combination | Similarity Score |
|---|---|
| PG haystack + PG needle | 0.529 (high) |
| PG haystack + arXiv needle | 0.368 (low) |
| arXiv haystack + arXiv needle | 0.654 (high) |
| arXiv haystack + PG needle | 0.394 (low) |

**Findings:**
- In PG haystack: the arXiv needle (low similarity) performed **better** than the PG needle — it stood out against the background
- In arXiv haystack: performance was nearly identical regardless of needle type
- The relationship is not uniform — lower similarity sometimes helps, sometimes doesn't, depending on domain and haystack structure

**Why it matters for GTM memory:** Contact memory should be structurally distinct from whatever else is in the agent's context. If you inject a contact's history as natural language prose, it blends with any other text. Structured, typed fields stand out. A contact object with explicit keys (`pipeline_stage: evaluating`, `last_signal: meeting_held`) retrieves more reliably than a paragraph about the same person.

---

### 3.4 Haystack Structure: Coherent vs. Shuffled

**Question:** Is organized, logically structured text easier to search than shuffled text?

**Setup:** Two versions of the same haystack were created: original (paragraphs in logical order) and shuffled (same sentences randomly rearranged). The same needle was inserted into both. Accuracy was compared across all 18 models.

**Findings (paradoxical):**
- All 18 models performed **better on shuffled text** than organized text
- The pattern was consistent across all needle-haystack combinations
- Effect became more pronounced as context length increased

**Hypotheses for why:**
- Coherent logical flow disperses model attention across multiple related sentences
- Shuffled sentences are processed more independently, making the relevant one stand out
- Structural coherence may create a kind of "blending" effect similar to high needle-haystack similarity

**Why it matters for GTM memory:** Don't pass contact summaries as narrative prose. Pass structured, independent facts. Each signal as its own entry. The model handles a list of disconnected typed facts better than a well-written paragraph describing the same information.

---

### 3.5 Repeated Words Task

**Question:** Does even the simplest possible task — exact replication — degrade with longer input?

**Setup:** Models were given a repeating word sequence with one unique word inserted (e.g., "apple apple apple BANANA apple apple...") and instructed to replicate it exactly. Tested across 12 length steps from 25 to 10,000 words, with 7 word combinations varying by case, plurality, and word distance. Accuracy was measured using Levenshtein distance.

**Findings:**
- All 18 models showed consistent length ↑ → accuracy ↓
- Unique word near the front of the sequence → higher accuracy, especially at long lengths
- At 10,000 words, models began generating words not present in the input at all
- **Model-specific behaviors:**

| Model | Notable behavior |
|---|---|
| Claude 3.5 Sonnet | Higher accuracy than newer Claude models up to 8,192 tokens |
| Claude Opus 4 | Slowest degradation curve — but 2.89% task refusal |
| GPT-4.1 | 2.55% task refusal around 2,500 words |
| Gemini | Starts generating random output at 500–750 words |

**Why it matters for GTM memory:** Context rot isn't about task complexity. It happens on the simplest possible tasks, just by making the sequence longer. The solution isn't a smarter model — it's keeping the injected context short, focused, and front-loaded with the most relevant facts.

---

### 3.6 LongMemEval: Realistic Conversation History

**Question:** When the answer exists somewhere in a 110k-token conversation history, can models reliably retrieve it? How much does irrelevant context hurt?

**Setup:** 306 questions were extracted from the LongMemEval_s benchmark and tested under two conditions:
- **Full (~113k tokens):** Entire conversation history (mostly irrelevant to the question)
- **Focused (~300 tokens):** Only the information needed to answer

**Question types tested:**

| Type | Description |
|---|---|
| Knowledge Update | "First said A, later changed to B — what's the current answer?" |
| Temporal Reasoning | "Did X first, then Y — what's the order?" |
| Multi-session | "Combining conclusions from last week and this week's conversations" |

**Findings:**
- Focused condition: all models score high
- Full condition: significant performance drop across all models
- **Claude:** Largest gap between Focused and Full — abstains ("cannot find answer") more in full context, driving lower scores but also lowest hallucination
- **GPT, Gemini, Qwen:** Also degrade in Full, but smaller gap than Claude
- **Hardest question types in Full context:** Temporal Reasoning and Multi-session (non-thinking mode); Multi-session (thinking mode)

**Why it matters for GTM memory:** These three question types are exactly what GTM agents need to answer. "When did they last engage?" = temporal reasoning. "Have they gone cold since the meeting?" = knowledge update. "What did we discuss last quarter vs. last week?" = multi-session. None of these are answerable by dumping a contact's history into a 100k-token context. The memory layer has to pre-compute answers to these questions and serve structured facts — not raw history.

---

## 4. Model Behavior Summary

| Model | Hallucination | Under Uncertainty | Notable |
|---|---|---|---|
| **Claude** | Lowest | States "cannot find answer" — abstains | Opus 4: 2.89% task refusal |
| **GPT** | Highest | Answers confidently when wrong | GPT-3.5: 60% excluded by content filter in testing |
| **Gemini** | Medium | Generates random output | Degrades earliest (500–750 words) |
| **Qwen** | Medium | Doesn't attempt task | Random output after 5,000 words |

---

## 5. Methodology

- 8 input lengths × 11 needle positions per experiment
- Temperature = 0 for deterministic output (model-specific exceptions noted)
- Qwen models use YaRN extension: 32k → 131k token range
- Evaluation: LLM judge calibrated to 99%+ agreement with human labeling
- Manual verification: ~500 NIAH, ~600 LongMemEval samples
- Refusal rate across 194,480 API calls: 0.035% (69 refusals)

---

## 6. Practical Guidelines

### Core Principle

> Information existing in context ≠ performance guaranteed.  
> Where and how it's placed determines performance.

| Recommendation | Reason |
|---|---|
| Place important info early | Earlier position = higher retrieval accuracy |
| Minimize distractors | Even 1 similar-but-wrong entry causes drop |
| Keep context short and focused | All models degrade linearly with length |
| Use structured fields over prose | Coherent narrative blends in; typed fields stand out |
| Pre-compute temporal facts | Temporal reasoning degrades fastest in long context |

### Model Selection

| Priority | Use | Reason |
|---|---|---|
| Low hallucination | Claude | Abstains when uncertain; doesn't confabulate |
| Uncertainty signaling needed | Claude | Explicitly returns "no answer found" |
| Long context unavoidable | All degrade | Compensate with context engineering before retrieval |

---

## 7. Limitations

- Research focused on relatively simple information retrieval tasks — real-world agents require multi-step synthesis, which likely degrades faster
- Why performance drops in logically organized text remains unexplained
- How attention mechanisms respond to structural coherence isn't fully characterized
- Task complexity and context length effects haven't been fully separated

---

## What This Changes for Us

Context rot is the argument for why a memory layer has to exist at all. If long context worked, you could feed everything in and let the model sort it out. It doesn't — and the failure modes are exactly the ones that matter most for GTM:

- Temporal confusion (time-shifted distractors cause the largest accuracy drops)
- Multi-session reasoning (performance collapses in full context)
- Knowledge updates (current state of a relationship vs. historical state)

For Proply's memory layer:

- **Don't inject raw signal history.** Pre-compute stage, connection score, and last-N typed activities. Serve structured facts.
- **Keep injected context tight.** ~300 focused tokens outperforms 113k unfocused tokens.
- **Classify and filter signals before retrieval.** Distractors compound. Unweighted signal history is a distractor factory.
- **Front-load the most relevant facts.** Earlier position in context = higher retrieval accuracy.
- **Pre-compute temporal and update questions.** "Are they still evaluating?" shouldn't require the agent to scan 60 days of history. The answer should be stored, computed, and ready.

---

## Citation

```bibtex
@techreport{hong2025context,
  title   = {Context Rot: How Increasing Input Tokens Impacts LLM Performance},
  author  = {Hong, Kelly and Troynikov, Anton and Huber, Jeff},
  year    = {2025},
  month   = {July},
  institution = {Chroma}
}
```

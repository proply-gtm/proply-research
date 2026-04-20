# Reflexion: Language Agents with Verbal Reinforcement Learning

**Source:** Northeastern University, MIT — [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366)  
**Authors:** Noah Shinn, Federico Cassano, Ashwin Gopinath, Karthik Narasimhan, Shunyu Yao (2023)  
**Published at:** NeurIPS 2023  
**Code:** [github.com/noahshinn/reflexion](https://github.com/noahshinn/reflexion)

---

## Why This Paper Matters

MemGPT answers "how does an agent remember?" Reflexion answers "how does an agent *learn* from what it remembers?"

The distinction is important. A memory layer that only stores and retrieves is a database. A memory layer that enables an agent to improve its behavior based on past outcomes is something fundamentally more useful. Reflexion is the framework that formalizes this — not through model retraining, not through fine-tuning, but through language itself as the feedback mechanism.

For GTM, the implication is direct: an agent that sent a proposal to a contact who wasn't ready and lost the deal should remember that. An agent that waited for the meeting to happen before sending and won the deal should remember that too. Reflexion is how agents learn from outcomes without requiring a data scientist.

---

## 1. The Problem: Agents That Can't Learn From Mistakes

Standard LLM agents have no feedback loop. They act, observe the result, and start fresh on the next task with no memory of what worked and what didn't. They make the same mistakes repeatedly because nothing carries forward.

Traditional reinforcement learning fixes this through weight updates — the model's parameters change based on rewards. But this requires:
- Thousands of training examples
- Expensive GPU compute
- Days of training time
- A frozen model afterward

For deployed agents acting in live systems, none of these are practical. You can't retrain Claude every time a prospect doesn't reply to a follow-up.

Reflexion's answer: use language as the reinforcement signal instead of gradient updates. The agent reflects on what happened, writes that reflection in plain text, and stores it in memory. On the next attempt, it reads its own past reflections before acting. No retraining. No fine-tuning. No compute cost beyond inference.

---

## 2. The Architecture: Three Components

### The Actor
The LLM that generates text and actions. It operates on a combination of:
- The current environment state (what's happening right now)
- Short-term memory (the current trajectory — recent actions and observations)
- Long-term memory (stored reflections from previous attempts)

The Actor is the decision-maker. It sees the task, it sees what it's tried before, and it acts.

### The Evaluator
Scores the Actor's output. The evaluation signal can be:
- **Scalar reward** — binary pass/fail (did the code compile? did the test pass?)
- **Heuristic** — rule-based judgment (did the agent reach the goal state?)
- **LLM-as-judge** — another model evaluates the quality of the output

The Evaluator answers: how did we do?

### The Self-Reflection Model
Takes the trajectory (what the actor did) and the evaluation score (how it went) and generates a **verbal reflection**: a natural language summary of what went wrong, why, and what to do differently.

This reflection is stored in the long-term memory buffer. On the next attempt, the Actor reads it before acting.

The key insight: **language is rich enough to carry nuanced lessons.** "I failed because I checked the wrong room first" is more informative than a scalar reward of -1. The agent can act on a sentence in a way it cannot act on a number.

---

## 3. The Memory System

### Short-term memory (current trajectory)
The immediate context: the current task description, the actions taken so far in this attempt, and the observations received. This is the in-context scratchpad for the current trial. Granular, detailed, ephemeral.

### Long-term memory (episodic reflection buffer)
A sliding window of the last 1–3 reflections from previous attempts. Each reflection is a paragraph of natural language summarizing what went wrong and what to try differently.

**Why a sliding window of 1–3, not unlimited?** Context window constraints. Storing all past reflections would eventually overflow the context. Keeping only the most recent 1–3 reflections is enough — the most recent failures are the most relevant, and older reflections are already incorporated into improved behavior.

This is a direct application of the MemGPT principle: the context window is finite, so memory must be curated. Reflexion's curation strategy is simple and effective: keep the most recent reflection, discard older ones.

---

## 4. The Feedback Loop

```
Task given to Actor
  ↓
Actor acts (using current trajectory + past reflections)
  ↓
Evaluator scores the result
  ↓
If score < threshold:
  Self-Reflection model generates verbal reflection
  Reflection stored in long-term memory buffer
  Actor tries again, now reading its own reflection
  ↓
Repeat until success or max trials
```

On each iteration, the Actor starts with better context than the previous attempt. It doesn't start from zero — it starts from "here's what I tried, here's why it didn't work, here's what I should try instead."

---

## 5. What Gets Written in a Reflection

The verbal reflections aren't just "that was wrong." They're specific, actionable, and causal. From the paper:

> "I was not successful in this task. I attempted to look in the wrong room first... I should try looking in the bathroom cabinet first next time."

> "My test passed but I missed the edge case where the input is empty. I need to add a check for empty input before processing."

> "I answered the question correctly but from the wrong source. I should have checked the Wikipedia article, not the news article."

Each reflection contains three things: **what happened**, **why it failed**, and **what to do differently**. This structure is what makes the feedback useful. A scalar reward tells you nothing about the why. A verbal reflection tells you everything.

---

## 6. Key Results

**Sequential decision-making (AlfWorld — household tasks):**
- Reflexion agents improved by 22% absolute over baselines
- Most improvement came in the first 3–4 learning iterations
- Agents learned room-specific navigation strategies through reflection

**Reasoning (HotPotQA — multi-hop question answering):**
- 20% improvement over Chain-of-Thought baseline
- The notable result: improvements held even when the CoT baseline had access to ground truth context

**Programming (HumanEval / MBPP — code generation):**
- **91% pass@1 accuracy on HumanEval**, surpassing GPT-4's 80%
- Agents generated their own test cases and used failed tests as feedback signals
- This result is significant: the agent outperformed a larger, more capable model by learning from its own mistakes

---

## 7. The Limits

**Reflexion requires multiple attempts.** It's a trial-and-error system. For tasks where you only get one shot — a live email, a proposal that's already sent — Reflexion's iterative improvement doesn't apply directly.

**Reflections can be wrong.** If the Self-Reflection model misdiagnoses the failure, the next attempt might be worse. The quality of the reflection is bounded by the quality of the LLM generating it.

**The sliding window loses old lessons.** A lesson from 10 attempts ago is gone. This is a practical constraint, not a fundamental flaw — but it means Reflexion agents don't accumulate unlimited learning.

**No cross-agent memory sharing.** Reflections are per-agent, per-task. If one agent learns something, a different agent working on the same class of tasks doesn't benefit. This is the multi-agent memory problem that Reflexion doesn't solve.

---

## 8. What This Changes for GTM Agent Memory

**Agents should reflect on outcomes, not just record them.** Proply's activity ledger records what happened: proposal sent, meeting held, deal won. Reflexion argues that recording is not enough — the agent should also record *why* it worked or didn't and what it should do differently. This is a richer form of memory than a typed activity log.

**The sales outcome is the evaluation signal.** Reflexion's Evaluator scores each attempt. In GTM, the evaluation signals are natural and already present: a proposal was signed (success), a contact went cold after a meeting (failure), a sequence got no replies (failure), a warm lead converted after a personalized follow-up (success). These are the reward signals that should trigger reflection.

**The minimum viable GTM reflection:**
When a contact reaches `client`, the agent should generate: "What combination of signals, timing, and actions preceded this outcome?" When a contact drops from `evaluating` to `aware`, the agent should generate: "What happened? Did we wait too long? Did we send the wrong message type?"

These reflections, stored against the workspace (not the contact), become an accumulating body of learned GTM judgment. Over time, they inform every future agent action — not through retraining, but through memory.

**The cross-agent problem is the hardest part.** Reflexion's limitation — reflections are per-agent, per-task — is exactly the limitation of most AI tooling in GTM. What one agent learns from one deal doesn't help the agent working the next deal. A shared memory layer that accumulates workspace-level reflections — what message types work for this ICP, what timing patterns precede conversion, what signals reliably predict churn — is the GTM equivalent of solving Reflexion's cross-agent limitation.

**This is what Proply's workspace memory is for.** Not just contact history. Learned judgment about what works in this specific sales motion, for this specific ICP, accumulated from every outcome across the workspace.

---

## Key Takeaway

Reflexion proves that agents can improve their behavior through language-based feedback without retraining. The mechanism is simple: act, evaluate, reflect, store the reflection, read it next time. The verbal reflection — specific, causal, actionable — is richer than any scalar reward signal.

The GTM implication: recording what happened is not enough. Memory that enables learning from outcomes is qualitatively different from memory that only records events. The distinction between a contact history log and a workspace-level reflection on what works is the difference between a database and a memory layer that makes agents smarter over time.

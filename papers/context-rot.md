# Context Rot: How Increasing Input Tokens Impacts LLM Performance

**Source:** Chroma Research — [trychroma.com/research/context-rot](https://www.trychroma.com/research/context-rot)  
**Authors:** Kelly Hong, Anton Troynikov, Jeff Huber (2025)  
**Code:** [github.com/chroma-core/context-rot](https://github.com/chroma-core/context-rot)

---

## Why We're Summarizing This

Building a memory layer for GTM agents means solving a specific problem: an agent needs to retrieve the right relationship context at the right moment, across potentially hundreds of contacts and thousands of signals. We assumed that larger context windows made this easier. This paper is evidence that they don't — and it changes how we think about what a memory layer actually needs to do.

---

## The Core Finding: Context Rot

Performance degrades as input tokens increase — and it affects every model, on even simple tasks.

Chroma tested 18 LLMs across Anthropic, OpenAI, Google, and Alibaba. Every single one showed the same pattern: accuracy drops as context gets longer. They named it **Context Rot**.

This matters for us because the dominant assumption in the agent ecosystem is that long context = solved memory. It isn't. Stuffing a contact's full history into a prompt and hoping the model finds what's relevant is not a memory strategy — it's context rot in slow motion.

---

## Why the Standard Benchmark Was Wrong

The field has been measuring long-context performance with **NIAH (Needle in a Haystack)** — hide a fact in a long document, ask the model to find it. Models score near-perfect, so "long context is solved."

The problem: NIAH relies on **lexical matching**. The question and the answer share keywords, so the model can find it by pattern-matching rather than understanding. Chroma found that **72.4% of real retrieval tasks require semantic inference** — the question and answer use different phrasing for the same meaning.

When you test semantic retrieval instead of keyword matching, performance collapses at scale.

---

## The Experiments (What Actually Matters for Agent Memory)

### 1. Semantic Gap Between Question and Answer

When a question and its answer use similar words, models find it even in long context. When they use different phrasing for the same meaning ("writing advice" vs "composition tips"), accuracy drops sharply past 10k tokens.

**GTM implication:** An agent asking "has this contact shown buying intent?" won't find a meeting transcript that says "they seemed ready to move forward" unless the memory layer does the semantic bridging first. Raw retrieval from long context won't surface it reliably.

### 2. Distractors Compound Degradation

Adding similar-but-wrong information makes models worse — and the effect compounds with each distractor. Even a single piece of similar-but-incorrect information causes a measurable drop in accuracy.

The most damaging distractor type: **time change** ("wrote last year" vs "wrote last week"). The second most: **sentiment change** ("best advice" vs "worst advice").

**GTM implication:** A contact who visited your pricing page three times and a different contact who visited your homepage once will produce similar-looking signals. Without a clear signal taxonomy and intent weighting, the model retrieves the wrong one. This is why we classify signals by type and weight rather than treating all activity equally.

### 3. Needle Standing Out vs. Blending In

When the target information is semantically different from the surrounding text, it actually stands out and is easier to find. When it blends in (same topic as the haystack), accuracy drops.

**GTM implication:** Contact memory should be structured, not narrative. If you inject a contact's full email thread as prose, it blends into any other text around it. If you inject it as structured fields (stage, last signal type, last activity summary), it stands out and retrieves reliably.

### 4. Shuffled Text Outperforms Organized Text (Paradox)

Across all 18 models, randomly shuffled sentences produced better retrieval than logically organized paragraphs. The hypothesis: coherent logical flow disperses attention, while isolated sentences are processed independently and the relevant one stands out.

**GTM implication:** Don't pass a beautifully written contact summary to an agent. Pass structured, independent facts. Each signal as its own entry. The model handles disconnected facts better than flowing narrative.

### 5. Simple Replication Degrades at Scale

Models asked to replicate a simple word sequence ("apple apple apple BANANA apple...") failed increasingly as sequence length increased — the same word-repetition task, just longer. Performance degraded linearly. At 10,000 words, models started generating words not even in the input.

**GTM implication:** Context rot isn't about complexity. It happens on the simplest possible tasks. The solution isn't a smarter model — it's keeping the context short and focused.

### 6. Focused vs. Full Context (LongMemEval)

Chroma tested models on questions from a 110k-token conversation history under two conditions: full context (113k tokens, mostly irrelevant) vs. focused context (~300 tokens, just the relevant part). Every model scored dramatically higher with focused context.

The hardest question types in full context:
- **Temporal reasoning** — "What happened first?"
- **Multi-session synthesis** — "Combining what you said last week and this week..."
- **Knowledge updates** — "You first said X, then changed to Y — what's current?"

**GTM implication:** These are exactly the questions a GTM agent needs to answer about a contact. "When did they last engage?" "Have they gone cold since the last meeting?" "Did their company raise since we last spoke?" None of these are answerable by dumping a contact's full history into context. The memory layer has to pre-compute answers to these questions and surface structured facts.

---

## Model Behavior: What We Learned for Agent Design

| Model | Hallucination | When Uncertain |
|---|---|---|
| Claude | Lowest | States "cannot find answer" — abstains |
| GPT | Highest | Answers confidently when wrong |
| Gemini | Medium | Starts generating random output |
| Qwen | Medium | Doesn't attempt task at long lengths |

Claude's abstention behavior is relevant for reliability-critical agents (don't want a follow-up agent confidently acting on the wrong contact stage). GPT's hallucination pattern matters when using it for context-heavy retrieval tasks.

---

## What This Changes for Us

The conclusion from this paper isn't "use a better model." It's that **context engineering is the work** — and that work has to happen before the model ever sees the data.

For Proply's memory layer, this means:

- **Don't pass raw signal history to agents.** Pre-compute stage, score, and last-N relevant activities. Pass structured facts, not timelines.
- **Keep injected context short and focused.** A contact's memory context should be ~300 tokens of structured, relevant facts — not a dump of every webhook event.
- **Separate signal types.** Distractors kill accuracy. Signals need to be classified, weighted, and filtered before being surfaced to an agent.
- **Temporal and knowledge-update questions need pre-computation.** "Are they still in evaluating stage?" shouldn't require the agent to scan 60 days of history. The stage should be stored, computed, and ready to serve.

Context rot is the reason a memory layer needs to exist at all. If long context worked, you could just feed everything in. It doesn't — which is exactly why structured, pre-computed, focused memory is the architecture.

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

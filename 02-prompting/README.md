# Phase 02 — LLM Fundamentals and Prompt Engineering

Detailed checklist for understanding how large language models actually work under the hood, and how to talk to them reliably — transformers and attention, tokens and context windows, embeddings, hallucination, and the prompting techniques that turn a probabilistic model into a dependable product feature.

## Checklist

- [ ] [LLM Fundamentals: Transformers, Tokens & Context](#llm-fundamentals-transformers-tokens--context)
- [ ] [Embeddings & Semantic Meaning](#embeddings--semantic-meaning)
- [ ] [Prompt Engineering Techniques](#prompt-engineering-techniques)
- [ ] [Reliability, Hallucination & Structured Outputs](#reliability-hallucination--structured-outputs)

<a id="llm-fundamentals-transformers-tokens--context"></a>

## LLM Fundamentals: Transformers, Tokens & Context

Before writing a single prompt, you need a working mental model of what's actually happening between your text going in and a response coming out.

### What is an LLM?

- **Large Language Model** — a neural network trained on massive amounts of text to predict the next token given everything that came before it. Every response you get, from a one-word answer to a full essay, is built one token at a time from that same prediction.
- **Pre-training vs fine-tuning** — pre-training teaches the model general language and world knowledge from broad text corpora; fine-tuning (including instruction-tuning and RLHF) shapes that raw model into one that follows instructions and behaves like an assistant.
- **Why this matters for engineering** — an LLM is not a database and not a deterministic function. It's a statistical next-token predictor, so every reliability technique you'll use later (structured outputs, RAG, evals) exists to compensate for that.

### Transformers and Attention

- **Transformer architecture** — the neural network design (introduced in "Attention Is All You Need," 2017) that underlies essentially every modern LLM. It processes an entire sequence of tokens in parallel rather than one at a time like older RNNs.
- **Self-attention** — for each token, the model computes how much every other token in the input should "matter" when interpreting it. This is how a model resolves something like a pronoun referring back to a noun several sentences earlier.
- **Why it matters** — attention is why LLMs are good at long-range context (a fact mentioned early in a long prompt can still influence output near the end) but also why longer inputs cost more compute — attention cost grows with sequence length.

```text
Input:  "The trophy didn't fit in the suitcase because it was too small."
                                                          ^^
Self-attention lets the model weigh "it" against both "trophy" and
"suitcase" and resolve the reference correctly using surrounding context.
```

### Tokens and Tokenization

- **Tokens, not words** — models don't read words; they read tokens, sub-word chunks produced by a tokenizer (e.g., byte-pair encoding). A single word can be one token or split into several ("hamburger" might become "ham" + "burger").
- **Why tokens matter practically** — API pricing is per token (input + output), rate limits are measured in tokens, and prompt length limits are token limits, not character or word limits. Roughly 1 token ≈ 4 characters of English text.
- **Tokenizer tools** — providers publish tokenizer playgrounds/libraries (e.g., `tiktoken` for OpenAI) so you can count tokens before sending a request instead of guessing.

```ts
// Rough token-budget check before sending a request — never assume, count.
import { encode } from "gpt-tokenizer";

const prompt = "Summarize this support ticket in two sentences: ...";
const tokenCount = encode(prompt).length;

if (tokenCount > MODEL_CONTEXT_LIMIT - RESERVED_FOR_RESPONSE) {
  throw new Error("Prompt exceeds available context budget");
}
```

### Context Windows

- **Context window** — the maximum number of tokens (input + output combined) a model can "see" in a single request. Anything outside that window simply doesn't exist to the model — it's not truncated gracefully, it's just gone.
- **Context window ≠ memory** — an LLM has no persistent memory between API calls. What looks like "the model remembers our conversation" is really your application resending the full conversation history as part of the prompt every time.
- **Trade-offs of a bigger window** — larger context windows (100K–1M+ tokens) let you stuff more documents/history into a single call, but cost and latency scale with input size, and models can still lose track of details buried in the middle of a very long context ("lost in the middle").

**Why it matters:** every design decision in Phase 03 (chat history management, streaming, cost control) and Phase 05 (RAG, chunking) exists because context windows are finite and expensive — you can't just paste your entire knowledge base into every prompt.

<a id="embeddings--semantic-meaning"></a>

## Embeddings & Semantic Meaning

- **Embeddings** — a numerical vector representation of text (or images) where semantic meaning is encoded as position in high-dimensional space. Texts with similar meaning end up as vectors that are close together, even if they share no exact words.
- **Cosine similarity** — the standard way to measure "how close" two embeddings are — the cosine of the angle between two vectors, ranging from -1 (opposite) to 1 (identical meaning direction). This is the math behind semantic search.
- **Embeddings vs keyword search** — a keyword search for "car" won't match a document about "automobile"; an embedding-based search will, because it compares meaning, not exact text.

```ts
// Conceptual: comparing two pieces of text by meaning, not exact words
const a = await embed("How do I reset my password?");
const b = await embed("I forgot my login credentials");

const similarity = cosineSimilarity(a, b); // high — semantically related
```

**Why it matters:** embeddings are the foundation of Phase 05 (RAG) — vector search, document retrieval, and semantic chunking all depend on the same idea introduced here: meaning as geometry.

<a id="prompt-engineering-techniques"></a>

## Prompt Engineering Techniques

A prompt is the only interface you have to steer a probabilistic model toward a deterministic-feeling outcome. Each technique below is a lever for reducing ambiguity.

### System Prompts vs User Prompts

- **System prompt** — sets the model's persistent role, constraints, and behavior for the whole conversation (tone, output format, what it should refuse to do). Sent once, applies to every turn.
- **User prompt** — the actual per-turn request or question. Keeping instructions in the system prompt and data/questions in the user prompt makes behavior easier to reason about and test.

```ts
const response = await anthropic.messages.create({
  model: "claude-sonnet-5",
  system:
    "You are a support-ticket triage assistant. Always respond with valid JSON matching the given schema. Never invent ticket IDs.",
  messages: [{ role: "user", content: "Ticket #4521: customer can't log in after password reset." }],
});
```

### Zero-shot vs Few-shot Prompting

- **Zero-shot** — asking the model to perform a task with instructions alone, no examples. Fast to write, works well for tasks the model has broad training exposure to (general summarization, translation).
- **Few-shot** — including a handful of input/output examples directly in the prompt before the real request, so the model infers the exact pattern, format, and tone you want. Improves consistency for tasks with a specific or unusual output shape.

```text
Classify the sentiment as positive, negative, or neutral.

Review: "Shipping was fast and the product works great."
Sentiment: positive

Review: "Arrived broken and support never replied."
Sentiment: negative

Review: "It's fine, does what it says."
Sentiment: neutral

Review: "Best purchase I've made this year."
Sentiment:
```

### Role Prompting

- **Assigning a persona or expertise** ("You are a senior security engineer reviewing this code for vulnerabilities") shifts the model's tone, vocabulary, and the kind of considerations it surfaces — it doesn't grant new knowledge, but it focuses which knowledge gets applied.
- **Use it for framing, not for facts** — role prompting changes *how* the model answers, not what's true. Don't rely on a persona to make an answer more accurate; use grounding (RAG, tool use) for that.

### Chain-of-Thought Prompting

- **Chain-of-thought (CoT)** — asking the model to reason step by step before giving a final answer ("Think through this step by step, then give your answer"), which measurably improves accuracy on multi-step reasoning, math, and logic tasks.
- **Trade-off** — CoT produces more output tokens (higher cost and latency) and, for user-facing products, the reasoning trace is often hidden or stripped before display, since the final answer is usually what's wanted, not the scratch work.

### Structured Outputs

- **Why free-text output is risky** — a plain-English response is hard for downstream code to parse reliably; a model might phrase the same answer differently each run.
- **Forcing a shape** — instructing the model to respond only in JSON matching a schema (or using a provider's native structured-output/tool-use mode) turns an LLM call into something your application can `JSON.parse()` and trust the shape of.

```ts
const response = await anthropic.messages.create({
  model: "claude-sonnet-5",
  system: "Extract the fields below. Respond with ONLY valid JSON, no prose.",
  messages: [
    {
      role: "user",
      content: `Extract name, email, and issue_summary from:
"Hi, I'm Dana Lee (dana@example.com), my invoice #882 charged me twice."`,
    },
  ],
});

// Application code can now safely parse and validate the shape:
const parsed = InvoiceIssueSchema.parse(JSON.parse(response.content[0].text));
```

<a id="reliability-hallucination--structured-outputs"></a>

## Reliability, Hallucination & Structured Outputs

### Hallucination

- **Hallucination** — when a model states something false or fabricated with the same confident tone as a true statement. It happens because the model is optimizing for plausible next tokens, not verified facts — it has no built-in mechanism to say "I don't actually know this."
- **Where it bites hardest** — specific, checkable details: citations, statistics, API parameters, code that calls functions that don't exist, or facts about niche/recent topics outside training data.

### Reducing Hallucination in Practice

- **Grounding** — give the model the actual source material in context (via RAG or direct injection) instead of relying on parametric memory, and instruct it to answer only from the provided context.
- **Lower temperature** — reducing the model's sampling randomness (`temperature` closer to 0) makes output more deterministic and conservative, which helps for factual/extraction tasks (though it doesn't eliminate hallucination).
- **Ask it to say "I don't know"** — explicitly permitting uncertainty in the system prompt ("If the answer isn't in the provided context, say you don't know") measurably reduces confident fabrication.
- **Verification loops** — for high-stakes output, add a second pass (another model call, a rules check, or a human) that verifies claims against a source before they reach the user.

### Practice: Prompting for Real Tasks

Designing prompts is a skill you build by deliberately practicing across task types, not just chatting:

- **Summarization** — compress a document to N sentences/bullets while preserving key facts and dropping filler.
- **Classification** — assign one of a fixed set of labels (sentiment, topic, priority) with a defined fallback for ambiguous cases.
- **Translation** — convert between languages while preserving tone, formatting, and any embedded code/placeholders.
- **Extraction** — pull structured fields (names, dates, amounts) out of unstructured text into a strict schema.
- **Content generation** — produce new text (emails, descriptions, code) that follows a specific voice, length, and format constraint.

### Mini Project: Prompt Playground

Build a small app where a user can enter a prompt, choose a technique (zero-shot / few-shot / with system prompt), and compare outputs across models or settings side by side. This forces you to actually observe how each lever (temperature, few-shot examples, system prompt wording) changes real output — the fastest way to build intuition that no amount of reading replaces.

**Why it matters:** this is the bridge from Phase 02's prompting fundamentals into Phase 03 — the same techniques (system/user separation, structured outputs, reliability handling) get wired into real API integrations with streaming, retries, and multi-provider support.

---

⬅️ Back to [Table of Contents](../README.md)

## Interview Angle: 🧠 **[Full Question and Answers →](interview-qa.md)**

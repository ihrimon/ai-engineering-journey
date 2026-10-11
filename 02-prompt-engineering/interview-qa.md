# Interview Questions & Answers — Phase 02: LLM Fundamentals and Prompt Engineering

Quick-reference Q&A for everything covered in [README.md](README.md).

| # | Question |
|---|----------|
| 1 | [What is a large language model, fundamentally?](#q1-what-is-a-large-language-model-fundamentally) |
| 2 | [What role does self-attention play in a transformer?](#q2-what-role-does-self-attention-play-in-a-transformer) |
| 3 | [What is a token, and why does tokenization matter for building AI apps?](#q3-what-is-a-token-and-why-does-tokenization-matter-for-building-ai-apps) |
| 4 | [What is a context window, and why isn't it the same as "memory"?](#q4-what-is-a-context-window-and-why-isnt-it-the-same-as-memory) |
| 5 | [What is an embedding, and how is similarity measured?](#q5-what-is-an-embedding-and-how-is-similarity-measured) |
| 6 | [What's the difference between zero-shot and few-shot prompting?](#q6-whats-the-difference-between-zero-shot-and-few-shot-prompting) |
| 7 | [What is the difference between a system prompt and a user prompt?](#q7-what-is-the-difference-between-a-system-prompt-and-a-user-prompt) |
| 8 | [What is chain-of-thought prompting, and what does it cost you?](#q8-what-is-chain-of-thought-prompting-and-what-does-it-cost-you) |
| 9 | [Why would you force structured (JSON) output instead of letting the model respond in free text?](#q9-why-would-you-force-structured-json-output-instead-of-letting-the-model-respond-in-free-text) |
| 10 | [What is hallucination, and why does it happen?](#q10-what-is-hallucination-and-why-does-it-happen) |
| 11 | [What are practical ways to reduce hallucination in a production prompt?](#q11-what-are-practical-ways-to-reduce-hallucination-in-a-production-prompt) |
| 12 | [What does the `temperature` parameter actually control?](#q12-what-does-the-temperature-parameter-actually-control) |

---

<a id="q1-what-is-a-large-language-model-fundamentally"></a>

### Q1. What is a large language model, fundamentally?

> An LLM is a neural network trained to predict the next token in a sequence, given all the tokens before it. Every response — a single word or a full essay — is generated one token at a time from that same underlying prediction process.
>
> It becomes an "assistant" through additional training stages (instruction-tuning, RLHF) on top of that base next-token predictor, not through any separate reasoning engine.

---

<a id="q2-what-role-does-self-attention-play-in-a-transformer"></a>

### Q2. What role does self-attention play in a transformer?

> Self-attention lets the model weigh how relevant every other token in the input is when interpreting a given token — it's how the model resolves references (like a pronoun pointing back to a noun several sentences earlier) and captures long-range relationships in text.
>
> It's also why transformers process a sequence in parallel rather than one token at a time like older RNN architectures, and why compute cost grows with input length.

---

<a id="q3-what-is-a-token-and-why-does-tokenization-matter-for-building-ai-apps"></a>

### Q3. What is a token, and why does tokenization matter for building AI apps?

> A token is a sub-word chunk of text produced by a tokenizer — not a whole word and not a character. A word like "hamburger" might be one token or split into several depending on the tokenizer.
>
> It matters practically because API pricing, rate limits, and context-window limits are all measured in tokens, not characters or words — so cost estimation, truncation logic, and prompt-budget checks all need to count tokens directly instead of guessing from string length.

---

<a id="q4-what-is-a-context-window-and-why-isnt-it-the-same-as-memory"></a>

### Q4. What is a context window, and why isn't it the same as "memory"?

> The context window is the maximum number of tokens (input plus output combined) a model can process in a single request. Anything outside it simply isn't visible to the model.
>
> It isn't "memory" because an LLM has no persistent state between API calls — an application that appears to "remember" a conversation is actually resending the full conversation history as part of the prompt on every request, up to the context limit.

---

<a id="q5-what-is-an-embedding-and-how-is-similarity-measured"></a>

### Q5. What is an embedding, and how is similarity measured?

> An embedding is a numerical vector representation of text where semantic meaning is encoded as position in high-dimensional space — texts with similar meaning produce vectors that are close together, even without sharing exact words.
>
> Similarity is typically measured with cosine similarity — the cosine of the angle between two vectors — which is why embedding-based search can match "car" to "automobile" where plain keyword search would miss it.

---

<a id="q6-whats-the-difference-between-zero-shot-and-few-shot-prompting"></a>

### Q6. What's the difference between zero-shot and few-shot prompting?

> Zero-shot prompting gives the model instructions alone with no examples, relying on its general training to infer the task. Few-shot prompting includes a handful of input/output example pairs directly in the prompt before the real request.
>
> Few-shot is worth the extra tokens when the output needs a specific, consistent format or style that instructions alone don't reliably produce — the examples pin down the exact pattern.

---

<a id="q7-what-is-the-difference-between-a-system-prompt-and-a-user-prompt"></a>

### Q7. What is the difference between a system prompt and a user prompt?

> The system prompt sets the model's persistent role, constraints, and output format for the entire conversation, and is sent once. The user prompt is the actual per-turn question or request.
>
> Keeping standing instructions in the system prompt and variable data/questions in the user prompt keeps behavior predictable and makes prompts easier to test and version independently of the data flowing through them.

---

<a id="q8-what-is-chain-of-thought-prompting-and-what-does-it-cost-you"></a>

### Q8. What is chain-of-thought prompting, and what does it cost you?

> Chain-of-thought prompting asks the model to reason step by step before producing a final answer, which measurably improves accuracy on multi-step reasoning, math, and logic tasks.
>
> The cost is more output tokens — higher latency and higher spend per request — and in user-facing products the intermediate reasoning is usually hidden or stripped before display, since only the final answer is normally wanted.

---

<a id="q9-why-would-you-force-structured-json-output-instead-of-letting-the-model-respond-in-free-text"></a>

### Q9. Why would you force structured (JSON) output instead of letting the model respond in free text?

> Free-text responses are hard for application code to parse reliably, and the same underlying answer can be phrased differently across runs since the model is probabilistic.
>
> Forcing a schema-constrained JSON response (via prompting or a provider's structured-output/tool-use mode) turns an LLM call into something downstream code can `JSON.parse()` and validate the shape of, which is what makes LLM output usable as a building block inside a larger system rather than just chat text.

---

<a id="q10-what-is-hallucination-and-why-does-it-happen"></a>

### Q10. What is hallucination, and why does it happen?

> Hallucination is when a model states something false or fabricated with the same confident tone as a true statement.
>
> It happens because the model is optimizing for the most statistically plausible next token, not for verified truth — it has no built-in mechanism to distinguish "I know this" from "this sounds right," so it's most likely to show up on specific, checkable details like citations, statistics, or facts outside its training data.

---

<a id="q11-what-are-practical-ways-to-reduce-hallucination-in-a-production-prompt"></a>

### Q11. What are practical ways to reduce hallucination in a production prompt?

> Ground the model in real source material (via RAG or direct context injection) and instruct it to answer only from what's provided; lower the `temperature` for more conservative, deterministic output on factual tasks; explicitly permit uncertainty in the system prompt ("say you don't know if the answer isn't in the context") instead of forcing an answer; and for high-stakes output, add a verification pass — another model call, a rules check, or a human — before it reaches the user.

---

<a id="q12-what-does-the-temperature-parameter-actually-control"></a>

### Q12. What does the `temperature` parameter actually control?

> Temperature controls how much randomness is applied when the model samples the next token from its predicted probability distribution. Low temperature (near 0) makes the model consistently pick the highest-probability token, producing more deterministic, conservative output; higher temperature flattens the distribution, making lower-probability tokens more likely to be chosen, which increases variety and creativity but also the chance of less coherent or less accurate output.

# Phase 1 — LLM Fundamentals: Core Concepts (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ এখানে দেওয়া price/number গুলো **উদাহরণ** — real number সবসময় provider এর official docs থেকে দেখবে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Tokens](#tokens) | LLM শব্দ না, **token** (শব্দের টুকরো) পড়ে ও লেখে |
| [Tokenization](#tokenization) | Text → token এ ভাঙার algorithm (BPE ইত্যাদি) |
| [Context windows](#context-windows) | এক request এ model সর্বোচ্চ কত token দেখতে পারে |
| [Token cost](#token-cost) | Billing হয় token হিসেবে, input আর output আলাদা দামে |
| [Input vs output tokens](#input-vs-output-tokens) | Input parallel এ process হয় (সস্তা, দ্রুত), output একটা একটা করে (দামি, ধীর) |
| [Model latency](#model-latency) | `TTFT + output_tokens ÷ tokens_per_second` |
| [Model capabilities](#model-capabilities) | Model কী ভালো পারে — নিজের eval দিয়ে যাচাই করো |
| [Model limitations](#model-limitations) | Hallucination, cutoff, stateless, non-deterministic — design করে সামলাতে হয় |

```text
Text ──► Tokenizer ──► Token IDs ──► [ Context Window ] ──► Model ──► Output tokens ──► Text
                                          │                     │
                                   input tokens (prefill)   output tokens (decode)
                                          └──────── cost + latency ────────┘
```

---

<a id="tokens"></a>

## 1. Tokens

### What is it?
Token হলো text এর সবচেয়ে ছোট unit, যেটা LLM পড়ে ও generate করে। এটা পুরো শব্দ (`hello`), শব্দের অংশ (`ham` + `burger`), space সহ শব্দ (` the`), punctuation বা একটা byte হতে পারে। প্রতিটা token এর একটা integer **ID** থাকে।

### Why does it exist?
Neural network শুধু সংখ্যা বোঝে, text না। Character-level এ ভাঙলে sequence অনেক লম্বা হয়ে যায়; word-level এ ভাঙলে vocabulary অসীম, আর নতুন শব্দ (typo, নাম, code) handle করা যায় না। Token হলো মাঝামাঝি সমাধান।

### What problem does it solve?
একটা fixed vocabulary (প্রায় 50K–200K token) দিয়ে পৃথিবীর যেকোনো text — যেকোনো ভাষা, code, emoji — represent করা যায়।

### How does it work?
```text
"I love AI"  →  ["I", " love", " AI"]  →  [40, 3021, 15592]  →  model
model  →  next token এর probability  →  একটা token বেছে নেয়  →  আবার repeat
```
Model প্রতি step এ **একটাই token** generate করে, তারপর সেটা input এ যোগ করে পরেরটা predict করে।

### Important concepts
- English এ মোটামুটি **1 token ≈ 4 character ≈ ¾ word** (rule of thumb, exact না)।
- **Vocabulary** — tokenizer যত token চেনে তার list।
- **Special tokens** — যেমন end-of-text, chat role marker; model এর ভেতরের control signal।
- প্রতিটা model family এর **নিজস্ব tokenizer** — এক model এর token count অন্যটায় খাটে না।

### Trade-offs
বড় vocabulary → একই text এ কম token (সস্তা, দ্রুত) কিন্তু model এর embedding table বড়। ছোট vocabulary → উল্টো।

### Alternatives
Character/byte-level model (যেমন ByT5), পুরনো NLP এর word-level approach। Production LLM এ এখন subword token-ই standard।

### When should I use it?
সবসময় — cost, limit, chunk size, budget সব হিসাব **token এ** করবে, character বা word এ না।

### When should I NOT use it?
Character count দিয়ে token limit অনুমান করবে না, আর এক model এর tokenizer দিয়ে অন্য model এর token গুনবে না।

### Hands-on experiment
```ts
// npm i js-tiktoken
import { getEncoding } from "js-tiktoken";

const enc = getEncoding("o200k_base");
for (const text of ["I love programming", "আমি প্রোগ্রামিং ভালোবাসি"]) {
  const ids = enc.encode(text);
  console.log(text, "→", ids.length, "tokens", ids.map((id) => enc.decode([id])));
}
```
Output দেখে দুই ভাষার token সংখ্যা তুলনা করো।

### Production implementation
- Request পাঠানোর আগে token count করো; response এর `usage` field (input/output tokens) সবসময় log করো।

### Failure cases
- "strawberry তে কয়টা r?" — model ভুল করতে পারে, কারণ সে letter দেখে না, token দেখে।
- লম্বা number অদ্ভুতভাবে ভাগ হয় → arithmetic ভুল।

### Security concerns
User input এ special token এর মতো string থাকলে prompt structure ভাঙার চেষ্টা হতে পারে — official SDK/chat format ব্যবহার করো, নিজে raw string জোড়া দিয়ে prompt বানাবে না।

### Performance concerns
বেশি token = বেশি compute = বেশি latency।

### Cost concerns
Billing হয় token এ — token কমালে সরাসরি খরচ কমে।

### What did I learn?
- LLM এর দুনিয়ায় "শব্দ" বলে কিছু নেই — আছে token।
- Cost, latency, limit — তিনটারই unit হলো token।

### What can I explain to someone else?
> "LLM text কে Lego block এর মতো ছোট টুকরোয় ভাঙে — সেই টুকরোগুলোই token। Model একবারে একটা block বসিয়ে উত্তর বানায়, আর তোমাকে প্রতিটা block এর জন্য টাকা দিতে হয়।"

### GitHub Evidence
- [ ] Token counter script (English vs Bangla comparison)

### LinkedIn Post
- [ ] "বাংলায় AI ব্যবহার কেন English এর চেয়ে বেশি খরচের?" — token count এর তুলনা দিয়ে

---

<a id="tokenization"></a>

## 2. Tokenization

### What is it?
Text কে token এ ভাঙার algorithm/process। জনপ্রিয় algorithm: **BPE** (Byte Pair Encoding — GPT family), **WordPiece** (BERT), **SentencePiece / Unigram** (Llama, Gemma, T5)।

### Why does it exist?
Tokenizer ঠিক করে দেয় text কীভাবে সংখ্যায় রূপ নেবে — এটা ছাড়া model text এ train বা run করা সম্ভব না।

### What problem does it solve?
Unknown word সমস্যা (OOV) দূর করে — byte-level BPE তে যেকোনো character অন্তত byte হিসেবে represent হয়, তাই "unknown" token বলে কিছু থাকে না।

### How does it work?
BPE (সংক্ষেপে):
1. শুরু হয় single byte/character থেকে।
2. Training data তে পাশাপাশি সবচেয়ে বেশি আসা pair খুঁজে merge করে নতুন token বানায় (`t` + `h` → `th`)।
3. Vocabulary size এ পৌঁছানো পর্যন্ত repeat করে।
4. Encoding এর সময় শেখা merge rule গুলো একই order এ apply হয়।

### Important concepts
- **Space token এর অংশ** — `"hello"` আর `" hello"` আলাদা token।
- **Case sensitive** — `Hello` আর `hello` আলাদা হতে পারে।
- **Language bias** — tokenizer মূলত English-heavy data তে train হয়, তাই **বাংলার মতো ভাষায় একই অর্থের text এ অনেক বেশি token লাগে** (প্রায়ই কয়েক গুণ)।
- Claude/Gemini এর tokenizer public library হিসেবে নেই — provider এর **token counting API** ব্যবহার করতে হয়।

### Trade-offs
Compression (কম token) vs ভাষাভেদে fairness; বড় vocab vs model size।

### Alternatives
Byte-level/character-level model; নতুন model গুলোতে multilingual-friendly বড় vocabulary।

### When should I use it?
Chunking, cost estimate, context budgeting — সব জায়গায় **target model এর নিজস্ব tokenizer** বা count API।

### When should I NOT use it?
"Approximate" tokenizer দিয়ে hard limit enforce করবে না — limit এর কাছাকাছি হলে exact count নাও।

### Hands-on experiment
- একই বাক্য English আর বাংলায় encode করে ratio বের করো।
- প্রতিটা token আলাদা decode করে দেখো শব্দ কোথায় কোথায় ভাঙছে।

### Production implementation
- Chunk size token এ define করো (যেমন 500 token, 50 overlap), character এ না।
- Multilingual app এ ভাষা অনুযায়ী আলাদা cost estimate রাখো।

### Failure cases
- English ratio ধরে বাংলা app এর budget বানালে খরচ অনেক বেশি আসে।
- Character দিয়ে chunk করলে কিছু chunk token limit পার হয়ে যায়।

### Security concerns
Unicode trick — zero-width character, homoglyph (দেখতে একই, আলাদা character) দিয়ে keyword filter bypass করা যায়, কিন্তু model ঠিকই অর্থ বোঝে। Filter করার আগে text **normalize (NFKC)** করো।

### Performance concerns
বেশি token = লম্বা sequence = ধীর processing।

### Cost concerns
একই উত্তর বাংলায় দিলে English এর চেয়ে বেশি output token → বেশি খরচ।

### What did I learn?
- Tokenizer neutral না — ভাষাভেদে খরচ ও performance আলাদা।
- Exact count এর জন্য model-specific tool লাগে।

### What can I explain to someone else?
> "Tokenizer হলো একটা dictionary যেটা text থেকে বারবার আসা অক্ষর-জোড়া শিখে নেয়। English এ বেশি data থাকায় English শব্দ বড় টুকরোয় যায়, বাংলা ছোট ছোট টুকরোয় ভাঙে — তাই বাংলায় token বেশি লাগে।"

### GitHub Evidence
- [ ] Tokenizer visualizer (text → colored tokens)

### LinkedIn Post
- [ ] "Tokenization: AI এর ভাষাগত বৈষম্য কোথা থেকে আসে"

---

<a id="context-windows"></a>

## 3. Context Windows

### What is it?
এক request এ model সর্বোচ্চ কত token "দেখতে" পারে — **input + output মিলিয়ে**। এর বাইরের কিছু model এর কাছে অস্তিত্বহীন। Output এর জন্য আলাদা একটা max limit (`max_tokens`) থাকে।

### Why does it exist?
Transformer এর attention computation ও memory (KV cache) sequence length এর সাথে বাড়ে, আর model একটা নির্দিষ্ট length পর্যন্তই train হয়। তাই hardware ও training — দুই দিক থেকেই সীমা থাকে।

### What problem does it solve?
এটা নিজে problem solve করে না — এটা একটা **constraint**, যেটা জানলে বোঝা যায় কতটুকু information একবারে model কে দেওয়া যাবে।

### How does it work?
```text
┌──────────────────── Context Window ────────────────────┐
│ System prompt │ Tool definitions │ Retrieved docs │     │
│ Conversation history │ User message │ ► Output space ◄ │
└─────────────────────────────────────────────────────────┘
```
**Model stateless** — "chat মনে রাখছে" মনে হয় কারণ তোমার app প্রতিবার পুরো history আবার পাঠায়।

### Important concepts
- **Max output tokens** context window থেকে আলাদা limit।
- **Lost in the middle** — লম্বা context এর মাঝখানের তথ্য model প্রায়ই কম গুরুত্ব দেয়।
- **Effective context < advertised context** — 1M window মানে 1M token এ সমান ভালো performance না।

### Trade-offs
বড় window = বেশি তথ্য একসাথে, কিন্তু বেশি cost, বেশি latency, আর accuracy কমে যেতে পারে।

### Alternatives
RAG (শুধু relevant অংশ আনা), summarization, rolling memory, chunking — Phase 3 ও 4 এ বিস্তারিত।

### When should I use it?
পুরো একটা contract/report/codebase file একসাথে analysis — যেখানে পুরো context দরকার।

### When should I NOT use it?
পুরো knowledge base প্রতিটা request এ dump করবে না — এর জন্য RAG।

### Hands-on experiment
**Needle in a haystack**: লম্বা একটা text এর শুরু, মাঝ ও শেষে একটা গোপন তথ্য বসিয়ে model কে জিজ্ঞেস করো — কোন position এ model সবচেয়ে বেশি ভুল করে দেখো।

### Production implementation
Context budget ভাগ করে রাখো, যেমন:
```text
System 2K │ History 8K │ RAG 6K │ User 1K │ Output reserve 2K
```
Limit পার হলে পুরনো history summarize বা drop করো।

### Failure cases
- Limit পার হলে API error (400) — অথবা কিছু framework চুপচাপ পুরনো message কেটে দেয়।
- গুরুত্বপূর্ণ instruction লম্বা context এর মাঝে চাপা পড়ে ignore হয়।

### Security concerns
Context এ যত বেশি external content (document, web page, tool result), **prompt injection** এর সুযোগ তত বেশি।

### Performance concerns
Input লম্বা হলে **TTFT** বাড়ে, কারণ পুরো input আগে process (prefill) করতে হয়।

### Cost concerns
প্রতিটা turn এ পুরো history resend হয় — তাই লম্বা conversation এ cost turn সংখ্যার সাথে দ্রুত (প্রায় quadratic হারে) বাড়ে। **Prompt caching** এটা অনেক কমায়।

### What did I learn?
- Context window = model এর short-term working memory, persistent memory না।
- বড় window থাকলেও context engineering দরকার।

### What can I explain to someone else?
> "Context window হলো model এর টেবিলের সাইজ — যত কাগজ টেবিলে ধরে ততটুকুই সে একবারে দেখতে পারে। প্রতিবার নতুন প্রশ্নে টেবিল খালি হয়ে যায়, তাই আগের কথা আবার টেবিলে রাখতে হয়।"

### GitHub Evidence
- [ ] Needle-in-a-haystack experiment + result table

### LinkedIn Post
- [ ] "1M token context window থাকলেও কেন RAG লাগে?"

---

<a id="token-cost"></a>

## 4. Token Cost

### What is it?
LLM API এর billing হয় token হিসেবে — সাধারণত **per 1M tokens** দামে, input আর output আলাদা rate এ।

### Why does it exist?
Provider এর আসল খরচ GPU compute — আর compute সরাসরি token সংখ্যার সাথে বাড়ে। তাই token-based pricing।

### What problem does it solve?
Usage অনুযায়ী ন্যায্য billing — বেশি ব্যবহার করলে বেশি খরচ।

### How does it work?
```text
cost = (input_tokens / 1M × input_price) + (output_tokens / 1M × output_price)
```
**উদাহরণ** (কাল্পনিক দাম: input $3/1M, output $15/1M):
```text
এক request: 2,000 input + 500 output
= 0.006 + 0.0075 = $0.0135
দিনে 100,000 request → $1,350/দিন ≈ $40,500/মাস
```
ছোট সংখ্যাও scale এ বিশাল হয়ে যায়।

### Important concepts
- **System prompt + tool definitions প্রতিটা call এ input হিসেবে গোনা হয়।**
- **Cached input** অনেক সস্তা (prompt caching)।
- **Batch API** সাধারণত বেশ discount দেয় (non-real-time কাজের জন্য)।
- **Reasoning/thinking tokens output হিসেবে bill হয়।**
- Image/PDF ও token এ রূপান্তরিত হয়ে bill হয়।

### Trade-offs
সস্তা ছোট model vs দামি বড় model এর quality।

### Alternatives
Model routing (সহজ কাজ ছোট model এ), caching, prompt compression, batch processing, self-hosted model — Phase 10 এ বিস্তারিত।

### When should I use it?
প্রতিটা feature design এর শুরুতেই **cost per request × expected volume** হিসাব করো।

### When should I NOT use it?
শুধু per-token দাম দেখে model বাছবে না — সস্তা model বেশি retry বা লম্বা output দিলে মোট খরচ বেশি হতে পারে। **Cost per successful task** দেখো।

### Hands-on experiment
```ts
type Price = { inputPerM: number; outputPerM: number };

function estimateCost(inputTokens: number, outputTokens: number, p: Price) {
  return (inputTokens / 1e6) * p.inputPerM + (outputTokens / 1e6) * p.outputPerM;
}

console.log(estimateCost(2000, 500, { inputPerM: 3, outputPerM: 15 })); // 0.0135
```
API response এর `usage` থেকে real token নিয়ে এই function এ বসাও।

### Production implementation
- প্রতিটা request এর token ও cost **user/tenant/feature** অনুযায়ী log করো।
- Budget alert, per-user quota, আর সবসময় `max_tokens` set করো।

### Failure cases
- Agent loop থামছে না → রাতারাতি বিশাল bill।
- `max_tokens` না দেওয়ায় অপ্রত্যাশিত লম্বা output।

### Security concerns
**Denial-of-wallet** attack — কেউ ইচ্ছা করে লম্বা request পাঠিয়ে তোমার bill বাড়াতে পারে। Rate limit ও input size limit দাও।

### Performance concerns
কম token সাধারণত cost আর latency দুটোই কমায় — একই optimization এ দুই লাভ।

### Cost concerns
এই topic টাই cost নিয়ে 🙂 — মূল কথা: **measure first, optimize second.**

### What did I learn?
- AI feature এর unit economics token দিয়ে হিসাব হয়।
- Hidden token (system prompt, tools, reasoning) খরচের বড় অংশ হতে পারে।

### What can I explain to someone else?
> "LLM ট্যাক্সির মতো — মিটার চলে token এ। যাওয়ার পথ (input) এক রেটে, ফেরার পথ (output) আরও বেশি রেটে।"

### GitHub Evidence
- [ ] Token cost calculator + usage logger

### LinkedIn Post
- [ ] "$0.01 এর একটা AI request কীভাবে মাসে $40K হয়ে যায়"

---

<a id="input-vs-output-tokens"></a>

## 5. Input vs Output Tokens

### What is it?
- **Input (prompt) tokens** — তুমি যা পাঠাও: system prompt, history, documents, user message।
- **Output (completion) tokens** — model যা generate করে।

### Why does it exist?
এরা আলাদাভাবে process হয়, তাই খরচ ও গতিও আলাদা।

### What problem does it solve?
এই পার্থক্য বুঝলে বোঝা যায় কোথায় optimize করলে সবচেয়ে বেশি লাভ।

### How does it work?
```text
PREFILL  (input):  সব input token একসাথে parallel এ process  → দ্রুত, সস্তা
DECODE   (output): প্রতিটা token এর জন্য আলাদা forward pass → ধীর, দামি
```
তাই output token সাধারণত input এর চেয়ে **কয়েক গুণ দামি** (প্রায়ই ৪–৫ গুণ)।

### Important concepts
- `max_tokens` output এর ceiling।
- Response এর **`stop_reason` / `finish_reason`** — `max_tokens` এ থামলে output অসম্পূর্ণ।
- Reasoning model এর "thinking" token = output token।

### Trade-offs
লম্বা, বিস্তারিত output = ভালো explanation কিন্তু বেশি খরচ ও latency।

### Alternatives
Structured short output (JSON), "শুধু উত্তর দাও, ব্যাখ্যা না" ধরনের instruction।

### When should I use it?
Prompt design এ: input এ প্রয়োজনীয় context দাও, কিন্তু **output যতটা সম্ভব ছোট ও নির্দিষ্ট** চাও।

### When should I NOT use it?
Output এত ছোট করবে না যে quality নষ্ট হয় — বিশেষ করে reasoning task এ।

### Hands-on experiment
একই প্রশ্ন দুইভাবে পাঠাও: "বিস্তারিত ব্যাখ্যা করো" vs "এক লাইনে উত্তর দাও" — `usage` আর response time তুলনা করো।

### Production implementation
- সবসময় `stop_reason` check করো; truncated হলে retry বা handle করো।
- Input আর output token আলাদা metric হিসেবে track করো।

### Failure cases
`max_tokens` কম দেওয়ায় JSON মাঝপথে কেটে গেছে → `JSON.parse` error।

### Security concerns
User যদি model কে খুব লম্বা output দিতে বাধ্য করতে পারে → cost attack। Output limit server-side enforce করো।

### Performance concerns
**Latency এর সবচেয়ে বড় driver হলো output length**, input না।

### Cost concerns
Output token কমানো = সবচেয়ে কার্যকর cost optimization।

### What did I learn?
- Input পড়া সস্তা, output লেখা দামি।
- Output সংক্ষিপ্ত রাখা cost আর speed দুটোর জন্যই ভালো।

### What can I explain to someone else?
> "Model এর জন্য পড়া সহজ — পুরো পাতা একবারে পড়ে ফেলে। লেখা কঠিন — একটা একটা শব্দ করে লেখে। তাই লেখার (output) দাম বেশি, সময়ও বেশি।"

### GitHub Evidence
- [ ] Verbose vs concise prompt experiment (token + latency table)

### LinkedIn Post
- [ ] "Prefill vs Decode: কেন AI এর output দামি"

---

<a id="model-latency"></a>

## 6. Model Latency

### What is it?
Request পাঠানো থেকে response পাওয়া পর্যন্ত সময়। মূল metric:
- **TTFT** (Time To First Token) — প্রথম token আসতে কত সময়।
- **TPS** (Tokens Per Second) — generation এর গতি।
- **Total latency** — পুরো response শেষ হতে কত সময়।

### Why does it exist?
Network, provider এর queue, input processing (prefill), আর token-by-token generation (decode) — প্রতিটা ধাপেই সময় লাগে।

### What problem does it solve?
Latency মাপলে বোঝা যায় UX কেমন হবে আর কোথায় bottleneck।

### How does it work?
```text
total ≈ network + queue + TTFT + (output_tokens ÷ TPS)

উদাহরণ: TTFT 0.5s, 50 tokens/s, 500 output tokens
→ 0.5 + 10 = 10.5s
Streaming চালু থাকলে user 0.5s এই লেখা দেখা শুরু করে।
```

### Important concepts
- **Streaming** total latency কমায় না, কিন্তু **perceived latency** অনেক কমায়।
- p50 / p95 / p99 — average না, tail latency দেখো।
- Agent এ প্রতিটা tool call = আরেকটা full round-trip।

### Trade-offs
বড় model / reasoning = ভালো quality কিন্তু ধীর; ছোট model = দ্রুত কিন্তু দুর্বল।

### Alternatives
Streaming, ছোট model, prompt caching (TTFT কমায়), parallel call, ছোট output, semantic cache।

### When should I use it?
- Chat UI → TTFT সবচেয়ে গুরুত্বপূর্ণ।
- Voice AI → পুরো pipeline এ খুব কঠোর latency budget (প্রায় ১ সেকেন্ডের আশেপাশে)।
- Batch job → latency না, throughput আর cost বেশি জরুরি।

### When should I NOT use it?
Background/batch কাজে latency optimize করতে গিয়ে দামি fast model ব্যবহার করবে না।

### Hands-on experiment
```ts
const start = performance.now();
let firstTokenAt = 0;
let outputTokens = 0;

for await (const chunk of stream) {           // যেকোনো provider এর streaming response
  if (!firstTokenAt) firstTokenAt = performance.now();
  outputTokens++;                              // আনুমানিক; exact count usage থেকে নাও
}

const end = performance.now();
console.log("TTFT ms:", firstTokenAt - start);
console.log("TPS:", outputTokens / ((end - firstTokenAt) / 1000));
```

### Production implementation
- TTFT, TPS, total latency এর p50/p95 dashboard।
- Timeout, exponential backoff সহ retry, fallback model।

### Failure cases
- লম্বা generation এ HTTP timeout।
- ১০টা sequential tool call → latency ১০ গুণ।

### Security concerns
খুব লম্বা input/output দিয়ে latency বাড়িয়ে service slow করা (resource exhaustion) — size limit ও rate limit দাও।

### Performance concerns
এই topic টাই performance — মূল lever: **output ছোট করো, stream করো, cache করো।**

### Cost concerns
দ্রুত provider/model অনেক সময় দামি; latency আর cost এর balance খুঁজতে হয়।

### What did I learn?
- Latency = TTFT + generation time; output length সবচেয়ে বড় factor।
- Streaming UX এর জন্য প্রায় বাধ্যতামূলক।

### What can I explain to someone else?
> "রেস্টুরেন্টে খাবার আসতে কতক্ষণ লাগে (TTFT) আর প্রতিটা আইটেম কত দ্রুত আসে (TPS) — দুটো মিলেই অপেক্ষার সময়। Streaming মানে রান্না হওয়া মাত্র এক এক করে পরিবেশন।"

### GitHub Evidence
- [ ] Latency benchmark script (TTFT/TPS, ২–৩টা model)

### LinkedIn Post
- [ ] "AI app ধীর লাগে কেন? TTFT vs TPS"

---

<a id="model-capabilities"></a>

## 7. Model Capabilities

### What is it?
একটা model কী কী কাজ ভালো পারে: text generation, summarization, classification, extraction, translation, code, reasoning, **tool/function calling**, **structured output (JSON)**, vision/multimodal input, long context, multilingual।

### Why does it exist?
সব model সমান না — size, training data, fine-tuning অনুযায়ী capability আলাদা।

### What problem does it solve?
Capability জানলে **task অনুযায়ী সঠিক model** বাছা যায় — অতিরিক্ত দামি model বা অক্ষম model এড়ানো যায়।

### How does it work?
Pre-training থেকে আসে ভাষা ও জ্ঞান; instruction tuning/RLHF থেকে আসে instruction মানা, tool use, structured output। **In-context learning** — prompt এ example দিলে নতুন pattern শিখে ফেলে (few-shot)।

### Important concepts
- **Capability ≠ reliability** — "পারে" মানে "সবসময় ঠিকভাবে পারে" না।
- Benchmark দিক নির্দেশ করে, কিন্তু **তোমার task এর নিজস্ব eval** সবচেয়ে বিশ্বাসযোগ্য।
- Model card / provider docs এ supported feature (tools, vision, JSON mode, context size) দেখো।

### Trade-offs
Frontier model = সর্বোচ্চ capability কিন্তু দামি ও ধীর; ছোট model = সস্তা, দ্রুত, সীমিত।

### Alternatives
Traditional ML model, rule-based code, specialized API (OCR, translation) — কিছু কাজে এগুলোই ভালো।

### When should I use it?
অসংগঠিত ভাষা নিয়ে কাজ: summarize, classify, extract, generate, fuzzy matching।

### When should I NOT use it?
- Exact হিসাব, deterministic business logic।
- সহজ rule (regex/if-else) দিয়ে যা হয়।
- Tool ছাড়া real-time তথ্য।

### Hands-on experiment
১০টা নিজের real task বানিয়ে ২–৩টা model এ চালাও, accuracy/latency/cost table বানাও — এটাই Phase 1 এর **LLM Model Comparison Lab** এর শুরু।

### Production implementation
- Use case অনুযায়ী capability matrix রাখো।
- Model config এ রাখো (hardcode না) যাতে feature flag দিয়ে model বদলানো যায়।

### Failure cases
Benchmark score দেখে model বাছা, কিন্তু নিজের domain (যেমন বাংলা legal document) এ খারাপ performance।

### Security concerns
বেশি capable model (tool use, code execution) = বেশি ক্ষতির সম্ভাবনা — permission সীমিত রাখো।

### Performance concerns
বেশি capability প্রায়ই বড় model = বেশি latency।

### Cost concerns
সব কাজে সবচেয়ে বড় model লাগে না — **সবচেয়ে ছোট যে model কাজটা ঠিকঠাক পারে**, সেটাই বাছো।

### What did I learn?
- Model selection একটা engineering decision, eval দিয়ে করতে হয়।

### What can I explain to someone else?
> "Model বাছা মানে কাজের জন্য সঠিক লোক নিয়োগ — সব কাজে সবচেয়ে সিনিয়র লোক লাগে না। Interview (eval) নিয়ে দেখো কে কাজটা ভালো পারে।"

### GitHub Evidence
- [ ] LLM Model Comparison Lab (initial eval set)

### LinkedIn Post
- [ ] "Benchmark দেখে model বাছা কেন ভুল"

---

<a id="model-limitations"></a>

## 8. Model Limitations

### What is it?
LLM এর মৌলিক সীমাবদ্ধতা:
- **Hallucination** — আত্মবিশ্বাসের সাথে ভুল/বানানো তথ্য।
- **Knowledge cutoff** — training এর পরের তথ্য জানে না।
- **Stateless** — call এর মাঝে কোনো memory নেই।
- **Non-deterministic** — একই input এ ভিন্ন output।
- **Exact math/counting এ দুর্বল** (tokenization এর কারণে আংশিকভাবে)।
- **Lost in the middle**, prompt sensitivity, bias, sycophancy (user কে খুশি করার প্রবণতা)।
- **Prompt injection** এ সহজে প্রভাবিত।

### Why does it exist?
LLM মূলত **next-token predictor** — "সত্য" না, "সম্ভাব্য/বিশ্বাসযোগ্য" text বানাতে optimized।

### What problem does it solve?
Limitation জানা থাকলে system design করে সেগুলো সামলানো যায় — এটাই AI Engineering এর মূল কাজ।

### How does it work?
Model এর কাছে "জানি না" বলার স্বাভাবিক mechanism দুর্বল; training data তে যা statistically সম্ভাব্য, সেটাই generate করে — সত্য হোক বা না হোক।

### Important concepts
প্রতিটা limitation এর engineering সমাধান:

| Limitation | Mitigation |
|---|---|
| Hallucination | RAG + source citation, validation |
| Knowledge cutoff | Web search / tools |
| Stateless | Memory system (Phase 3) |
| Math/counting | Calculator / code execution tool |
| Non-determinism | Low temperature, structured output, eval |
| Prompt injection | Guardrails, least privilege (Phase 11) |

### Trade-offs
প্রতিটা mitigation এ নতুন complexity, latency ও cost যোগ হয়।

### Alternatives
High-stakes কাজে deterministic system বা human review।

### When should I use it?
প্রতিটা AI feature design এর সময় নিজেকে জিজ্ঞেস করো: "এখানে model ভুল করলে কী হবে?"

### When should I NOT use it?
Human review ছাড়া medical/legal/financial চূড়ান্ত সিদ্ধান্তে LLM output সরাসরি ব্যবহার করবে না।

### Hands-on experiment
- Cutoff এর পরের কোনো ঘটনা জিজ্ঞেস করো।
- একটা শব্দে নির্দিষ্ট অক্ষর কয়টা গুনতে বলো।
- কোনো বিষয়ে "research paper citation" চাও, তারপর সেগুলো আসলে আছে কিনা যাচাই করো।
- একই prompt ৫ বার চালিয়ে output তুলনা করো।

### Production implementation
- **Model output কখনো blindly trust করবে না** — schema validation, business rule check।
- Source দেখাও, uncertainty হলে fallback বা human escalation।

### Failure cases
Chatbot বানানো refund policy বলে দিল, customer সেটা দাবি করল — real business loss।

### Security concerns
Injection, data leakage, আর user এর অতিরিক্ত বিশ্বাস (over-reliance)।

### Performance concerns
Validation, retry, guardrail — প্রতিটা latency বাড়ায়; balance দরকার।

### Cost concerns
Retry আর extra check token খরচ বাড়ায়, কিন্তু ভুলের খরচ সাধারণত তার চেয়ে অনেক বেশি।

### What did I learn?
- LLM শক্তিশালী কিন্তু অবিশ্বস্ত component — **reliability আসে system design থেকে, model থেকে না।**

### What can I explain to someone else?
> "LLM একজন খুব জ্ঞানী কিন্তু মাঝে মাঝে আত্মবিশ্বাসের সাথে ভুল বলা intern এর মতো। তার কাজ যাচাই করার process (validation, source, review) তোমাকেই বানাতে হবে।"

### GitHub Evidence
- [ ] Limitation demo notebook (hallucination, cutoff, counting, non-determinism)

### LinkedIn Post
- [ ] "LLM এর ৬টা limitation আর প্রতিটার engineering সমাধান"

---

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Root README](../README.md)

# Phase 4 — RAG & Knowledge Systems: Query Optimization (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Code example গুলো TypeScript এ (local: Ollama `qwen2.5:3b` + `bge-m3`)। প্রতিটা technique **eval দিয়ে যাচাই** করবে। সব technique সব data তে কাজ করে না।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Query rewriting](#query-rewriting) | User এর অস্পষ্ট/conversational প্রশ্নকে search-friendly, standalone query তে রূপান্তর |
| [Query expansion](#query-expansion) | Synonym, related term, acronym এর পূর্ণরূপ যোগ করে recall বাড়ানো |
| [HyDE](#hyde) | আগে একটা কাল্পনিক উত্তর লেখাও, তারপর সেই উত্তরের embedding দিয়ে খোঁজো |
| [Multi-query retrieval](#multi-query-retrieval) | একই প্রশ্নের কয়েকটা ভিন্ন রূপ বানিয়ে আলাদা search, তারপর RRF |
| [Multi-hop retrieval](#multi-hop-retrieval) | প্রথম search এর ফল দেখে পরের search: চেইন প্রশ্নের জন্য |
| [Agentic retrieval](#agentic-retrieval) | Agent নিজে ঠিক করে কখন, কোথায়, কী দিয়ে খুঁজবে আর কখন থামবে |

```text
User question ──► [ Query optimization layer ] ──► retriever(s) ──► rerank ──► LLM
                    rewrite · expand · HyDE
                    multi-query · decompose (multi-hop)
                    agentic loop (search → read → search again)

মূল কথা: "User যেভাবে প্রশ্ন করে" আর "document যেভাবে লেখা" — এই gap কমানো।
```

---

<a id="query-rewriting"></a>

## 1. Query Rewriting

### What is it?
User এর প্রশ্নকে retrieval এর জন্য ভালো রূপে নতুন করে লেখা: conversation থেকে **standalone** বানানো ("ওটার দাম কত?" → "Pro plan এর মাসিক দাম কত?"), typo ঠিক করা, অপ্রয়োজনীয় অংশ বাদ দেওয়া, ভাষা বদলানো (বাংলা প্রশ্ন → English doc এর জন্য)।

### Why does it exist?
Conversational প্রশ্নে pronoun আর আগের context থাকে, যা search engine বোঝে না। User এর ভাষা আর document এর ভাষাও আলাদা।

### What problem does it solve?
Follow-up প্রশ্নে retrieval ভেঙে পড়া ঠেকায়, আর noise কমায়।

### How does it work?
```text
history: "Pro plan এ কী কী আছে?" → (উত্তর)
user:    "আর ওটার দাম?"
rewrite → "Pro plan এর দাম কত?"   → retriever
```

### Important concepts
- **Condense question** pattern (history + follow-up → standalone)।
- **Rewrite শুধু retrieval এর জন্য**। Final answer এ user এর আসল প্রশ্নই দাও।
- **ছোট/দ্রুত model** দিয়ে করো।
- **Skip when not needed**: প্রথম turn আর ইতিমধ্যে স্পষ্ট প্রশ্নে rewrite লাগে না।

### Trade-offs
ভালো retrieval, কিন্তু extra LLM call (latency), আর rewrite ভুল হলে (অর্থ বদলালে) পুরো pipeline ভুল।

### Alternatives
History সহ পুরো text embed করা (noisy), শেষ কয়েক turn concat।

### When should I use it?
Multi-turn chat RAG এ প্রায় বাধ্যতামূলক।

### When should I NOT use it?
Single-turn search box এ পরিষ্কার keyword query তে।

### Hands-on experiment
```ts
const REWRITE = `Rewrite the user's last message as a standalone search query using the conversation.
Keep names, numbers, and product terms. Output only the query.`;

async function rewrite(history: string, last: string) {
  const r = await ollama.chat({
    model: "qwen2.5:3b",
    messages: [{ role: "system", content: REWRITE }, { role: "user", content: `${history}\n\nLast: ${last}` }],
  });
  return r.message.content.trim();
}
```
১০টা follow-up প্রশ্নে rewrite সহ ও ছাড়া Recall@5 তুলনা করো।

### Production implementation
- Log: original query, rewritten query, retrieved ids (debug এর জন্য অপরিহার্য)।
- Rewrite timeout: ব্যর্থ হলে original দিয়ে search।

### Failure cases
Rewrite এ model নিজের অনুমান যোগ করেছে ("Pro plan এর ২০২৩ দাম"), ফলে ভুল filter।

### Security concerns
User message এর injection rewrite prompt কে প্রভাবিত করতে পারে। Rewrite output কে শুধু query হিসেবে ব্যবহার করো, instruction হিসেবে না।

### Performance concerns
+১০০–৫০০ms (ছোট model)। TTFT এ যোগ হয়।

### Cost concerns
ছোট call। সস্তা model ব্যবহার করো।

### What did I learn?
Search engine conversation বোঝে না। তাকে পরিষ্কার একটা প্রশ্ন দিতে হয়।

### What can I explain to someone else?
> "লাইব্রেরিয়ানকে 'ওই বইটা' বললে বুঝবে না। আগের কথাবার্তা থেকে বইয়ের পুরো নাম বের করে বলতে হয়।"

### GitHub Evidence
- [ ] Follow-up rewrite eval

### LinkedIn Post
- [ ] "Chatbot এর follow-up প্রশ্নে RAG কেন ভেঙে পড়ে"

---

<a id="query-expansion"></a>

## 2. Query Expansion

### What is it?
Original query এর সাথে related term যোগ করা: synonym ("refund" + "money back" + "টাকা ফেরত"), acronym এর পূর্ণরূপ ("KYC" + "Know Your Customer"), domain term। LLM, thesaurus, বা pseudo-relevance feedback (প্রথম result থেকে term নেওয়া) দিয়ে।

### Why does it exist?
**Vocabulary mismatch**: user এক শব্দ ব্যবহার করে, document অন্য শব্দ। বিশেষ করে keyword (BM25) search এ এটা বড় সমস্যা।

### What problem does it solve?
Recall বাড়ায়: যেসব relevant doc ভিন্ন শব্দে লেখা, সেগুলোও আসে।

### How does it work?
```text
"KYC কেন লাগে?"
→ expand: "KYC Know Your Customer verification identity কেন প্রয়োজন requirement"
→ BM25 / hybrid search
```

### Important concepts
- **Sparse search এ বেশি লাভ**। Dense ইতিমধ্যে কিছু synonym বোঝে।
- **Weight**: original term বেশি গুরুত্ব পাবে।
- **Domain glossary**: company এর নিজস্ব acronym list deterministic expansion এর জন্য সেরা।
- **Over-expansion** = precision কমে (topic drift)।

### Trade-offs
Recall ↑, precision ↓ (বেশি noise)। তাই প্রায়ই reranker সাথে লাগে।

### Alternatives
Multi-query, HyDE, ভালো multilingual embedding।

### When should I use it?
Acronym/jargon-heavy domain, BM25-heavy pipeline, বাংলা-ইংরেজি মিশ্র query।

### When should I NOT use it?
Exact-match query (ID, code) এ। Expansion এখানে ক্ষতি করে।

### Hands-on experiment
একটা glossary (`{ "KYC": ["Know Your Customer"], "OTP": ["one time password"] }`) বানাও। BM25 এ expansion সহ ও ছাড়া ১০টা acronym প্রশ্নের recall মাপো।

### Production implementation
- Deterministic glossary expansion আগে, LLM expansion optional।
- Expansion term log করো।

### Failure cases
"Apple" expand হয়ে "fruit" যোগ হয়েছে tech support corpus এ, ফলে noise।

### Security concerns
নগণ্য।

### Performance concerns
Glossary expansion ~০ms। LLM expansion এ একটা call।

### Cost concerns
Glossary free।

### What did I learn?
User আর document এর "ভাষার" ফাঁক পূরণ করাই expansion।

### What can I explain to someone else?
> "Google এ এক শব্দে না পেলে তুমি নিজেই আরেকটা প্রতিশব্দ দিয়ে খোঁজো। Expansion সেটা আগেই করে দেয়।"

### GitHub Evidence
- [ ] Glossary expansion on BM25

### LinkedIn Post
- [ ] —

---

<a id="hyde"></a>

## 3. HyDE (Hypothetical Document Embeddings)

### What is it?
প্রশ্ন সরাসরি embed না করে, আগে LLM দিয়ে প্রশ্নের একটা **কাল্পনিক উত্তর (hypothetical document)** লেখাও, তারপর সেই উত্তরের embedding দিয়ে search করো। (Gao et al., 2022)

### Why does it exist?
প্রশ্ন আর উত্তরের লেখার ধরন আলাদা। প্রশ্ন ছোট ও জিজ্ঞাসাবাচক, document লম্বা ও বিবৃতিমূলক। Embedding space এ "উত্তরের মতো দেখতে" text এর কাছে আসল উত্তর থাকার সম্ভাবনা বেশি।

### What problem does it solve?
Question-document asymmetry, বিশেষ করে ছোট/অস্পষ্ট প্রশ্ন আর domain-specific corpus এ।

### How does it work?
```text
Q: "পাসওয়ার্ড ভুলে গেলে কী করব?"
LLM → "পাসওয়ার্ড রিসেট করতে login পেজে 'Forgot password' এ ক্লিক করুন, ইমেইলে একটা link যাবে..."
       (ভুল detail থাকতে পারে — সমস্যা নেই, শুধু "আকৃতি" দরকার)
embed(hypothetical answer) → ANN → আসল document
```

### Important concepts
- **Hypothetical answer factually ভুল হতে পারে**, শুধু retrieval এর জন্য ব্যবহার, user কে দেখানো হয় না।
- **Query + hypothetical** দুটোর embedding average করাও একটা variant।
- **Model knowledge লাগে**: সম্পূর্ণ private/অজানা domain এ hypothetical অর্থহীন হতে পারে।

### Trade-offs
কিছু ক্ষেত্রে retrieval ভালো, কিন্তু extra LLM generation (latency), আর LLM ভুল দিকে গেলে retrieval ও ভুল দিকে যায়।

### Alternatives
Query rewriting, ভালো asymmetric embedding model (query/doc prefix সহ), doc এর জন্য hypothetical questions (উল্টো: indexing এর সময় প্রতিটা chunk এর জন্য সম্ভাব্য প্রশ্ন বানিয়ে embed)।

### When should I use it?
ছোট, অস্পষ্ট প্রশ্ন, আর general-knowledge-ঘেঁষা domain।

### When should I NOT use it?
Latency-critical path, সম্পূর্ণ proprietary jargon, বা ID/number-ভিত্তিক query।

### Hands-on experiment
```ts
async function hydeSearch(q: string) {
  const fake = await ollama.chat({
    model: "qwen2.5:3b",
    messages: [{ role: "user", content: `Write a short passage that answers: ${q}` }],
  });
  const { embeddings: [v] } = await ollama.embed({ model: "bge-m3", input: fake.message.content });
  return vectorSearch(v, 5);
}
```
২০টা প্রশ্নে plain vs HyDE এর Recall@5 তুলনা করো। কোন ধরনের প্রশ্নে HyDE জেতে, কোথায় হারে, লেখো।

### Production implementation
- Feature flag দিয়ে চালু/বন্ধ, eval দিয়ে সিদ্ধান্ত।
- Plain আর HyDE দুটো চালিয়ে RRF (safer)।

### Failure cases
Company-specific প্রশ্নে ("Acme এর leave policy") LLM একটা generic policy বানিয়েছে, ফলে generic doc retrieve।

### Security concerns
Hypothetical text কখনো final answer বা citation হিসেবে যাবে না।

### Performance concerns
একটা পূর্ণ generation (+০.৫–২s)। বড় latency cost।

### Cost concerns
প্রতি query তে output token খরচ।

### What did I learn?
কখনো কখনো উত্তর খুঁজতে আগে উত্তরের "খসড়া" কল্পনা করা কাজে দেয়।

### What can I explain to someone else?
> "হারানো জিনিস খুঁজতে আগে কল্পনা করো ওটা দেখতে কেমন। ছবিটা মাথায় থাকলে খোঁজা সহজ, এমনকি ছবির রং একটু ভুল হলেও।"

### GitHub Evidence
- [ ] Plain vs HyDE recall by question type

### LinkedIn Post
- [ ] "HyDE: উত্তর কল্পনা করে উত্তর খোঁজা"

---

<a id="multi-query-retrieval"></a>

## 4. Multi-Query Retrieval

### What is it?
LLM দিয়ে একই প্রশ্নের ৩–৫টা ভিন্ন রূপ (paraphrase/perspective) বানানো, প্রতিটা দিয়ে আলাদা search, তারপর ফলাফল RRF দিয়ে মেলানো। **RAG-Fusion** নামেও পরিচিত।

### Why does it exist?
একটা query embedding প্রশ্নের একটা "দিক" ধরে। ভিন্নভাবে জিজ্ঞেস করলে ভিন্ন relevant doc আসে।

### What problem does it solve?
একক query এর phrasing এর উপর নির্ভরতা কমায়, recall আর robustness বাড়ায়।

### How does it work?
```text
"remote কাজের নিয়ম কী?"
 ├─ "work from home policy"
 ├─ "দূর থেকে কাজ করার অনুমতি ও শর্ত"
 └─ "remote employee eligibility and equipment"
   → ৪টা search (original সহ) parallel → RRF → top-k
```

### Important concepts
- **Original query সবসময় রাখো**।
- **Diversity**: variant গুলো আলাদা দিক ধরবে, শুধু শব্দ বদল না।
- **Parallel execution**।
- **Dedup** fusion এর আগে।

### Trade-offs
Recall ↑, কিন্তু N গুণ search, একটা extra LLM call, আর context এ বেশি noise এর ঝুঁকি।

### Alternatives
Query expansion (এক query তে), HyDE, hybrid search।

### When should I use it?
Broad/অস্পষ্ট প্রশ্ন, বড় heterogeneous corpus।

### When should I NOT use it?
Specific factual lookup, কঠোর latency/cost budget।

### Hands-on experiment
```ts
const variants = await generateVariants(q, 3);          // LLM → JSON array
const lists = await Promise.all([q, ...variants].map((v) => searchIds(v, 20)));
const fused = rrf(lists).slice(0, 5);                   // retrieval.md এর rrf()
```
Single vs multi-query এর Recall@5 আর latency মাপো।

### Production implementation
Variant generation এ structured output (JSON array), search parallel, total timeout।

### Failure cases
Variant গুলো প্রায় একই ("remote policy", "remote work policy", "policy for remote"), ফলে কোনো লাভ নেই, শুধু খরচ।

### Security concerns
প্রতিটা search এ একই ACL filter।

### Performance concerns
Search parallel, কিন্তু variant generation sequential (+ LLM latency)।

### Cost concerns
১টা LLM call + N embedding + N search।

### What did I learn?
একই প্রশ্ন কয়েকভাবে জিজ্ঞেস করলে বেশি উত্তর পাওয়া যায়।

### What can I explain to someone else?
> "একটা বিষয়ে লাইব্রেরিতে খুঁজতে ৩ জন বন্ধুকে আলাদা শব্দে খুঁজতে পাঠাও, তারপর সবার আনা বই মিলিয়ে সবচেয়ে বেশিবার আসা বইগুলো নাও।"

### GitHub Evidence
- [ ] Single vs multi-query (RAG-Fusion) eval

### LinkedIn Post
- [ ] —

---

<a id="multi-hop-retrieval"></a>

## 5. Multi-Hop Retrieval

### What is it?
যেসব প্রশ্নের উত্তর একাধিক document এর তথ্য **চেইন করে** পেতে হয়, সেখানে ধাপে ধাপে retrieval: প্রথম search এর ফল থেকে নতুন তথ্য নিয়ে পরের search।
উদাহরণ: "আমাদের সবচেয়ে বড় client এর account manager কোন office এ বসে?" → (১) সবচেয়ে বড় client কে? → (২) তার account manager কে? → (৩) সে কোন office এ?

### Why does it exist?
Single retrieval এ প্রশ্নের embedding শেষ ধাপের তথ্যের সাথে মেলে না। মাঝের তথ্য (client এর নাম) না জেনে সেটা খোঁজা যায় না।

### What problem does it solve?
Compositional/bridge প্রশ্ন, comparison প্রশ্ন ("X আর Y এর policy পার্থক্য")।

### How does it work?
```text
Option A — Decompose first:  question → sub-questions [q1, q2, q3] → sequential (q2 তে q1 এর উত্তর বসাও)
Option B — Iterative:        retrieve → read → "আর কী জানা দরকার?" → next query → ... → enough → answer
```

### Important concepts
- **Question decomposition** (LLM দিয়ে)।
- **Bridge entity**: মাঝের ধাপে পাওয়া নাম/ID যা পরের query তে যায়।
- **Stopping condition**: max hop (যেমন ৩), বা "যথেষ্ট তথ্য আছে"।
- **Error propagation**: প্রথম hop ভুল হলে বাকি সব ভুল।

### Trade-offs
জটিল প্রশ্নের সমাধান, কিন্তু বহু LLM call, বেশি latency, আর error compounding।

### Alternatives
GraphRAG (relationship আগে থেকেই graph এ), structured retrieval (SQL join), long-context এ সব document দিয়ে দেওয়া।

### When should I use it?
Research assistant, analyst Q&A, compliance প্রশ্ন যেখানে একাধিক source জোড়া লাগে।

### When should I NOT use it?
Simple factual প্রশ্নে। Overkill আর ধীর।

### Hands-on experiment
১০টা ২-hop প্রশ্ন বানাও (ছোট fake company docs দিয়ে)। Single-shot RAG vs decompose-then-retrieve এর accuracy তুলনা করো।

### Production implementation
- Hop-by-hop trace log (প্রতিটা sub-question, retrieved ids, intermediate answer)।
- Max hop + total timeout।
- Intermediate answer এর confidence কম হলে থামো এবং বলো।

### Failure cases
Decomposition এ প্রশ্নের একটা অংশ হারিয়ে গেছে, তাই আংশিক উত্তর আত্মবিশ্বাসের সাথে।

### Security concerns
প্রতিটা hop এ একই permission। Hop এর মাধ্যমে permission ছাড়া তথ্য জোড়া লাগানো (inference attack) নিয়ে সচেতন থাকো।

### Performance concerns
Hop সংখ্যার সাথে latency linear বাড়ে।

### Cost concerns
প্রতি hop এ LLM call + retrieval।

### What did I learn?
কিছু প্রশ্নের উত্তর একবারে পাওয়া যায় না, সূত্র ধরে ধরে এগোতে হয়।

### What can I explain to someone else?
> "গোয়েন্দা গল্পের মতো: প্রথম সূত্র থেকে দ্বিতীয় সূত্রের ঠিকানা, সেখান থেকে তৃতীয়। প্রথম সূত্র না পেলে দ্বিতীয়টা খোঁজাই যায় না।"

### GitHub Evidence
- [ ] Single-shot vs decomposed 2-hop QA

### LinkedIn Post
- [ ] "কিছু প্রশ্নের উত্তর একবারে খোঁজা যায় না: multi-hop retrieval"

---

<a id="agentic-retrieval"></a>

## 6. Agentic Retrieval

### What is it?
Retrieval কে একটা fixed pipeline না রেখে **agent এর tool** বানানো। Model নিজে ঠিক করে: খোঁজা দরকার কিনা, কোন source এ (vector DB, SQL, web, file), কী query দিয়ে, ফল যথেষ্ট কিনা, আর আবার খুঁজবে কিনা। একে **Agentic RAG** বলে।

### Why does it exist?
Fixed pipeline (সবসময় top-5 আনো) সব প্রশ্নে মানায় না: কোনো প্রশ্নে retrieval লাগেই না, কোনটায় ৩টা source লাগে, কোনটায় বারবার খুঁজতে হয়।

### What problem does it solve?
Adaptive retrieval: জটিল, open-ended প্রশ্নে উচ্চ quality। Multi-hop, multi-source, self-correction সব এক loop এ।

### How does it work?
```text
tools: search_docs(query, filters) · query_db(metric, period) · web_search(q) · read_doc(id)

loop:
  model: "আগে policy docs এ খুঁজি" → search_docs(...)
  model: "এখানে ২০২৪ এর তথ্য, ২০২৬ এর লাগবে" → search_docs(..., year=2026)
  model: "এবার revenue number" → query_db(...)
  model: "যথেষ্ট তথ্য আছে" → final answer with citations
```
Phase 5 এর agent pattern (ReAct) এর উপর দাঁড়ানো।

### Important concepts
- **Retrieval-as-a-tool**: ভালো tool description, ছোট shaped output।
- **Self-reflection**: "ফল কি প্রশ্নের উত্তর দেয়?" (Self-RAG, Corrective RAG ধারণা)।
- **Budget**: max tool call, max token।
- **Router**: সহজ প্রশ্ন → সাধারণ RAG, জটিল → agentic (খরচ নিয়ন্ত্রণে)।

### Trade-offs
সবচেয়ে flexible ও শক্তিশালী, কিন্তু ধীর, দামি, কম predictable, আর debug ও eval কঠিন।

### Alternatives
Fixed pipeline + query optimization (বেশিরভাগ ক্ষেত্রে যথেষ্ট), multi-hop pipeline।

### When should I use it?
Research, analysis, অনেক source এর data, যেখানে প্রশ্নের ধরন আগে থেকে জানা নেই।

### When should I NOT use it?
High-volume, latency-sensitive, সরল FAQ/support। সেখানে fixed pipeline।

### Hands-on experiment
একই ১০টা প্রশ্ন (৫টা সহজ, ৫টা multi-hop) দুইভাবে চালাও: (১) fixed RAG, (২) agent with `search_docs` tool (max 4 calls)। Accuracy, tool call সংখ্যা, latency, token table বানাও।

### Production implementation
- Tool call trace (OpenTelemetry/Langfuse)।
- Max steps, timeout, budget (দেখো [context-optimization.md](../03-context/context-optimization.md#token-budgeting))।
- Router: classifier দিয়ে প্রশ্ন ভাগ।

### Failure cases
- Agent একই query বারবার চালাচ্ছে (loop)।
- Agent আত্মবিশ্বাসী হয়ে retrieval ছাড়াই নিজের জ্ঞান থেকে (ভুল) উত্তর দিল।

### Security concerns
Agent এর হাতে web_search + internal docs থাকলে indirect injection (web page) দিয়ে internal data বাইরে পাঠানোর চেষ্টা হতে পারে। Tool permission আর output egress control লাগে।

### Performance concerns
একাধিক LLM round-trip, তাই কয়েক সেকেন্ড থেকে মিনিট।

### Cost concerns
Fixed RAG এর ৩–১০ গুণ বা বেশি। Router দিয়ে শুধু জটিল প্রশ্নে।

### What did I learn?
Retrieval কখন, কোথায়, কতবার, এই সিদ্ধান্ত model কে দিলে flexibility বাড়ে, কিন্তু control কমে।

### What can I explain to someone else?
> "Fixed RAG হলো librarian কে বলা 'প্রথম ৫টা বই দাও'। Agentic RAG হলো একজন research assistant, যে নিজে ঠিক করে কোন তাকে যাবে, কী পড়বে, আর আরও খুঁজতে হবে কিনা।"

### GitHub Evidence
- [ ] Fixed vs agentic RAG comparison table

### LinkedIn Post
- [ ] "Agentic RAG: কখন লাগে, কখন অপচয়"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

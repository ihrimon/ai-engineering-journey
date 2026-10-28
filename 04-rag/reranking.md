# Phase 4 — RAG & Knowledge Systems: Reranking (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 🔗 Reranker এর মৌলিক ধারণা (কেন লাগে, cross-encoder কী) Phase 1 এ: [rerankers.md](../01-llm-fundamentals/rerankers.md)। এখানে **production RAG এ provider বাছাই আর pipeline design**।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Model version আর দাম দ্রুত বদলায়। নাম গুলো উদাহরণ, official docs দেখে নাও।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Cohere rerankers](#cohere-rerankers) | Managed API, multilingual, সবচেয়ে পরিচিত hosted reranker |
| [BGE rerankers](#bge-rerankers) | BAAI এর open-weight cross-encoder: self-host, free, multilingual (`bge-reranker-v2-m3`) |
| [Voyage rerankers](#voyage-rerankers) | Managed API, retrieval-specialist company, instruction-following reranker |
| [Cross-encoder reranking](#cross-encoder-reranking) | Query + doc একসাথে model এ: ধীর কিন্তু অনেক accurate relevance score |
| [Retrieval → reranking → generation](#rerank-pipeline) | বেশি আনো (recall), rerank করে কমাও (precision), সেরাটা LLM কে দাও |

```text
             Recall stage (দ্রুত, সস্তা)          Precision stage (ধীর, accurate)
question ──► hybrid search top-50 ──────────────► reranker ──► top-5 ──► LLM
             bi-encoder + BM25                    cross-encoder
             "সম্ভাব্য সব আনো"                     "আসলেই সেরা কোনগুলো?"
```

---

<a id="cohere-rerankers"></a>

## 1. Cohere Rerankers

### What is it?
Cohere এর hosted **Rerank API** (যেমন Rerank 3.5 বা নতুন version): query আর document list পাঠাও, প্রতিটার relevance score সহ sorted list ফেরত পাও। AWS Bedrock, Azure ইত্যাদিতেও পাওয়া যায়।

### Why does it exist?
নিজে reranker host করা ঝামেলা (GPU, scaling)। একটা API call দিয়ে production-quality reranking।

### What problem does it solve?
Infra ছাড়াই RAG precision বাড়ানো, multilingual (বাংলা সহ অনেক ভাষা) support সহ।

### How does it work?
```text
POST /rerank { model, query, documents: [...50 docs], top_n: 5 }
→ [{ index: 12, relevance_score: 0.94 }, { index: 3, relevance_score: 0.88 }, ...]
```

### Important concepts
- Pricing সাধারণত **per search** (একটা query + নির্দিষ্ট সংখ্যক document)।
- লম্বা document নিজে chunk করে, তাই max token per doc দেখো।
- Semi-structured data (JSON/YAML) rerank support।

### Trade-offs
Zero-ops ও ভালো quality, কিন্তু data third party তে যায়, network latency, আর per-call খরচ।

### Alternatives
Voyage (hosted), BGE/Jina (self-host), LLM-as-reranker।

### When should I use it?
দ্রুত production, GPU নেই, multilingual corpus।

### When should I NOT use it?
Data বাইরে পাঠানো নিষেধ, বা খুব বেশি QPS এ খরচ বেশি।

### Hands-on experiment
```ts
// npm i cohere-ai   (model name docs থেকে নাও)
import { CohereClientV2 } from "cohere-ai";
const co = new CohereClientV2({ token: process.env.COHERE_API_KEY });
const res = await co.rerank({ model: "rerank-v3.5", query: q, documents: candidates, topN: 5 });
console.log(res.results.map((r) => [r.index, r.relevanceScore]));
```
Rerank এর আগে আর পরে top-5 এর hit rate তুলনা করো।

### Production implementation
Timeout + fallback (rerank fail হলে original order), relevance score threshold দিয়ে খারাপ chunk বাদ।

### Failure cases
Rerank API down, আর fallback না থাকায় পুরো chatbot down।

### Security concerns
Document content provider এর কাছে যায়। Data processing agreement আর retention policy দেখো।

### Performance concerns
সাধারণত ১০০–৫০০ms (document সংখ্যা ও দৈর্ঘ্য অনুযায়ী)।

### Cost concerns
Per-search billing, query volume × দাম হিসাব করো।

### What did I learn?
Hosted reranker = সবচেয়ে কম পরিশ্রমে RAG quality বাড়ানো।

### What can I explain to someone else?
> "Search এর ৫০টা result একজন expert এর হাতে দাও, সে পড়ে বলে দেয় কোন ৫টা আসলেই প্রশ্নের উত্তর দেয়। Cohere সেই expert ভাড়া দেয়।"

### GitHub Evidence
- [ ] Cohere rerank before/after hit-rate

### LinkedIn Post
- [ ] —

---

<a id="bge-rerankers"></a>

## 2. BGE Rerankers

### What is it?
BAAI (Beijing Academy of AI) এর open-weight cross-encoder reranker family। যেমন `bge-reranker-v2-m3` (multilingual, হালকা), `bge-reranker-large`, আর LLM-based বড় variant। Hugging Face থেকে download করে self-host করা যায়।

### Why does it exist?
Open, free, নিজের infra তে চালানো যায় এমন উচ্চমানের reranker দরকার।

### What problem does it solve?
Data নিজের কাছে রেখে reranking, per-call খরচ ছাড়া।

### How does it work?
```text
input:  "[CLS] query [SEP] document [SEP]"  → transformer → single relevance score (logit)
প্রতিটা (query, doc) pair আলাদা forward pass
```
চালানোর উপায়: Python `FlagEmbedding`/`sentence-transformers`, Hugging Face TEI (Text Embeddings Inference) server, Infinity server।

### Important concepts
- **m3 version** = multilingual, বাংলা সহ।
- Score normalize করতে sigmoid।
- GPU তে অনেক দ্রুত, CPU তে ছোট model চলে কিন্তু ধীর।

### Trade-offs
Free ও private, কিন্তু hosting, scaling, GPU খরচ আর maintenance তোমার।

### Alternatives
Jina reranker, mixedbread reranker (open), Cohere/Voyage (hosted)।

### When should I use it?
Self-host requirement, high volume, বা শেখার জন্য।

### When should I NOT use it?
GPU/ops সামর্থ্য নেই আর volume কম। তখন hosted সহজ।

### Hands-on experiment
```bash
# Hugging Face TEI দিয়ে local reranker server (CPU image)
docker run -p 8080:80 ghcr.io/huggingface/text-embeddings-inference:cpu-latest --model-id BAAI/bge-reranker-v2-m3
```
```ts
const r = await fetch("http://localhost:8080/rerank", {
  method: "POST", headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ query: q, texts: candidates }),
}).then((x) => x.json());   // [{ index, score }, ...]
```
৮GB RAM এ CPU তে ২০টা doc rerank করতে কত সময় লাগে মাপো।

### Production implementation
TEI/Infinity server, GPU node, batching, health check, autoscaling।

### Failure cases
CPU তে ৫০টা লম্বা doc rerank করতে ৫ সেকেন্ড, ফলে UX নষ্ট। Candidate কমাও বা GPU।

### Security concerns
Self-host, তাই data বাইরে যায় না। Server endpoint internal network এ রাখো।

### Performance concerns
GPU ছাড়া latency বড় সমস্যা। Doc length truncate (যেমন ৫১২ token)।

### Cost concerns
Per-call free, কিন্তু GPU server এর স্থায়ী খরচ।

### What did I learn?
Open reranker = control আর privacy, বিনিময়ে ops।

### What can I explain to someone else?
> "Expert ভাড়া না করে নিজের অফিসে একজন expert নিয়োগ: মাসিক বেতন (server) দিতে হবে, কিন্তু গোপন কাগজ বাইরে যায় না।"

### GitHub Evidence
- [ ] Local bge-reranker-v2-m3 via TEI + latency

### LinkedIn Post
- [ ] —

---

<a id="voyage-rerankers"></a>

## 3. Voyage Rerankers

### What is it?
Voyage AI (এখন MongoDB এর অংশ) এর hosted reranker (যেমন `rerank-2.5`, `rerank-2.5-lite`)। Voyage embedding model এর জন্যও পরিচিত। Anthropic তাদের docs এ embedding এর জন্য Voyage recommend করে।

### Why does it exist?
Retrieval-specialist company, embedding + rerank দুটো একই vendor থেকে উচ্চ মানে দেওয়ার জন্য।

### What problem does it solve?
High-quality reranking, লম্বা context per document, আর instruction দিয়ে relevance এর সংজ্ঞা steer করা।

### How does it work?
```text
POST /v1/rerank { query, documents, model, top_k }
→ [{ index, relevance_score }]
instruction-following: query তে নির্দেশনা যোগ করা যায়, যেমন "Prefer official policy documents over forum posts"
```

### Important concepts
- **Lite vs full**: latency/cost বনাম quality।
- Token-based pricing।
- Domain-specific embedding model (code, law, finance) এর সাথে ভালো মেলে।

### Trade-offs
Cohere এর মতো: managed সুবিধা বনাম vendor dependency।

### Alternatives
Cohere, Jina (hosted/open), BGE।

### When should I use it?
Voyage embedding আগে থেকেই ব্যবহার করলে, বা instruction-steered reranking লাগলে।

### When should I NOT use it?
Self-host বাধ্যতামূলক হলে।

### Hands-on experiment
একই candidate set এ Cohere, Voyage আর local BGE তিনটা দিয়ে rerank করে nDCG@5 আর latency table বানাও (free tier/trial দিয়ে)।

### Production implementation
Provider abstraction (`Reranker` interface), যাতে বদলানো সহজ হয়, আর eval দিয়ে provider বাছাই।

### Failure cases
Instruction খুব কঠোর ("শুধু ২০২৬"), ফলে relevant কিন্তু পুরনো doc এর score শূন্য।

### Security concerns
Third-party data processing। Policy দেখো।

### Performance concerns
Lite model দ্রুত।

### Cost concerns
Token-based, তাই doc length truncate করে খরচ নিয়ন্ত্রণ।

### What did I learn?
Reranker ও একটা swappable component। Interface বানাও, eval দিয়ে বাছো।

### What can I explain to someone else?
> "আরেকজন expert, যাকে বলে দেওয়া যায় 'অফিসিয়াল কাগজকে বেশি গুরুত্ব দাও'।"

### GitHub Evidence
- [ ] Reranker provider comparison (nDCG@5 + latency + cost)

### LinkedIn Post
- [ ] "৩টা reranker, এক eval: কোনটা জিতল?"

---

<a id="cross-encoder-reranking"></a>

## 4. Cross-Encoder Reranking

### What is it?
এমন model যেটা **query আর document একসাথে** input হিসেবে নিয়ে একটা relevance score দেয়। Bi-encoder (embedding) এর বিপরীতে, যেখানে দুটো আলাদা encode হয়।

### Why does it exist?
Bi-encoder দ্রুত কিন্তু query আর doc এর শব্দের সূক্ষ্ম interaction দেখে না (আগে থেকে আলাদা vector)। Cross-encoder এ attention দুইটার প্রতিটা token এর মধ্যে কাজ করে, তাই অনেক বেশি accurate।

### What problem does it solve?
"দেখতে কাছাকাছি কিন্তু আসলে উত্তর না" এমন chunk আলাদা করা, যেমন negation, নির্দিষ্ট শর্ত, সময়।

### How does it work?
```text
Bi-encoder:     E(q) · E(d)          → আগে থেকে index, O(1) per doc at query time
Cross-encoder:  M([q ; d]) → score   → প্রতিটা pair এ পূর্ণ forward pass, index করা যায় না
তাই: bi-encoder দিয়ে ১০ লাখ থেকে ৫০টা, cross-encoder দিয়ে ৫০ থেকে ৫টা
```

### Important concepts
- **Cannot pre-compute**: প্রতিটা query তে নতুন করে চলে, তাই candidate সংখ্যা সীমিত রাখো।
- **LLM-as-reranker**: LLM কে list দিয়ে rank করতে বলা (listwise/pointwise), যা শক্তিশালী কিন্তু ধীর ও দামি।
- **ColBERT (late interaction)**: মাঝামাঝি, token-level vector আগে থেকে রেখে দ্রুত interaction।

### Trade-offs
Accuracy ↑↑, latency ↑ (candidate সংখ্যার সাথে linear)।

### Alternatives
ColBERT, LLM reranking, শুধু hybrid (reranker ছাড়া)।

### When should I use it?
Precision গুরুত্বপূর্ণ, top-k ছোট রাখতে চাও (context খরচ কমাতে)।

### When should I NOT use it?
Ultra-low latency (<১০০ms মোট) বা candidate হাজার হাজার।

### Hands-on experiment
একটা tricky set বানাও: "কোন plan এ refund **নেই**?" প্রশ্নে "Pro plan এ refund আছে" আর "Free plan এ refund নেই" দুইটা chunk। Embedding score আর cross-encoder score তুলনা করো।

### Production implementation
Candidate N (২০–১০০) আর output top-n (৩–৮) eval দিয়ে tune, doc truncate, batching।

### Failure cases
২০০ candidate rerank করায় p95 latency ৩ সেকেন্ড।

### Security concerns
Reranker নিজে injection প্রতিরোধ করে না। Malicious doc ও "relevant" score পেতে পারে।

### Performance concerns
Latency ∝ candidate সংখ্যা × doc length।

### Cost concerns
Hosted এ per-search, self-host এ GPU। কিন্তু ছোট top-n মানে LLM context খরচ কমে, যা প্রায়ই লাভজনক।

### What did I learn?
দ্রুত কিন্তু মোটা ছাঁকনি (bi-encoder) আর ধীর কিন্তু সূক্ষ্ম ছাঁকনি (cross-encoder), দুই স্তরে ছাঁকো।

### What can I explain to someone else?
> "চাকরির ১০০০ আবেদন থেকে CV দেখে ৫০ জন বাছাই (bi-encoder), তারপর ৫০ জনের সাথে সরাসরি interview নিয়ে ৫ জন (cross-encoder)।"

### GitHub Evidence
- [ ] Negation test: embedding vs cross-encoder score

### LinkedIn Post
- [ ] "Bi-encoder vs cross-encoder: CV বাছাই বনাম interview"

---

<a id="rerank-pipeline"></a>

## 5. Retrieval → Reranking → Generation

### What is it?
Production RAG এর পূর্ণ three-stage pipeline: **retrieve (recall) → rerank (precision) → generate (answer with citations)**, প্রতিটা ধাপের সংখ্যা, threshold আর fallback সহ।

### Why does it exist?
একটা ধাপ দিয়ে সব লক্ষ্য (দ্রুত, সম্পূর্ণ, সঠিক, সস্তা) পূরণ হয় না। ধাপে ভাগ করলে প্রতিটা ধাপ নিজের কাজে optimize করা যায়।

### What problem does it solve?
High recall + high precision + ছোট context = ভালো উত্তর, কম খরচ, কম hallucination।

### How does it work?
```text
1. Query optimize       rewrite (multi-turn এ)
2. Retrieve             hybrid: dense 50 + BM25 50 → RRF → top 40     [ACL filter]
3. Rerank               cross-encoder → top 5, score < 0.3 বাদ
4. Assemble context     <doc id source page>…</doc>, সেরাটা শুরুতে
5. Generate             "শুধু context থেকে উত্তর দাও, cite করো, না থাকলে বলো জানি না"
6. Post-process         citation validate (cited id গুলো আসলেই context এ ছিল কিনা)
```

### Important concepts
- **Recall@N stage 2 তে** আর **Precision@n stage 3 তে** আলাদা মাপো, যাতে বোঝা যায় কোন ধাপ দুর্বল।
- **Empty-context path**: rerank এর পরে কিছু না থাকলে LLM কে বানাতে না দিয়ে "পাওয়া যায়নি"।
- **Citations**: chunk id দিয়ে।
- **Fallbacks**: reranker fail → retrieval order।

### Trade-offs
Quality সর্বোচ্চ, কিন্তু প্রতিটা ধাপ latency আর জটিলতা যোগ করে।

### Alternatives
Retrieve → generate (reranker ছাড়া, সহজ), long-context (retrieval ছাড়া, ছোট corpus এ), agentic RAG।

### When should I use it?
Production RAG এর standard architecture।

### When should I NOT use it?
১০–২০ পাতার corpus। সেখানে পুরোটা context এ দাও।

### Hands-on experiment
```ts
async function answer(q: string, user: User) {
  const [dense, sparse] = await Promise.all([denseSearch(q, user, 50), bm25Search(q, user, 50)]);
  const candidates = rrf([dense, sparse]).slice(0, 40).map(([id]) => chunkById(id));
  const ranked = (await rerank(q, candidates)).filter((r) => r.score >= 0.3).slice(0, 5);
  if (!ranked.length) return { answer: "এ বিষয়ে তথ্য পাওয়া যায়নি।", citations: [] };
  const context = ranked.map((c) => `<doc id="${c.id}" source="${c.source}">\n${c.text}\n</doc>`).join("\n");
  return generateWithCitations(q, context);
}
```
প্রতিটা stage এর latency আলাদা log করো, আর ৩০ প্রশ্নের eval এ stage-wise metric বের করো।

### Production implementation
- Stage-wise tracing (span per stage)।
- Config: N, n, threshold, retriever weights, version করা।
- Offline eval CI তে (Phase 9)।

### Failure cases
- Retrieval এ সঠিক chunk আসেনি (recall সমস্যা), কিন্তু team reranker বদলাতে ব্যস্ত (ভুল ধাপ optimize)।
- Model context এর বাইরে থেকে উত্তর দিয়ে ভুল citation দিয়েছে।

### Security concerns
ACL stage 2 তে। Context এ doc = untrusted data (tag + instruction)। Citation link এ শুধু allowlisted URL।

### Performance concerns
Typical budget: retrieve ৫০–১৫০ms, rerank ১০০–৪০০ms, generate বাকি। Stream করো।

### Cost concerns
Reranker দিয়ে context ৪০ → ৫ chunk করলে LLM input খরচ অনেক কমে, প্রায়ই reranker এর খরচের চেয়ে বেশি সাশ্রয়।

### What did I learn?
RAG একটা pipeline, প্রতিটা ধাপ আলাদা মাপতে হয়, নাহলে ভুল জায়গা ঠিক করবে।

### What can I explain to someone else?
> "জাল ফেলে অনেক মাছ ধরো (retrieve), হাতে বেছে ভালোগুলো রাখো (rerank), তারপর রান্না করো (generate)। রান্না খারাপ হলে আগে দেখো ভালো মাছ জালে উঠেছিল কিনা।"

### GitHub Evidence
- [ ] Full 3-stage pipeline + stage-wise metrics

### LinkedIn Post
- [ ] "Production RAG এর ৩ ধাপ: retrieve, rerank, generate"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

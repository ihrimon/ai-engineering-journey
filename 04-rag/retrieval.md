# Phase 4 — RAG & Knowledge Systems: Retrieval (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Code example গুলো TypeScript এ (local: Ollama `bge-m3`)। Number গুলো **উদাহরণ**, নিজের eval set দিয়ে measure করবে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Dense retrieval](#dense-retrieval) | Embedding vector এর similarity দিয়ে খোঁজা: অর্থ বোঝে |
| [Sparse retrieval](#sparse-retrieval) | শব্দভিত্তিক sparse vector (বেশিরভাগ ঘর ০): exact term ধরে |
| [BM25](#bm25) | সবচেয়ে জনপ্রিয় keyword ranking formula: TF + IDF + length normalization |
| [Hybrid search](#hybrid-search) | Dense + sparse একসাথে, তারপর ফল মেলানো |
| [Reciprocal Rank Fusion (RRF)](#rrf) | Score না, **rank** দিয়ে একাধিক list মেলানোর সহজ formula |
| [Metadata filtering](#metadata-filtering) | Retrieval এর সময় structured condition (tenant, date, type) |
| [Structured retrieval](#structured-retrieval) | SQL/API/table থেকে exact data আনা (text-to-SQL, tool) |

```text
            question
               │
   ┌───────────┼─────────────┬──────────────────┐
 Dense      Sparse/BM25   Structured (SQL)   Metadata filter (সবগুলোতে)
   │           │             │
   └──── RRF ──┘             │
        │                    │
    top-k chunks ──► rerank ─┴─► context ──► LLM
```

**Retrieval metric গুলো মনে রাখো:** Recall@k (সঠিক chunk top-k তে এসেছে কিনা), Precision@k, MRR (প্রথম সঠিকটা কত নম্বরে), nDCG। বিস্তারিত Phase 9।

---

<a id="dense-retrieval"></a>

## 1. Dense Retrieval

### What is it?
Query আর document দুটোকেই embedding model দিয়ে dense vector (যেমন ১০২৪ dimension, প্রায় সব ঘরে মান আছে) এ রূপান্তর করে, vector similarity দিয়ে কাছের document খোঁজা। একে **bi-encoder** retrieval বলে (query আর doc আলাদাভাবে encode হয়)।

### Why does it exist?
Keyword search এ "টাকা ফেরত" আর "refund" আলাদা শব্দ। Dense embedding অর্থের স্তরে মেলায়।

### What problem does it solve?
Synonym, paraphrase, multilingual matching, আর প্রশ্নের ধরন আলাদা হলেও অর্থ মিললে খুঁজে পাওয়া।

### How does it work?
```text
indexing: doc chunk → embed → vector DB
query:    question → embed (same model) → ANN top-k
```

### Important concepts
- **Bi-encoder** (দ্রুত, আগে থেকে index করা যায়) বনাম **cross-encoder** (reranker, ধীর কিন্তু accurate)।
- **Domain mismatch**: general model নির্দিষ্ট domain jargon এ দুর্বল হতে পারে।
- **Multilingual model** (bge-m3, multilingual-e5) দিয়ে বাংলা প্রশ্নে English doc।
- **Query instruction/prefix** কিছু model এ লাগে।

### Trade-offs
অর্থ বোঝে, কিন্তু exact term, rare entity, নতুন শব্দ, number এ দুর্বল। আর কেন একটা doc এলো সেটা ব্যাখ্যা করা কঠিন।

### Alternatives
BM25, hybrid, fine-tuned embedding model।

### When should I use it?
Natural-language প্রশ্ন, FAQ, support, multilingual।

### When should I NOT use it?
একা ব্যবহার করবে না code/ID/SKU-heavy corpus এ।

### Hands-on experiment
```ts
import ollama from "ollama";
const docs = ["রিফান্ড ৭ দিনের মধ্যে করা যায়", "Shipping takes 3-5 days", "ERR_4471 means invalid token"];
const { embeddings: D } = await ollama.embed({ model: "bge-m3", input: docs });
const { embeddings: [q] } = await ollama.embed({ model: "bge-m3", input: "How do I get my money back?" });
const scored = D.map((d, i) => ({ doc: docs[i], score: cosine(q, d) })).sort((a, b) => b.score - a.score);
console.table(scored);
```
English প্রশ্নে বাংলা refund doc প্রথমে আসে কিনা দেখো। তারপর "ERR_4471" দিয়ে খোঁজো।

### Production implementation
Query embedding cache, batch embedding এ indexing, embedding model version metadata তে।

### Failure cases
"ERR_4471" query তে অন্য error code এর doc আসে কারণ vector এ সংখ্যাগুলো "কাছাকাছি"।

### Security concerns
Embedding inversion: vector থেকে আংশিক text পুনরুদ্ধার সম্ভব, তাই vector কে sensitive data হিসেবে রক্ষা করো।

### Performance concerns
Query embed (~১০–১০০ms) + ANN search।

### Cost concerns
Indexing এ একবার embedding খরচ, query তে সামান্য।

### What did I learn?
Dense retrieval অর্থের জন্য দারুণ, কিন্তু একা যথেষ্ট না।

### What can I explain to someone else?
> "প্রতিটা লেখাকে মানচিত্রে একটা বিন্দু বানাও যেখানে একই অর্থের লেখা কাছাকাছি থাকে। প্রশ্নের বিন্দুর পাশের বিন্দুগুলোই উত্তর।"

### GitHub Evidence
- [ ] Dense retrieval demo (bn ↔ en)

### LinkedIn Post
- [ ] —

---

<a id="sparse-retrieval"></a>

## 2. Sparse Retrieval

### What is it?
Text কে **sparse vector** এ represent করা: vocabulary এর প্রতিটা শব্দ একটা dimension, বেশিরভাগ মান ০, শুধু উপস্থিত শব্দগুলোর মান আছে। Classic: TF-IDF, BM25। Learned sparse: SPLADE, bge-m3 এর sparse output (model শেখে কোন শব্দ কতটা গুরুত্বপূর্ণ, এমনকি সম্পর্কিত শব্দও যোগ করে)।

### Why does it exist?
Exact keyword match দ্রুত, ব্যাখ্যাযোগ্য, আর rare term (নাম, code) এ dense এর চেয়ে ভালো।

### What problem does it solve?
Product code, error code, নাম, acronym, legal term এর মতো exact-term query।

### How does it work?
```text
"refund policy 2026" → {refund: 2.1, policy: 1.3, 2026: 3.4}  (বাকি লাখো শব্দ = ০)
inverted index: শব্দ → কোন কোন doc এ আছে → score যোগ করে rank
```

### Important concepts
- **Inverted index**: খুব দ্রুত lookup।
- **Tokenization/stemming** ভাষা-নির্ভর। বাংলায় ভালো analyzer কম, যা একটা চ্যালেঞ্জ।
- **Learned sparse** = keyword এর ব্যাখ্যাযোগ্যতা + কিছুটা semantic expansion।

### Trade-offs
Exact ও ব্যাখ্যাযোগ্য, কিন্তু synonym/paraphrase বোঝে না (classic sparse)।

### Alternatives
Dense, hybrid।

### When should I use it?
Hybrid এর অংশ হিসেবে প্রায় সবসময়। একা: log search, code search, catalog।

### When should I NOT use it?
একা conversational/multilingual প্রশ্নে।

### Hands-on experiment
একই ১০ প্রশ্নে TF-IDF (নিজে লেখা) আর dense এর ফল পাশাপাশি রাখো। কোন প্রশ্নে কোনটা জেতে, label করো।

### Production implementation
Elasticsearch/OpenSearch, Postgres `tsvector`, বা vector DB এর sparse vector।

### Failure cases
বাংলা text whitespace tokenizer এ ঠিক ভাঙেনি ("রিফান্ডের" ≠ "রিফান্ড"), তাই match নেই।

### Security concerns
Dense এর মতোই ACL filter।

### Performance concerns
Inverted index খুব দ্রুত আর memory-efficient।

### Cost concerns
Classic sparse এ কোনো model খরচ নেই।

### What did I learn?
Sparse = "শব্দ মিলাও"। পুরনো, কিন্তু এখনো অপরিহার্য।

### What can I explain to someone else?
> "বইয়ের শেষে যে index থাকে, 'refund – পৃষ্ঠা ১২, ৪৫'। Sparse retrieval ঠিক সেই index ব্যবহার করে।"

### GitHub Evidence
- [ ] TF-IDF from scratch vs dense

### LinkedIn Post
- [ ] —

---

<a id="bm25"></a>

## 3. BM25

### What is it?
**Best Matching 25**: keyword relevance ranking formula, Elasticsearch/Lucene/OpenSearch এর default। TF-IDF এর উন্নত রূপ।

### Why does it exist?
Raw term frequency এ সমস্যা: একটা শব্দ ১০০ বার থাকলে ১০০ গুণ relevant না, আর লম্বা document স্বাভাবিকভাবে বেশি শব্দ ধারণ করে।

### What problem does it solve?
Keyword match কে ভালোভাবে rank করে: rare শব্দ বেশি গুরুত্ব, repetition এর লাভ saturate হয়, লম্বা doc এর সুবিধা normalize হয়।

### How does it work?
```text
score(D, Q) = Σ  IDF(q) × [ f(q,D) × (k1 + 1) ] / [ f(q,D) + k1 × (1 − b + b × |D|/avgdl) ]
             q∈Q
IDF(q)  : শব্দটা কত কম document এ আছে (rare = বেশি মূল্য)
f(q,D)  : D তে q কতবার আছে (k1 দিয়ে saturate, সাধারণত 1.2–2.0)
b       : length normalization (সাধারণত 0.75)
```

### Important concepts
- **TF saturation**: ৩ বার আর ৩০ বার এর পার্থক্য ছোট।
- **IDF**: "the", "এবং" এর মতো common শব্দের মূল্য প্রায় ০।
- **k1, b tuning** domain অনুযায়ী।
- **Strong baseline**: অনেক benchmark এ BM25 এখনো কঠিন প্রতিযোগী।

### Trade-offs
দ্রুত, training-free, ব্যাখ্যাযোগ্য। কিন্তু অর্থ বোঝে না।

### Alternatives
SPLADE (learned sparse), dense।

### When should I use it?
Hybrid এর sparse অংশ হিসেবে, আর প্রথম baseline হিসেবে।

### When should I NOT use it?
একা paraphrase-heavy বা cross-lingual search এ।

### Hands-on experiment
```ts
// npm i wink-bm25-text-search   (বা নিজে implement করো, শেখার জন্য ভালো)
function bm25Score(qTerms: string[], doc: string[], df: Map<string, number>, N: number, avgdl: number, k1 = 1.5, b = 0.75) {
  return qTerms.reduce((s, t) => {
    const f = doc.filter((w) => w === t).length;
    if (!f) return s;
    const idf = Math.log(1 + (N - (df.get(t) ?? 0) + 0.5) / ((df.get(t) ?? 0) + 0.5));
    return s + idf * ((f * (k1 + 1)) / (f + k1 * (1 - b + (b * doc.length) / avgdl)));
  }, 0);
}
```
নিজে implement করে ছোট corpus এ চালাও, `k1`/`b` বদলে ranking কীভাবে বদলায় দেখো।

### Production implementation
OpenSearch/Elasticsearch, বা Postgres FTS (`ts_rank` BM25 না, কিন্তু কাছাকাছি, আর ParadeDB/pg_search এ BM25)।

### Failure cases
Stopword list না থাকায় বাংলা "এবং", "ও" শব্দ score এ প্রভাব ফেলছে।

### Security concerns
নেই বিশেষ।

### Performance concerns
খুব দ্রুত।

### Cost concerns
সস্তা।

### What did I learn?
BM25 = ৩০ বছরের পুরনো, কিন্তু এখনো hybrid search এর অর্ধেক।

### What can I explain to someone else?
> "যে শব্দ সব বইয়ে আছে তার মূল্য কম, যে শব্দ কম বইয়ে আছে সেটা মিললে বড় ব্যাপার। আর একই শব্দ বারবার বললে লাভ একটা সময় পর আর বাড়ে না।"

### GitHub Evidence
- [ ] BM25 from scratch (TypeScript)

### LinkedIn Post
- [ ] "BM25: ৩০ বছরের পুরনো algorithm কেন AI যুগেও জরুরি"

---

<a id="hybrid-search"></a>

## 4. Hybrid Search

### What is it?
Dense আর sparse retrieval দুটো চালিয়ে ফলাফল একত্র করা। Retrieval quality এর দিক থেকে এখানে দেখছি। DB implementation: [vector-databases.md](vector-databases.md#hybrid-search)।

### Why does it exist?
Dense আর sparse এর দুর্বলতা পরস্পরের পরিপূরক। একটা যা miss করে, অন্যটা প্রায়ই ধরে।

### What problem does it solve?
Recall বাড়ায়, বিশেষ করে mixed query তে ("ERR_4471 হলে কী করব?")।

### How does it work?
```text
dense top-50   ─┐
                ├─ fusion (RRF বা weighted normalized score) ─► top-20 ─► reranker ─► top-5
BM25 top-50    ─┘
```

### Important concepts
- **Fusion method**: RRF (সহজ, robust) বা weighted score (`α·dense + (1−α)·sparse`, normalize করে)।
- **Candidate depth**: প্রতিটা থেকে বেশি নাও (৫০), fusion এর পরে কমাও।
- **Reranker** প্রায়ই hybrid এর পরে বসে।

### Trade-offs
ভালো recall, কিন্তু দুই pipeline, বেশি latency আর tuning।

### Alternatives
Dense + reranker (অনেক ক্ষেত্রে যথেষ্ট), query routing (keyword-like query → BM25, প্রশ্ন → dense)।

### When should I use it?
Production RAG এর default হিসেবে, বিশেষ করে technical content এ।

### When should I NOT use it?
খুব ছোট corpus বা latency budget খুব কঠোর হলে, তখন eval দেখে সিদ্ধান্ত নাও।

### Hands-on experiment
Dense, BM25, hybrid (RRF) তিনটার Recall@5 আর MRR একটা ৩০-প্রশ্নের eval set এ মাপো।

### Production implementation
Dense আর BM25 `Promise.all` এ parallel, তারপর RRF, তারপর rerank।

### Failure cases
Weighted fusion এ BM25 score normalize না করায় BM25 সব কিছু dominate করছে।

### Security concerns
দুই path এই একই ACL filter, একটায় ভুলে গেলেই leak।

### Performance concerns
Parallel চালালে latency ≈ ধীরটার সমান।

### Cost concerns
দ্বিতীয় index এর storage আর query compute।

### What did I learn?
Hybrid হলো retrieval এর "belt and suspenders"।

### What can I explain to someone else?
> "দুইজন গোয়েন্দা আলাদা পদ্ধতিতে সন্দেহভাজনের তালিকা বানায়। দুই তালিকাতেই যার নাম উপরের দিকে, সে-ই প্রধান সন্দেহভাজন।"

### GitHub Evidence
- [ ] Dense vs BM25 vs hybrid Recall@5/MRR table

### LinkedIn Post
- [ ] "Hybrid search: RAG এর recall বাড়ানোর সবচেয়ে সহজ উপায়"

---

<a id="rrf"></a>

## 5. Reciprocal Rank Fusion (RRF)

### What is it?
একাধিক ranked list কে একত্র করার formula, যেটা score না দেখে শুধু **rank (position)** দেখে:
```text
RRF(d) = Σ  1 / (k + rank_i(d))       k সাধারণত 60
        lists
```
(Cormack et al., 2009)

### Why does it exist?
BM25 score (যেমন 12.7) আর cosine score (যেমন 0.83) আলাদা scale এ। সরাসরি যোগ করা অর্থহীন। Normalize করাও fragile।

### What problem does it solve?
Score scale নিয়ে চিন্তা ছাড়াই যেকোনো retriever এর ফল মেলানো, tuning প্রায় লাগে না।

### How does it work?
```text
doc X: dense rank 1, BM25 rank 3  → 1/61 + 1/63 = 0.0323
doc Y: dense rank 2, BM25 না       → 1/62        = 0.0161
doc Z: BM25 rank 1, dense rank 10 → 1/61 + 1/70 = 0.0307
→ X > Z > Y   (দুই list এ ভালো করা doc জেতে)
```

### Important concepts
- **k** বড় হলে নিচের rank এর গুরুত্ব তুলনামূলক বাড়ে। ৬০ একটা robust default।
- **Weighted RRF**: list অনুযায়ী weight দেওয়া যায়।
- Multi-query retrieval এও ব্যবহার হয় (একাধিক query variant এর ফল মেলাতে)।

### Trade-offs
সহজ ও robust, কিন্তু score এর "কতটা ভালো" তথ্য হারায় (rank 1 আর 2 এর মাঝে বিশাল gap থাকলেও বোঝে না)।

### Alternatives
Weighted normalized score fusion, learned fusion (learning-to-rank)।

### When should I use it?
Hybrid search, multi-query, multi-index fusion এর default।

### When should I NOT use it?
যখন score calibrated আর একটা retriever স্পষ্টভাবে বেশি নির্ভরযোগ্য, তখন weighted fusion ভালো হতে পারে।

### Hands-on experiment
```ts
function rrf(lists: string[][], k = 60) {
  const scores = new Map<string, number>();
  for (const list of lists) list.forEach((id, i) => scores.set(id, (scores.get(id) ?? 0) + 1 / (k + i + 1)));
  return [...scores.entries()].sort((a, b) => b[1] - a[1]);
}
console.log(rrf([["X", "Y", "W"], ["Z", "A", "X"]]));
```
`k` = 1, 10, 60, 200 দিয়ে final order কীভাবে বদলায় দেখো।

### Production implementation
Qdrant/Weaviate/Elasticsearch এ native RRF আছে, নিজে লিখতে হলে উপরের ১০ লাইন।

### Failure cases
একটা list এ duplicate id (overlapping chunk), ফলে double counting। আগে dedupe করো।

### Security concerns
নেই।

### Performance concerns
নগণ্য।

### Cost concerns
নগণ্য।

### What did I learn?
কখনো কখনো সবচেয়ে সহজ formula ই সবচেয়ে robust।

### What can I explain to someone else?
> "দুইজন বিচারক আলাদা নম্বর দেন, একজন ১০ এর মধ্যে, আরেকজন ১০০ এর মধ্যে। নম্বর যোগ না করে শুধু দেখো কে কাকে কত নম্বর অবস্থানে রেখেছে।"

### GitHub Evidence
- [ ] RRF implementation + k sweep

### LinkedIn Post
- [ ] "RRF: ১০ লাইনের algorithm যেটা hybrid search চালায়"

---

<a id="metadata-filtering"></a>

## 6. Metadata Filtering

### What is it?
Retrieval এর সময় structured condition প্রয়োগ। DB mechanics (pre/post filter): [vector-databases.md](vector-databases.md#metadata-filtering)। এখানে দেখছি **retrieval strategy** হিসেবে: কখন filter, কে filter বানায়, আর **self-query** (প্রশ্ন থেকে filter বের করা)।

### Why does it exist?
"২০২৫ সালের Q3 report এ revenue কত?" প্রশ্নে "২০২৫" আর "Q3" semantic search এ দুর্বলভাবে ধরা পড়ে। Structured filter হিসেবে দিলে নিখুঁত।

### What problem does it solve?
Time-scoped, entity-scoped, permission-scoped retrieval।

### How does it work?
```text
question: "২০২৫ এর Q3 report এ revenue কত?"
   → LLM (structured output): { filters: { year: 2025, quarter: "Q3", doc_type: "report" }, query: "revenue" }
   → validate (allowed fields + values) → + mandatory tenant/ACL filter (code থেকে)
   → filtered search
```

### Important concepts
- **Two kinds**: security filter (code, mandatory) আর relevance filter (LLM/user, optional)।
- **Validation**: LLM এর বানানো filter শুধু allowlisted field এ।
- **Fallback**: filter দিয়ে ০ result এলে relax করে আবার।

### Trade-offs
Precise, কিন্তু LLM ভুল filter বানালে (ভুল বছর) সঠিক doc বাদ পড়ে।

### Alternatives
শুধু semantic + reranker, UI তে user নিজে filter বাছাই।

### When should I use it?
Report, versioned policy, catalog, যেখানে সময়/entity গুরুত্বপূর্ণ।

### When should I NOT use it?
Metadata অসম্পূর্ণ বা অনির্ভরযোগ্য হলে।

### Hands-on experiment
২০টা প্রশ্নে LLM self-query extraction এর accuracy মাপো (expected filter vs extracted)। তারপর filter সহ ও ছাড়া retrieval তুলনা করো।

### Production implementation
```ts
const FilterSchema = z.object({
  year: z.number().int().min(2015).max(2030).optional(),
  doc_type: z.enum(["report", "policy", "faq"]).optional(),
});
// LLM output → FilterSchema.parse → securityFilter(user) এর সাথে merge
```

### Failure cases
"গত বছর" বুঝতে গিয়ে LLM training cutoff এর বছর ধরে নিয়েছে। Context এ আজকের তারিখ দাও।

### Security concerns
LLM-generated filter কখনো `tenant_id`/ACL override করতে পারবে না। "সব tenant এর data দেখাও" ধরনের injection কে schema দিয়ে আটকাও।

### Performance concerns
Self-query = extra LLM call (ছোট model দিয়ে করো)।

### Cost concerns
Extra ছোট LLM call প্রতি query তে।

### What did I learn?
Filter দুই রকম: যেটা নিরাপত্তা দেয় (code) আর যেটা relevance দেয় (LLM)। দুটো কখনো মেশাবে না।

### What can I explain to someone else?
> "Online shopping এ 'দাম ১০০০ এর নিচে, ব্র্যান্ড X' filter দিয়ে খোঁজা। AI তোমার প্রশ্ন থেকে নিজেই filter বসিয়ে দেয়, কিন্তু কোন দোকানে ঢুকতে পারবে সেটা দোকানের দারোয়ান ঠিক করে।"

### GitHub Evidence
- [ ] Self-query filter extraction + validation

### LinkedIn Post
- [ ] —

---

<a id="structured-retrieval"></a>

## 7. Structured Retrieval

### What is it?
Unstructured text এর বদলে **structured source** থেকে data আনা: SQL database, spreadsheet, API, knowledge graph। সাধারণত text-to-SQL বা predefined tool (`get_order(id)`, `revenue_by_month(year)`) দিয়ে।

### Why does it exist?
"গত মাসে কত order হয়েছে?" এর উত্তর কোনো document এ নেই। এটা DB তে, aggregation লাগে। Vector search এখানে অর্থহীন।

### What problem does it solve?
Exact, fresh, aggregatable data দিয়ে উত্তর, number এ hallucination ছাড়া।

### How does it work?
```text
question → router: unstructured? structured? দুটোই?
structured: LLM → SQL (read-only, schema দিয়ে) → validate → execute → rows → LLM উত্তর
        বা: LLM → tool call get_sales(month) → typed result
```

### Important concepts
- **Predefined tools > free text-to-SQL** production এ (নিরাপদ, testable)।
- **Schema context**: table/column description, sample row, relationship।
- **Read-only DB user**, row limit, timeout।
- **Semantic layer**: business metric (revenue, active user) এর নির্দিষ্ট সংজ্ঞা।

### Trade-offs
Exact আর fresh, কিন্তু text-to-SQL ভুল query লিখতে পারে, আর security ঝুঁকি বেশি।

### Alternatives
Data কে text এ রূপান্তর করে RAG (stale, aggregation দুর্বল), BI dashboard।

### When should I use it?
Number, aggregation, live status (order, inventory, metrics)।

### When should I NOT use it?
Policy/explanation ধরনের প্রশ্নে, যা document এ আছে।

### Hands-on experiment
SQLite এ একটা ছোট `orders` table বানাও। (১) Free text-to-SQL, (২) ৩টা predefined tool, দুইভাবে ১৫টা প্রশ্নের accuracy তুলনা করো।

### Production implementation
- Read-only replica + restricted user + `statement_timeout` + `LIMIT`।
- SQL AST parse করে শুধু `SELECT` allow।
- Tenant filter query তে inject করো (বা RLS)।

### Failure cases
- LLM `JOIN` ভুল করে double counting, ফলে revenue দ্বিগুণ।
- Column নাম অস্পষ্ট (`amt`, `amt2`), তাই ভুল column।

### Security concerns
**SQL injection via LLM**: user প্রশ্নের মাধ্যমে `DROP TABLE` বা অন্য tenant এর data। Read-only credential, allowlist, RLS বাধ্যতামূলক।

### Performance concerns
খারাপ query (full scan) DB কে ধীর করতে পারে। Timeout ও replica।

### Cost concerns
SQL generation এ LLM call, তবে উত্তর ছোট।

### What did I learn?
সব retrieval vector search না। সংখ্যার প্রশ্নের উত্তর DB তে।

### What can I explain to someone else?
> "লাইব্রেরিতে গল্প খুঁজতে বইয়ের তাক, কিন্তু হিসাব জানতে accountant এর খাতা। AI কে জানতে হবে কোন প্রশ্নে কোথায় যেতে হবে।"

### GitHub Evidence
- [ ] Text-to-SQL vs predefined tools accuracy

### LinkedIn Post
- [ ] "সব RAG এ vector DB লাগে না: structured retrieval"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

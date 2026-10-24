# Phase 4 — RAG & Knowledge Systems: Vector Databases (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Vector DB গুলো খুব দ্রুত feature যোগ করে (hybrid search, reranking, serverless)। এখানে লেখা feature/limit গুলো **সাধারণ চিত্র**। সিদ্ধান্ত নেওয়ার আগে official docs আর নিজের benchmark দেখবে।

## Quick Cheat Sheet

| DB | ধরন | এক লাইনে পরিচয় |
|---|---|---|
| [Qdrant](#qdrant) | Open-source (Rust) + Cloud | দ্রুত, শক্তিশালী payload filter, hybrid (sparse+dense), self-host সহজ |
| [Pinecone](#pinecone) | Fully managed (closed) | Zero-ops serverless, namespace, enterprise-ready, কিন্তু vendor lock-in |
| [Weaviate](#weaviate) | Open-source (Go) + Cloud | Built-in hybrid (BM25+vector), vectorizer module, multi-tenancy |
| [pgvector](#pgvector) | Postgres extension | আগের Postgres এ vector: SQL join, transaction, এক DB |
| [Chroma](#chroma) | Open-source, developer-first | Prototype/local এর জন্য সবচেয়ে সহজ |
| [Milvus](#milvus) | Open-source distributed + Zilliz Cloud | বিলিয়ন-স্কেল, অনেক index type, GPU support |

**Understand:** [Indexing](#indexing) · [Similarity search](#similarity-search) · [Metadata filtering](#metadata-filtering) · [Hybrid search](#hybrid-search) · [Collection design](#collection-design)

```text
            Vector DB বাছাই — একটা সহজ decision tree
                              │
        আগে থেকেই Postgres আছে, < কয়েক কোটি vector? ── হ্যাঁ ──► pgvector
                              │ না
        Ops করতে চাও না, budget আছে? ───────────────── হ্যাঁ ──► Pinecone (বা managed Qdrant/Weaviate)
                              │ না
        বিলিয়ন-স্কেল / distributed? ─────────────────── হ্যাঁ ──► Milvus
                              │ না
        Self-host, filter + hybrid দরকার ──► Qdrant / Weaviate     শুধু prototype ──► Chroma
```

---

# Part A — Vector Databases

<a id="qdrant"></a>

## 1. Qdrant

### What is it?
Rust এ লেখা open-source vector database, self-host (Docker) বা Qdrant Cloud। Data model: **collection → points** (id + vector + JSON payload)।

### Why does it exist?
Production-grade vector search, শক্তিশালী filtering সহ, যেটা self-host করা সহজ আর দ্রুত।

### What problem does it solve?
High-performance ANN search + জটিল payload filter একসাথে, আর data নিজের infra তে রাখা।

### How does it work?
HNSW index, filter-aware search (filter আর vector search একসাথে), payload index। **Named vectors** (একটা point এ একাধিক vector) আর **sparse vectors** (BM25/SPLADE ধরনের), যা দিয়ে hybrid search হয়। Quantization (scalar/binary) দিয়ে memory কমানো যায়।

### Important concepts
- Payload index না বানালে filter ধীর।
- Multitenancy: একটা collection + `tenant_id` payload (indexed) সাধারণত বহু collection এর চেয়ে ভালো।
- Query API তে prefetch + fusion (RRF) দিয়ে hybrid।

### Trade-offs
Self-host এ ops (backup, scaling) তোমার। Managed cloud এ খরচ।

### Alternatives
Weaviate (similar), pgvector (সহজ stack), Pinecone (managed)।

### When should I use it?
Self-host চাও, filter-heavy query, hybrid search, ভালো performance।

### When should I NOT use it?
আগে থেকে Postgres আছে আর scale ছোট হলে আলাদা DB যোগ করা অপ্রয়োজনীয়।

### Hands-on experiment
```ts
// docker run -p 6333:6333 qdrant/qdrant   |   npm i @qdrant/js-client-rest
import { QdrantClient } from "@qdrant/js-client-rest";
const q = new QdrantClient({ url: "http://localhost:6333" });
await q.createCollection("docs", { vectors: { size: 1024, distance: "Cosine" } }); // bge-m3 = 1024 dim
await q.upsert("docs", { points: [{ id: 1, vector: emb, payload: { tenant_id: "acme", year: 2026 } }] });
const hits = await q.search("docs", {
  vector: queryEmb, limit: 5,
  filter: { must: [{ key: "tenant_id", match: { value: "acme" } }] },
});
```

### Production implementation
Payload index (`tenant_id`, `doc_id`), snapshot backup, replication, quantization দিয়ে RAM কমানো।

### Failure cases
Payload index ছাড়া filter দিয়ে লাখো point এ query, ফলে latency spike।

### Security concerns
API key চালু করো (default এ খোলা থাকতে পারে), network isolation, tenant filter বাধ্যতামূলক।

### Performance concerns
HNSW পুরোটা RAM এ রাখলে দ্রুত। বড় হলে on-disk + quantization।

### Cost concerns
Self-host এ শুধু server খরচ। RAM হলো মূল খরচ।

### What did I learn?
Qdrant = "filter + vector" এর জন্য শক্তিশালী open-source পছন্দ।

### What can I explain to someone else?
> "Qdrant হলো দ্রুত একটা লাইব্রেরি, যেখানে প্রতিটা বইয়ের গায়ে label থাকে, আর label ও বিষয় দুটো দিয়ে একসাথে খোঁজা যায়।"

### GitHub Evidence
- [ ] Qdrant local setup + filtered search script

### LinkedIn Post
- [ ] —

---

<a id="pinecone"></a>

## 2. Pinecone

### What is it?
Fully managed, closed-source vector database (serverless)। কোনো server চালাতে হয় না, API call করলেই হয়।

### Why does it exist?
অনেক team infra চালাতে চায় না। Pinecone vector search কে একটা managed service বানিয়েছে।

### What problem does it solve?
Zero-ops, auto-scaling vector search, enterprise compliance সহ।

### How does it work?
Index → namespace → records (id, values, metadata)। Serverless architecture এ storage আর compute আলাদা, usage অনুযায়ী bill হয়। Sparse-dense hybrid, metadata filter, আর integrated embedding/reranking (hosted model) ও আছে।

### Important concepts
- **Namespace** দিয়ে tenant ভাগ করা খুব সাধারণ pattern।
- Read/write unit ভিত্তিক pricing।
- Data provider এর cloud এ থাকে।

### Trade-offs
Ops শূন্য, কিন্তু vendor lock-in, data residency সীমাবদ্ধতা, আর বড় scale এ খরচ।

### Alternatives
Managed Qdrant/Weaviate/Zilliz, pgvector (Supabase/Neon এ managed Postgres)।

### When should I use it?
দ্রুত production এ যেতে হবে, team ছোট, ops করতে চাও না।

### When should I NOT use it?
Data নিজের infra তে রাখা বাধ্যতামূলক, বা খুব বেশি volume এ cost-sensitive।

### Hands-on experiment
Free tier এ একটা index বানাও, দুইটা namespace (`tenant-a`, `tenant-b`) এ data রেখে query করো, latency মাপো।

### Production implementation
Namespace-per-tenant, metadata filter, API key secret manager এ, usage monitoring।

### Failure cases
Namespace ভুল দেওয়ায় অন্য tenant এর data (code bug), বা usage spike এ অপ্রত্যাশিত bill।

### Security concerns
API key leak মানে পুরো index exposed। Key rotation, least-privilege key।

### Performance concerns
Network latency (cloud region) যোগ হয়। App এর কাছাকাছি region বাছো।

### Cost concerns
Usage-based, তাই read-heavy app এ হিসাব করে নাও।

### What did I learn?
Pinecone কিনে নেওয়া মানে ops এর বদলে টাকা দেওয়া।

### What can I explain to someone else?
> "নিজে গুদাম না বানিয়ে ভাড়া করা গুদাম: ঝামেলা নেই, কিন্তু ভাড়া দিতে হয় আর চাবি মালিকের কাছেও থাকে।"

### GitHub Evidence
- [ ] Pinecone namespace demo

### LinkedIn Post
- [ ] —

---

<a id="weaviate"></a>

## 3. Weaviate

### What is it?
Go তে লেখা open-source vector database (self-host বা Weaviate Cloud), schema-based collection, built-in hybrid search আর "vectorizer modules"।

### Why does it exist?
শুধু vector store না, একটা "AI-native database" দেওয়া, যেখানে embedding, hybrid search, generative module DB তেই।

### What problem does it solve?
Embedding তৈরি (module দিয়ে), BM25 + vector hybrid, multi-tenancy, সব এক জায়গায়।

### How does it work?
Collection এ properties (schema), HNSW vector index + inverted index (BM25)। `hybrid` query তে `alpha` দিয়ে keyword বনাম vector এর ওজন। Native multi-tenancy (প্রতি tenant আলাদা shard)।

### Important concepts
- `alpha = 0` → শুধু BM25, `alpha = 1` → শুধু vector।
- Vectorizer module চাইলে নিজে embed করে (তবে নিজের embedding দেওয়াও যায়)।
- gRPC/REST/GraphQL API।

### Trade-offs
অনেক feature built-in, কিন্তু শেখার curve আর resource ব্যবহার বেশি।

### Alternatives
Qdrant, Elasticsearch/OpenSearch (keyword-first + vector)।

### When should I use it?
Hybrid search বাধ্যতামূলক, অনেক tenant, all-in-one চাও।

### When should I NOT use it?
খুব সহজ use case বা আগে থেকে Postgres যথেষ্ট।

### Hands-on experiment
Docker এ চালিয়ে একই query `alpha` = 0, 0.5, 1 দিয়ে চালাও: product code (যেমন "SKU-4471") আর অর্থভিত্তিক প্রশ্নে ফল তুলনা করো।

### Production implementation
Multi-tenancy enable, tenant offload (inactive tenant cold storage এ), backup module।

### Failure cases
Schema পরিবর্তনে re-index লাগে, বড় data তে downtime পরিকল্পনা না থাকা।

### Security concerns
Auth (API key/OIDC) চালু করো, tenant isolation যাচাই।

### Performance concerns
BM25 + vector দুই index, তাই memory বেশি।

### Cost concerns
Self-host এ RAM/CPU। Cloud এ dimension আর object সংখ্যা ভিত্তিক।

### What did I learn?
Weaviate = hybrid আর multi-tenancy native।

### What can I explain to someone else?
> "একই খোঁজে শব্দ মিলিয়েও খোঁজে, অর্থ মিলিয়েও খোঁজে, আর তুমি ঠিক করো কোনটার কথা বেশি শুনবে।"

### GitHub Evidence
- [ ] Weaviate alpha sweep experiment

### LinkedIn Post
- [ ] —

---

<a id="pgvector"></a>

## 4. pgvector

### What is it?
PostgreSQL এর extension যেটা `vector` column type, distance operator (`<->` L2, `<=>` cosine, `<#>` inner product) আর ANN index (HNSW, IVFFlat) যোগ করে।

### Why does it exist?
অনেক app এর data ইতিমধ্যে Postgres এ। আলাদা vector DB যোগ করলে sync, consistency, ops এর ঝামেলা। pgvector এক DB তেই সব দেয়।

### What problem does it solve?
Vector + relational data এক জায়গায়: SQL join, transaction, existing backup/auth।

### How does it work?
```sql
CREATE EXTENSION vector;
CREATE TABLE chunks (id bigserial PRIMARY KEY, tenant_id text, content text, embedding vector(1024));
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
SELECT id, content FROM chunks
WHERE tenant_id = 'acme'
ORDER BY embedding <=> $1 LIMIT 5;
```

### Important concepts
- HNSW (ভালো recall/speed, build ধীর) বনাম IVFFlat (দ্রুত build, data আগে লাগে)।
- `halfvec` (16-bit) দিয়ে storage অর্ধেক।
- Filter + ANN এর interplay: খুব selective filter এ recall কমতে পারে (iterative scan option দেখো)।
- Full-text search (`tsvector`) এর সাথে মিলিয়ে hybrid।

### Trade-offs
সহজ stack আর ACID, কিন্তু খুব বড় scale (শত কোটি vector) এ dedicated DB এর চেয়ে পিছিয়ে।

### Alternatives
Qdrant/Weaviate (dedicated), managed Postgres (Supabase, Neon) তে pgvector।

### When should I use it?
আগে থেকে Postgres, কয়েক লাখ থেকে কয়েক কোটি vector, join/transaction দরকার। **বেশিরভাগ app এর জন্য সেরা শুরু।**

### When should I NOT use it?
বিলিয়ন-স্কেল, বা vector workload মূল DB কে ধীর করে দিচ্ছে।

### Hands-on experiment
Docker এ `pgvector/pgvector` image চালাও, ১০K chunk insert করো, index ছাড়া আর HNSW সহ query time মাপো (`EXPLAIN ANALYZE`)।

### Production implementation
Row-level security (RLS) দিয়ে tenant isolation, HNSW `ef_search` tune, Drizzle/Prisma তে vector type।

### Failure cases
Index ছাড়া query, ফলে full table scan। `ORDER BY` এ ভুল operator (index এর সাথে মেলে না), তাই index ব্যবহার হয় না।

### Security concerns
RLS + parameterized query। Embedding থেকেও কিছু তথ্য পুনরুদ্ধার সম্ভব, তাই sensitive data হিসেবে ধরো।

### Performance concerns
HNSW index RAM এ ধরলে দ্রুত। Build এর সময় `maintenance_work_mem` বাড়াও।

### Cost concerns
নতুন DB খরচ নেই, existing Postgres এ চলে।

### What did I learn?
"Just use Postgres" অনেক RAG app এর জন্য সঠিক উত্তর।

### What can I explain to someone else?
> "নতুন আলমারি না কিনে পুরনো আলমারিতেই একটা নতুন তাক বসানো।"

### GitHub Evidence
- [ ] pgvector + HNSW benchmark (with/without index)

### LinkedIn Post
- [ ] "RAG এর জন্য নতুন DB লাগবে? আগে pgvector দেখো"

---

<a id="chroma"></a>

## 5. Chroma

### What is it?
Open-source, developer-friendly vector database। In-process/local বা client-server মোডে চলে, Chroma Cloud ও আছে।

### Why does it exist?
Vector search শুরু করা যেন ৩ লাইনের কাজ হয়, notebook আর prototype এর জন্য।

### What problem does it solve?
Setup ছাড়াই local এ embedding store ও search।

### How does it work?
Collection → `add(ids, documents, embeddings, metadatas)` → `query(query_embeddings, n_results, where)`। চাইলে embedding function নিজে চালায়।

### Important concepts
- Local persistent mode (disk এ)।
- `where` (metadata) আর `where_document` (text contains) filter।

### Trade-offs
খুব সহজ, কিন্তু বড় production scale/HA এর জন্য অন্যগুলো বেশি পরীক্ষিত।

### Alternatives
LanceDB (embedded), pgvector, Qdrant (local Docker)।

### When should I use it?
শেখা, prototype, ছোট internal tool।

### When should I NOT use it?
Multi-tenant production, কঠোর SLA, বড় scale।

### Hands-on experiment
```ts
// docker run -p 8000:8000 chromadb/chroma   |   npm i chromadb
import { ChromaClient } from "chromadb";
const chroma = new ChromaClient({ path: "http://localhost:8000" });
const col = await chroma.getOrCreateCollection({ name: "notes" });
await col.add({ ids: ["1"], embeddings: [emb], documents: ["..."], metadatas: [{ phase: 4 }] });
console.log(await col.query({ queryEmbeddings: [qEmb], nResults: 3 }));
```

### Production implementation
ছোট internal tool এ server mode + backup। বড় হলে migrate পরিকল্পনা রাখো।

### Failure cases
Embedding function না দিয়ে default ব্যবহার করে পরে query এ ভিন্ন model, ফলে vector mismatch।

### Security concerns
Server mode এ auth চালু করো, public expose করবে না।

### Performance concerns
ছোট-মাঝারি data তে ভালো।

### Cost concerns
Free/local।

### What did I learn?
Chroma = শেখার আর prototype এর জন্য সবচেয়ে কম friction।

### What can I explain to someone else?
> "বড় গুদাম না, বাসার একটা ছোট তাক: দ্রুত শুরু করার জন্য perfect।"

### GitHub Evidence
- [ ] Chroma quickstart with Ollama embeddings

### LinkedIn Post
- [ ] —

---

<a id="milvus"></a>

## 6. Milvus

### What is it?
Open-source, cloud-native, **distributed** vector database (LF AI & Data Foundation), managed version Zilliz Cloud। বিলিয়ন-স্কেলের জন্য design করা।

### Why does it exist?
খুব বড় scale (image search, recommendation) এ single-node vector DB যথেষ্ট না।

### What problem does it solve?
Horizontal scaling, অনেক ধরনের index (HNSW, IVF, DiskANN, GPU index), আর storage/compute আলাদা।

### How does it work?
Microservice architecture (query node, data node, index node), object storage (S3/MinIO) + message queue। Milvus Lite (local) থেকে distributed cluster পর্যন্ত।

### Important concepts
- Partition/partition key দিয়ে data ভাগ (multi-tenant)।
- Index type বাছাই workload অনুযায়ী।
- Sparse vector + hybrid search support।

### Trade-offs
বিশাল scale, কিন্তু distributed mode চালানো জটিল (অনেক component)।

### Alternatives
Qdrant (distributed mode), Pinecone, Vespa।

### When should I use it?
কোটি থেকে বিলিয়ন vector, high QPS, dedicated platform team।

### When should I NOT use it?
ছোট/মাঝারি RAG app। Overkill।

### Hands-on experiment
Milvus Lite দিয়ে local এ collection বানিয়ে দুইটা index type (HNSW vs IVF_FLAT) এ recall আর latency তুলনা করো।

### Production implementation
Zilliz Cloud বা Kubernetes (Helm/operator) এ cluster, monitoring (Prometheus)।

### Failure cases
Self-hosted cluster এর কোনো একটা dependency (etcd/MinIO) down, ফলে পুরো search down।

### Security concerns
RBAC, TLS, network policy।

### Performance concerns
Scale এ সবচেয়ে শক্তিশালীদের একটা, কিন্তু tuning দরকার।

### Cost concerns
Distributed infra খরচ বেশি। ছোট workload এ অপচয়।

### What did I learn?
Scale যখন সত্যিই বিশাল, তখন Milvus। তার আগে না।

### What can I explain to someone else?
> "পাড়ার দোকান না, অনেক গুদাম আর ট্রাক নিয়ে একটা বিশাল distribution center।"

### GitHub Evidence
- [ ] Milvus Lite index type comparison

### LinkedIn Post
- [ ] "৬টা vector database: কোনটা কখন"

---

# Part B — Understand

<a id="indexing"></a>

## 7. Indexing

### What is it?
লাখো vector এর মধ্যে দ্রুত nearest neighbor খোঁজার জন্য data structure। Exact search (প্রতিটার সাথে তুলনা) ধীর, তাই **ANN (Approximate Nearest Neighbor)** index ব্যবহার করা হয়।

### Why does it exist?
১ কোটি vector × ১০২৪ dimension এর সাথে প্রতিটা query তুলনা করা প্রতি query তে বিশাল compute।

### What problem does it solve?
Millisecond এ search, সামান্য accuracy (recall) ছাড় দিয়ে।

### How does it work?
```text
Flat (brute force) : সব vector এর সাথে তুলনা → 100% recall, ধীর
IVF               : vector গুলোকে cluster এ ভাগ → query এর কাছের কয়েকটা cluster এ খোঁজো
HNSW              : multi-layer graph → উপরের layer এ বড় লাফ, নিচে সূক্ষ্ম খোঁজ
PQ / quantization : vector compress → কম memory, কিছু accuracy loss
DiskANN           : SSD-based graph → বিশাল data কম RAM এ
```

### Important concepts
- **Recall@k**: ANN result এ আসল top-k এর কত ভাগ এসেছে।
- **HNSW parameters**: `M` (graph degree), `ef_construction` (build quality), `ef_search` (query quality vs speed)।
- **Build time vs query time vs memory**: তিনটা trade-off।
- **Quantization**: scalar (float32 → int8), binary, product quantization।

### Trade-offs
Speed ↔ recall ↔ memory। একটা বাড়ালে অন্যটা কমে।

### Alternatives
Brute force (ছোট data, < ~১০K vector এ যথেষ্ট দ্রুত)।

### When should I use it?
হাজার হাজার vector এর বেশি হলে।

### When should I NOT use it?
খুব ছোট dataset এ index tuning সময় নষ্ট।

### Hands-on experiment
pgvector এ `ef_search` = 10, 40, 100, 200 দিয়ে recall@10 (exact search এর সাথে তুলনা করে) আর latency chart বানাও।

### Production implementation
Recall eval set রাখো, index parameter বদলালে recall regression test।

### Failure cases
Default `ef_search` কম, তাই recall ৮০%, আর সঠিক chunk retrieve হচ্ছে না, কেউ টেরও পাচ্ছে না।

### Security concerns
নেই সরাসরি।

### Performance concerns
এই topic টাই performance: index type ও parameter latency নির্ধারণ করে।

### Cost concerns
HNSW RAM-heavy, quantization দিয়ে RAM খরচ ৪–৩২ গুণ কমানো যায়।

### What did I learn?
Vector search "approximate", আর সেই approximation tune করতে হয়।

### What can I explain to someone else?
> "শহরে ঠিকানা খুঁজতে প্রতিটা বাড়িতে নক না করে আগে এলাকা, তারপর রাস্তা, তারপর বাড়ি। HNSW ঠিক এরকম স্তরে স্তরে খোঁজে।"

### GitHub Evidence
- [ ] ef_search vs recall/latency chart

### LinkedIn Post
- [ ] "Vector search আসলে approximate: HNSW সহজ ভাষায়"

---

<a id="similarity-search"></a>

## 8. Similarity Search

### What is it?
Query vector এর সবচেয়ে "কাছের" k টা vector খুঁজে বের করা (k-NN), একটা distance metric দিয়ে: cosine, dot product, বা Euclidean (L2)। ভিত্তি: [embeddings.md](../01-llm-fundamentals/embeddings.md)।

### Why does it exist?
Embedding space এ অর্থের মিল = vector এর কাছাকাছি থাকা। Search মানে কাছের vector খোঁজা।

### What problem does it solve?
Keyword না মিললেও অর্থ মিলে এমন content খুঁজে পাওয়া ("টাকা ফেরত" ↔ "refund")।

### How does it work?
```text
query → same embedding model → q
score(q, d) = cosine(q, d)  → top-k sort → (optional) score threshold
```

### Important concepts
- **একই embedding model** index আর query তে।
- **Normalized vector** হলে cosine = dot product (দ্রুত)।
- **Score threshold**: সবসময় top-k আসে, এমনকি অপ্রাসঙ্গিক হলেও, তাই minimum score রাখো।
- **Query vs document asymmetry**: কিছু model এ query তে prefix লাগে (যেমন `"query: "`), model card দেখো।

### Trade-offs
Semantic match ভালো, কিন্তু exact term (code, নাম, ID) এ দুর্বল। এজন্য hybrid।

### Alternatives
Keyword/BM25, hybrid।

### When should I use it?
Natural-language প্রশ্ন, paraphrase, multilingual (বাংলা প্রশ্ন ↔ English doc, multilingual model দিয়ে)।

### When should I NOT use it?
Exact lookup (order ID, SKU)। এখানে SQL/keyword।

### Hands-on experiment
১০টা প্রশ্নের জন্য top-5 এর score দেখো। অপ্রাসঙ্গিক প্রশ্নেও ("আজ আবহাওয়া কেমন?") কত score আসে? সেখান থেকে threshold ঠিক করো।

### Production implementation
Score log করো, low-score হলে "তথ্য পাওয়া যায়নি" path (hallucination এড়াতে)।

### Failure cases
Threshold না থাকায় অপ্রাসঙ্গিক chunk দিয়ে model বানিয়ে উত্তর দিয়েছে।

### Security concerns
Query embedding log এ রাখলে তা user এর প্রশ্নের প্রতিনিধি, তাই PII হিসেবে ধরো।

### Performance concerns
Query embedding এর latency (API call) + ANN search।

### Cost concerns
প্রতি query তে একটা embedding call (সস্তা), আর বারবার আসা query এর embedding cache করা যায়।

### What did I learn?
Similarity search সবসময় "কিছু" ফেরত দেয়। সেটা relevant কিনা যাচাই তোমার কাজ।

### What can I explain to someone else?
> "Map এ তোমার অবস্থানের সবচেয়ে কাছের ৫টা দোকান। কিন্তু সবচেয়ে কাছেরটাও ১০ কিমি দূরে হতে পারে, তাই দূরত্ব দেখাও জরুরি।"

### GitHub Evidence
- [ ] Score distribution + threshold calibration

### LinkedIn Post
- [ ] —

---

<a id="metadata-filtering"></a>

## 9. Metadata Filtering

### What is it?
Vector search এর সাথে structured condition যোগ করা: `tenant_id = X AND year >= 2025 AND doc_type IN (...)`। [Chunk metadata](chunking.md#chunk-metadata) এর ব্যবহার।

### Why does it exist?
শুধু অর্থের মিল যথেষ্ট না। Permission, সময়, product, ভাষা অনুযায়ী ছাঁকতে হয়।

### What problem does it solve?
Security (tenant isolation), relevance (সঠিক version/product), আর search space ছোট করা।

### How does it work?
```text
Pre-filter  : আগে filter, তারপর বাকিদের মধ্যে ANN   → সঠিক, কিন্তু selective filter এ ANN দুর্বল হতে পারে
Post-filter : আগে ANN top-k, তারপর filter          → দ্রুত, কিন্তু top-k থেকে সব বাদ পড়ে ০ result হতে পারে
Filtered ANN: DB নিজে search এর সময় filter মানে   → (Qdrant, Weaviate, pgvector iterative scan)
```

### Important concepts
- **Security filter সবসময় server-side, mandatory**।
- **Filter fields index করো**।
- **LLM-generated filter** (self-query): প্রশ্ন থেকে filter বের করা ("২০২৫ এর report" → `year=2025`), কিন্তু security filter কখনো LLM এর হাতে না।

### Trade-offs
শক্ত filter = precise, কিন্তু খুব কঠোর হলে প্রাসঙ্গিক data বাদ পড়ে।

### Alternatives
আলাদা index/collection per tenant (শক্ত isolation, কিন্তু অনেক index)।

### When should I use it?
Multi-tenant, versioned document, সময়-সংবেদনশীল তথ্য।

### When should I NOT use it?
Filter field এ data অসম্পূর্ণ হলে (অর্ধেক doc এ `year` নেই), নাহলে সেগুলো চুপচাপ বাদ পড়বে।

### Hands-on experiment
১০০০ chunk, ১০ tenant। Post-filter এ `limit 5` দিয়ে কতবার ০ result আসে মাপো, তারপর pre/filtered search এ তুলনা করো।

### Production implementation
```ts
function secureSearch(user: User, queryVec: number[], extra?: Filter) {
  const must = [{ key: "tenant_id", match: { value: user.tenantId } }]; // কখনো বাদ যাবে না
  return vectorDb.search({ vector: queryVec, filter: { must: [...must, ...(extra?.must ?? [])] }, limit: 8 });
}
```

### Failure cases
Post-filter দিয়ে ছোট tenant এর জন্য প্রায় সবসময় খালি result।

### Security concerns
এটা RAG এর **সবচেয়ে গুরুত্বপূর্ণ security control**। Filter bug = data breach। Integration test লিখো: tenant A এর query তে tenant B এর doc কখনো আসবে না।

### Performance concerns
Indexed filter দ্রুত। Unindexed filter বিশাল slow-down।

### Cost concerns
নগণ্য।

### What did I learn?
Metadata filter = RAG এর authorization layer।

### What can I explain to someone else?
> "লাইব্রেরিতে শুধু তোমার সদস্যপদের অনুমতি আছে এমন তাকগুলোতেই খোঁজা হবে, বাকি তাক তোমার চোখেই পড়বে না।"

### GitHub Evidence
- [ ] Tenant isolation integration test

### LinkedIn Post
- [ ] "RAG এ data leak এর সবচেয়ে সাধারণ কারণ"

---

<a id="hybrid-search"></a>

## 10. Hybrid Search

### What is it?
**Dense (vector/semantic)** আর **sparse (keyword/BM25)** search একসাথে চালিয়ে ফল মিলিয়ে দেওয়া। Vector DB এর দিক থেকে এখানে দেখছি এটা কীভাবে implement হয়। Retrieval এর দিক থেকে বিস্তারিত: [retrieval.md](retrieval.md#hybrid-search)।

### Why does it exist?
Vector search অর্থ বোঝে কিন্তু exact term (error code, product নাম, ID) miss করে। Keyword search exact term ধরে কিন্তু paraphrase বোঝে না।

### What problem does it solve?
দুই ধরনের প্রশ্নেই ভালো recall।

### How does it work?
```text
             ┌─► dense ANN  → ranked list A ─┐
query ───────┤                               ├─► fusion (RRF / weighted) → final top-k
             └─► BM25/sparse → ranked list B ─┘
```
DB-level support: Weaviate (`hybrid` + `alpha`), Qdrant (prefetch + RRF fusion), Pinecone (sparse-dense), Milvus, Elasticsearch/OpenSearch, Postgres (pgvector + `tsvector`)।

### Important concepts
- Score scale আলাদা (BM25 unbounded, cosine 0–1), তাই সরাসরি যোগ না করে RRF বা normalize।
- Sparse vector: BM25 বা learned sparse (SPLADE, bge-m3 sparse output)।

### Trade-offs
ভালো recall, কিন্তু দুই index, বেশি complexity আর latency।

### Alternatives
শুধু dense + reranker, শুধু BM25 (ছোট, keyword-heavy corpus)।

### When should I use it?
Technical docs, product catalog, code, legal, যেখানে exact term গুরুত্বপূর্ণ। **Production RAG এ প্রায় default।**

### When should I NOT use it?
খুব conversational/semantic content এ লাভ কম হতে পারে। Eval করে দেখো।

### Hands-on experiment
৩০টা প্রশ্ন বানাও: ১০টা exact-term ("ERR_4471 কী?"), ১০টা paraphrase, ১০টা mixed। Dense, BM25, hybrid তিনটার hit rate table।

### Production implementation
Indexing এ dense + sparse দুই vector একসাথে লেখো, query তে DB এর native fusion ব্যবহার করো।

### Failure cases
BM25 tokenizer বাংলা ঠিকমতো ভাঙছে না, তাই বাংলা keyword search কাজ করছে না।

### Security concerns
দুই path এই একই tenant/ACL filter।

### Performance concerns
দুই search parallel এ চালাও।

### Cost concerns
Index storage প্রায় দ্বিগুণ।

### What did I learn?
Hybrid = "শব্দ" আর "অর্থ" দুই দিক থেকে খোঁজা।

### What can I explain to someone else?
> "দুইজন খোঁজাখুঁজি করছে: একজন হুবহু শব্দ মেলায়, আরেকজন অর্থ বোঝে। দুজনের তালিকা মিলিয়ে সেরাটা নাও।"

### GitHub Evidence
- [ ] Dense vs BM25 vs hybrid table

### LinkedIn Post
- [ ] —

---

<a id="collection-design"></a>

## 11. Collection Design

### What is it?
Vector DB তে data কীভাবে সাজাবে তার পরিকল্পনা: কয়টা collection/index, multi-tenancy কৌশল, কোন vector (dense/sparse/named), কোন payload field, embedding model version কীভাবে handle করবে।

### Why does it exist?
ভুল design পরে বদলানো মানে পুরো re-index, migration, downtime। আর ভুল tenancy design = security বা performance সমস্যা।

### What problem does it solve?
Scalable, secure, migrate-যোগ্য knowledge store।

### How does it work?
```text
Option A: one collection, tenant_id payload (indexed)   → সহজ, অনেক tenant এ ভালো
Option B: collection per tenant                          → শক্ত isolation, কম tenant এ
Option C: DB-native multi-tenancy (Weaviate tenants, Milvus partition key, Pinecone namespace)

Versioning: docs_v1 (bge-m3) → docs_v2 (new model)
            → v2 তে নতুন করে embed → alias swap (blue-green)
```

### Important concepts
- **Embedding model = schema**: model বদলালে পুরনো vector অব্যবহারযোগ্য, পুরো re-embed।
- **Alias**: app সবসময় `docs` alias এ query করে, পেছনে collection swap।
- **Dimension + distance** collection তৈরির সময় fix।
- **Chunk ↔ document mapping**: document delete/update হলে তার সব chunk খুঁজে মুছতে `doc_id` লাগে।

### Trade-offs
এক collection = সহজ ops, কিন্তু noisy neighbor ও filter bug এর ঝুঁকি। অনেক collection = isolation, কিন্তু ops overhead।

### Alternatives
Tenant অনুযায়ী আলাদা DB instance (enterprise, কঠোর compliance)।

### When should I use it?
Production এ যাওয়ার আগে, design review তে।

### When should I NOT use it?
Prototype এ over-design করবে না, কিন্তু অন্তত `doc_id` আর `tenant_id` রাখো।

### Hands-on experiment
Blue-green re-index practice: `docs_v1` (একটা model) থেকে `docs_v2` (অন্য model) বানাও, alias swap করো, দুইটার retrieval eval তুলনা করে তারপর পুরনোটা মুছো।

### Production implementation
- Ingestion idempotent: `upsert` by deterministic chunk id (`doc_id#chunk_index`)।
- Document update → পুরনো chunk delete by `doc_id` → নতুন insert।
- Collection config (model, dim, chunker version) metadata হিসেবে store।

### Failure cases
- Embedding model upgrade করে শুধু নতুন document নতুন model এ embed, পুরনো গুলো পুরনো model এ, ফলে mixed space আর search ভাঙা।
- Document delete হয়েছে কিন্তু chunk রয়ে গেছে ("ghost" data, GDPR সমস্যা)।

### Security concerns
Right-to-delete: user/doc delete হলে সব chunk ও vector মুছতে হবে, backup সহ।

### Performance concerns
Collection খুব বেশি হলে (হাজারো) কিছু DB তে overhead।

### Cost concerns
Re-index = পুরো corpus আবার embed করার খরচ, তাই model বদলানোর আগে হিসাব করো।

### What did I learn?
Embedding model + chunker + schema = তোমার index এর "version"। এগুলো track করো।

### What can I explain to someone else?
> "গুদাম সাজানোর নকশা: কোন মালিকের মাল কোন তাকে, লেবেল কী, আর নতুন সিস্টেমে যাওয়ার সময় পুরনো গুদাম চালু রেখে নতুনটা সাজিয়ে তারপর দরজা বদলানো।"

### GitHub Evidence
- [ ] Blue-green re-index with alias swap

### LinkedIn Post
- [ ] "Embedding model বদলালে কী হয়? Vector DB migration"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

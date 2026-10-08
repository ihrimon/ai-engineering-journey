# Phase 1 — LLM Fundamentals: Rerankers (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Reranker model এর নাম আর দাম বদলায়। এখানের model নামগুলো উদাহরণ, ব্যবহারের আগে docs দেখবে।
>
> 🧪 Hands-on এ Python এর `sentence-transformers` দিয়ে local reranker চালানো হয়েছে (খরচ শূন্য, Python Track এর সাথে মেলে)। বাংলার জন্য multilingual reranker লাগবে, যেমন `BAAI/bge-reranker-v2-m3`।
>
> 🔗 আগের chapter [Embeddings](embeddings.md) এর পরের ধাপ। Embedding দিয়ে **খোঁজা**, reranker দিয়ে **বাছাই**। Phase 4 (RAG) এ এই দুটো একসাথে লাগবে।

## Quick Cheat Sheet

| Topic | এক লাইনে |
|---|---|
| [Why reranking is required](#why-reranking-is-required) | Embedding search দ্রুত কিন্তু মোটা দাগের। সবচেয়ে প্রাসঙ্গিক document উপরে আনতে দ্বিতীয় একটা যাচাই লাগে |
| [Cross-encoder concepts](#cross-encoder-concepts) | প্রশ্ন আর document **একসাথে** পড়ে relevance score দেয়, তাই বেশি নির্ভুল কিন্তু ধীর |
| [Reranking pipeline](#reranking-pipeline) | দ্রুত retrieve (অনেকগুলো) → reranker (ভালোভাবে সাজানো) → সেরা কয়েকটা LLM কে |
| [Reranker trade-offs](#reranker-trade-offs) | ভালো quality বনাম বাড়তি latency আর খরচ; কতগুলো document rerank করবে, সেটাই মূল সিদ্ধান্ত |

```text
                   ┌────────────── দ্রুত, মোটা দাগে ──────────────┐   ┌──── ধীর, নির্ভুল ────┐
প্রশ্ন ──► Embedding ──► Vector DB থেকে top 50 ──► Reranker (cross-encoder) ──► top 5 ──► LLM ──► উত্তর
                   └──────────── লক্ষ্য: কিছু যেন বাদ না পড়ে ────┘   └─ লক্ষ্য: সেরাটা উপরে ─┘
                                   (Recall)                                 (Precision)
```

> **Embeddings chapter এর সাথে যোগ:** সেখানে দেখেছি "আমি কুকুর ভালোবাসি" আর "আমি কুকুর ঘৃণা করি" embedding এ প্রায় একই। কারণ embedding প্রশ্ন আর document কে **আলাদা আলাদা** পড়ে। Reranker দুটোকে একসাথে পড়ে, তাই এই সূক্ষ্ম পার্থক্য ধরতে পারে।

---

<a id="why-reranking-is-required"></a>

## 1. Why Reranking is Required

### What is it?
Reranking মানে প্রথম ধাপের search এ পাওয়া document গুলোকে আরেকটা, আরও নির্ভুল model দিয়ে **নতুন করে সাজানো**, যাতে সবচেয়ে প্রাসঙ্গিক document সবার উপরে আসে।

### Why does it exist?
Embedding search লাখ document থেকে দ্রুত খুঁজতে পারে, কিন্তু তার মাপ হলো "কতটা কাছাকাছি বিষয়", "কতটা সঠিক উত্তর" না। ফলে সঠিক document প্রায়ই top 20 এ থাকে, কিন্তু top 3 এ থাকে না।

### What problem does it solve?
LLM কে কম কিন্তু **সঠিক** context দেওয়া। RAG এ LLM এর উত্তর ততটাই ভালো, যতটা ভালো context সে পায়।

### How does it work?
একটা উদাহরণ (score গুলো কাল্পনিক, বোঝার জন্য):

```text
প্রশ্ন: "পাসওয়ার্ড reset করার ধাপ কী?"

Embedding search এর ক্রম:
  1. "আমিও password reset করতে পারছি না, কেউ সাহায্য করুন"   0.91  ← একই বিষয়, কিন্তু উত্তর নেই
  2. "Password এর নিয়ম: কমপক্ষে ৮ অক্ষর"                        0.87
  3. "Settings > Security তে গিয়ে Reset Password চাপুন"          0.84  ← আসল উত্তর

Reranker এর পরে:
  1. "Settings > Security তে গিয়ে Reset Password চাপুন"          ✅ উপরে উঠে এলো
  2. "Password এর নিয়ম: কমপক্ষে ৮ অক্ষর"
  3. "আমিও password reset করতে পারছি না..."                     ⬇️ নিচে নামলো
```

Embedding দেখে "একই বিষয় কিনা", reranker দেখে "এই document কি প্রশ্নের উত্তর দেয়"।

### Important concepts
- **Recall vs Precision:**
  - **Recall** = সঠিক document গুলো খুঁজে পাওয়া তালিকায় আছে কিনা (প্রথম ধাপের কাজ)।
  - **Precision** = উপরের কয়েকটা document আসলেই প্রাসঙ্গিক কিনা (reranker এর কাজ)।
- **Reranker নতুন document খোঁজে না:** প্রথম ধাপে যা বাদ পড়েছে, reranker তা ফেরত আনতে পারে না। শুধু যা পেয়েছে তা সাজায়।
- **কম context = ভালো উত্তর:** অপ্রাসঙ্গিক document LLM কে বিভ্রান্ত করে (lost in the middle, Core Concepts)।

### Trade-offs
Quality বাড়ে, কিন্তু প্রতিটা প্রশ্নে বাড়তি একটা ধাপ, তাই latency আর খরচ বাড়ে।

### Alternatives
- Hybrid search (keyword + embedding) দিয়ে প্রথম ধাপই ভালো করা।
- ভালো embedding model আর ভালো chunking।
- LLM কে দিয়েই document বাছাই করানো (ধীর আর দামি)।

### When should I use it?
- RAG এ উত্তর প্রায়ই ভুল বা অসম্পূর্ণ, অথচ সঠিক document টা search result এ কোথাও আছে।
- Document সংখ্যা অনেক আর একই বিষয়ে অনেক কাছাকাছি লেখা আছে (FAQ, support ticket, policy)।

### When should I NOT use it?
- Document খুব কম (কয়েকশো), আর embedding search ই ভালো কাজ করছে।
- সঠিক document প্রথম ধাপেই খুঁজে পাওয়া যাচ্ছে না। তখন সমস্যা retrieval এ, reranker দিয়ে ঠিক হবে না।

### Hands-on experiment
Embeddings chapter এর ছোট retrieval eval টা আবার চালাও, এবার দুইভাবে:
1. শুধু embedding: সঠিক document top-3 এ কতবার এলো?
2. Embedding top-20 → reranker → top-3: এখন কতবার এলো?

পার্থক্যটাই reranker এর মূল্য। ([Cross-encoder](#cross-encoder-concepts) এ code আছে।)

### Production implementation
Reranker যোগ করার আগে আর পরে একই eval চালিয়ে প্রমাণ করো যে সত্যিই উন্নতি হয়েছে। মাপ ছাড়া শুধু "ভালো হবে" ভেবে যোগ করবে না।

### Failure cases
Reranker যোগ করা হলো, কিন্তু quality বাড়লো না, কারণ সঠিক document প্রথম ধাপের top-20 তেই ছিল না।

### Security concerns
কেউ এমন document ঢোকাতে পারে যা অনেক প্রশ্নের "উত্তর" এর মতো শোনায়, যাতে reranker সেটা উপরে তোলে। Reranker ও শেষ পর্যন্ত একটা model, ভুল করতে পারে।

### Performance concerns
প্রতিটা প্রশ্নে বাড়তি একটা model call। Real-time chat এ latency budget এ জায়গা রাখতে হবে।

### Cost concerns
API reranker সাধারণত প্রতি request বা প্রতি document হিসেবে bill করে। Local reranker এ খরচ শুধু hardware এর।

### What did I learn?
- Search দুই ধাপের: আগে "কিছু যেন বাদ না পড়ে", তারপর "সেরাটা উপরে আসুক"।

### What can I explain to someone else?
> "Job এর জন্য ৫০০ CV এলো। প্রথমে keyword দেখে দ্রুত ৫০টা বাছাই (embedding)। তারপর সেই ৫০টা মন দিয়ে পড়ে সেরা ৫ জনকে interview তে ডাকা (reranker)।"

### GitHub Evidence
- [ ] Embedding-only vs embedding + reranker: hit rate comparison

### LinkedIn Post
- [ ] "RAG এর উত্তর ভুল হচ্ছে? সমস্যা হয়তো search এর ক্রমে"

---

<a id="cross-encoder-concepts"></a>

## 2. Cross-encoder Concepts

### What is it?
Reranker সাধারণত একটা **cross-encoder** model। এটা প্রশ্ন আর document কে **একসাথে একটা input হিসেবে** পড়ে, আর একটা সংখ্যা দেয়: এই document এই প্রশ্নের জন্য কতটা প্রাসঙ্গিক।

### Why does it exist?
Embedding model (যাকে বলে **bi-encoder**) প্রশ্ন আর document আলাদা আলাদা vector বানায়, তাই দুটোর মধ্যে শব্দ ধরে ধরে তুলনা করতে পারে না। Cross-encoder সেটা পারে।

### What problem does it solve?
Embedding এর যে সূক্ষ্মতা হারায় (negation, প্রশ্ন বনাম উত্তর, নির্দিষ্ট শর্ত), সেগুলো ধরা।

### How does it work?
```text
Bi-encoder (Embedding):
  প্রশ্ন   ──► Model ──► vector A ┐
                                   ├──► cosine(A, B) = মিল
  Document ──► Model ──► vector B ┘
  ✅ Document এর vector আগেই বানিয়ে রাখা যায় → খুব দ্রুত
  ❌ দুটো কখনো একসাথে "দেখা" হয় না

Cross-encoder (Reranker):
  [প্রশ্ন + Document] ──► Model ──► relevance score (একটা সংখ্যা)
  ✅ প্রশ্নের প্রতিটা শব্দ document এর প্রতিটা শব্দের সাথে মিলিয়ে দেখে → নির্ভুল
  ❌ প্রতিটা (প্রশ্ন, document) জোড়ার জন্য নতুন করে চালাতে হয় → ধীর
```

| | Bi-encoder (Embedding) | Cross-encoder (Reranker) |
|---|---|---|
| Input | প্রশ্ন আর document আলাদা | প্রশ্ন + document একসাথে |
| Output | Vector | একটা score |
| আগে থেকে হিসাব করা যায় | হ্যাঁ (document embed করে রাখা যায়) | না |
| গতি | খুব দ্রুত, লাখ document এ চলে | ধীর, কয়েক ডজনেই ভালো |
| নির্ভুলতা | মোটামুটি | বেশি |

### Important concepts
- **Attention এর জাদু:** Core Concepts এর Transformer এর self-attention মনে আছে? Cross-encoder এ প্রশ্ন আর document একই input এ থাকায় attention দুটোর শব্দগুলোকে একে অপরের সাথে মিলিয়ে দেখতে পারে।
- **Score এর মান model ভেদে আলাদা:** অনেক reranker এর score probability না, তাই 0.8 মানে "80% নিশ্চিত" না। Threshold নিজের data দিয়ে ঠিক করতে হয়।
- **জনপ্রিয় reranker (উদাহরণ):** Cohere Rerank, Voyage rerank (API); `bge-reranker` (BAAI), Jina reranker (open model, local এ চালানো যায়)।

### Trade-offs
নির্ভুলতার বিনিময়ে গতি। এজন্যই cross-encoder সব document এ না, শুধু প্রথম ধাপের বাছাই করা কয়েকটায় চালানো হয়।

### Alternatives
- **LLM-as-reranker:** LLM কে document গুলো দিয়ে বলা "প্রাসঙ্গিকতা অনুযায়ী সাজাও"। নমনীয়, কিন্তু ধীর আর দামি।
- **Late-interaction model (যেমন ColBERT):** bi-encoder আর cross-encoder এর মাঝামাঝি।

### When should I use it?
প্রথম ধাপে পাওয়া ২০ থেকে ১০০টা document নতুন করে সাজাতে।

### When should I NOT use it?
লাখ document এর পুরো collection এ সরাসরি। প্রতিটা প্রশ্নে লাখবার model চালানো অসম্ভব রকম ধীর।

### Hands-on experiment
Python এ local cross-encoder (Python Track এর সাথে মেলে):

```python
# pip install -U sentence-transformers
from sentence_transformers import CrossEncoder

model = CrossEncoder("BAAI/bge-reranker-v2-m3")  # multilingual, বাংলা সহ

query = "পাসওয়ার্ড reset করার ধাপ কী?"
docs = [
    "আমিও password reset করতে পারছি না, কেউ সাহায্য করুন",
    "Password এর নিয়ম: কমপক্ষে ৮ অক্ষর",
    "Settings > Security তে গিয়ে Reset Password চাপুন",
]

scores = model.predict([(query, doc) for doc in docs])
for score, doc in sorted(zip(scores, docs), reverse=True):
    print(f"{score:.3f}  {doc}")
```

একই document গুলোর embedding similarity (Embeddings chapter এর code) এর ক্রমের সাথে তুলনা করো। Embeddings chapter এর "ভালোবাসি বনাম ঘৃণা করি" জোড়াটাও reranker দিয়ে চালিয়ে দেখো।

### Production implementation
- Reranker model একবার load করে রাখো, প্রতিটা request এ নতুন করে load করবে না।
- একটা প্রশ্নের সব (প্রশ্ন, document) জোড়া একসাথে batch করে পাঠাও।

### Failure cases
- English-only reranker দিয়ে বাংলা document সাজানো, ফলে ক্রম প্রায় এলোমেলো।
- লম্বা document reranker এর input limit এ কেটে যায়, আসল উত্তর ছিল কাটা অংশে।

### Security concerns
Reranker ও document এর লেখা পড়ে, তাই document এ লুকানো লেখা দিয়ে তাকে প্রভাবিত করার চেষ্টা সম্ভব। Reranker এর ফলকে চূড়ান্ত সত্য ধরবে না।

### Performance concerns
সময় বাড়ে document সংখ্যা × document এর দৈর্ঘ্য অনুযায়ী। CPU তে ৫০টা লম্বা document rerank করা লক্ষণীয় সময় নিতে পারে, GPU তে অনেক দ্রুত।

### Cost concerns
Local model এ API খরচ নেই, কিন্তু real-time এ দ্রুত চালাতে GPU লাগতে পারে।

### What did I learn?
- Embedding "আলাদা আলাদা পড়ে মেলায়", cross-encoder "একসাথে পড়ে বিচার করে"।

### What can I explain to someone else?
> "Bi-encoder হলো দুইজনের আলাদা আলাদা biodata দেখে মিল খোঁজা। Cross-encoder হলো দুইজনকে মুখোমুখি বসিয়ে কথা বলতে দেখে বিচার করা। দ্বিতীয়টা বেশি নির্ভুল, কিন্তু সবার সাথে সবাইকে বসানো সম্ভব না।"

### GitHub Evidence
- [ ] Bi-encoder vs cross-encoder ranking comparison (বাংলা প্রশ্ন)

### LinkedIn Post
- [ ] —

---

<a id="reranking-pipeline"></a>

## 3. Reranking Pipeline

### What is it?
Retrieval থেকে উত্তর পর্যন্ত পুরো ধাপগুলো, যেখানে reranker মাঝখানে বসে।

### Why does it exist?
দ্রুত কিন্তু মোটা দাগের পদ্ধতি (embedding) আর ধীর কিন্তু নির্ভুল পদ্ধতি (cross-encoder), দুটোর সুবিধা একসাথে পেতে।

### What problem does it solve?
লাখ document থেকে কয়েক সেকেন্ডের মধ্যে সবচেয়ে প্রাসঙ্গিক কয়েকটা বের করা।

### How does it work?
```text
1. প্রশ্ন আসে
2. Retrieve:  Embedding (+ keyword/hybrid) দিয়ে top-K বের করো     (যেমন K = 50)
3. Rerank:    Cross-encoder দিয়ে ৫০টা জোড়ার score বের করে সাজাও
4. Filter:    Score threshold এর নিচেরগুলো বাদ দাও (ঐচ্ছিক)
5. Select:    Top-N নাও                                              (যেমন N = 5)
6. Generate:  সেই N টা document context হিসেবে দিয়ে LLM এর উত্তর
```

### Important concepts
- **দুইটা সংখ্যা, দুইটা আলাদা সিদ্ধান্ত:**
  - **K (কতগুলো retrieve):** বড় হলে সঠিক document তালিকায় থাকার সম্ভাবনা বেশি (recall), কিন্তু reranking ধীর।
  - **N (কতগুলো LLM কে):** ছোট হলে context পরিষ্কার আর সস্তা, কিন্তু দরকারি তথ্য বাদ পড়ার ঝুঁকি।
- **"পাইনি" বলার সুযোগ:** সব score threshold এর নিচে হলে LLM কে document না দিয়ে "এই বিষয়ে তথ্য নেই" বলা, hallucination কমায়।
- **Metadata filter আগে:** User এর অনুমতি নেই এমন document reranking এর আগেই বাদ দাও।

### Trade-offs
K বাড়ালে quality কিছুটা বাড়ে, latency আর খরচ সরাসরি বাড়ে।

### Alternatives
Rerank ছাড়া retrieve → LLM (সহজ pipeline); অথবা LLM কে tool দিয়ে নিজে খুঁজতে দেওয়া (agentic retrieval, Phase 4)।

### When should I use it?
Production RAG, যেখানে উত্তরের নির্ভুলতা গুরুত্বপূর্ণ।

### When should I NOT use it?
Prototype এর প্রথম দিন। আগে সহজ pipeline বানাও, eval দিয়ে মাপো, তারপর দরকার হলে reranker যোগ করো।

### Hands-on experiment
K = 10, 20, 50 আর N = 3, 5 এর ভিন্ন ভিন্ন জোড়া দিয়ে একই eval চালাও। একটা table বানাও: hit rate আর মোট সময়। কোন জোড়া তোমার জন্য সবচেয়ে ভালো balance দেয়?

### Production implementation
```text
প্রতিটা request এ log করো:
  retrieve এর সময় │ rerank এর সময় │ K │ N │ top score │ সঠিক উত্তর এলো কিনা (feedback)
```
এতে বোঝা যায় সমস্যা retrieval এ নাকি reranking এ।

### Failure cases
- K খুব ছোট (যেমন 5), তাই reranker এর সাজানোর মতো কিছুই থাকে না।
- Threshold না থাকায় একদম অপ্রাসঙ্গিক document ও LLM এর কাছে যায়।

### Security concerns
Access control এর filter reranking এর আগে বসাও। পরে বসালে অননুমোদিত document reranker এর কাছে আর log এ চলে যায়।

### Performance concerns
মোট latency = retrieve + rerank + LLM। Rerank অংশটা K দিয়ে নিয়ন্ত্রণ করা যায়।

### Cost concerns
ভালো reranking এ N ছোট রাখা যায়, তাই LLM এর input token কমে। অনেক সময় reranker এর খরচ LLM এর token সাশ্রয়েই উঠে আসে।

### What did I learn?
- Pipeline এর প্রতিটা ধাপ আলাদা মাপতে হয়, নাহলে বোঝা যায় না ভুলটা কোথায়।

### What can I explain to someone else?
> "লাইব্রেরিতে প্রথমে catalog দেখে ৫০টা বই নামানো, তারপর সূচিপত্র পড়ে ৫টা বেছে নেওয়া, আর শেষে শুধু সেই ৫টা থেকে উত্তর লেখা।"

### GitHub Evidence
- [ ] K/N sweep table (hit rate + latency)

### LinkedIn Post
- [ ] —

---

<a id="reranker-trade-offs"></a>

## 4. Reranker Trade-offs

### What is it?
Reranker যোগ করার লাভ আর দামের হিসাব, আর কোন reranker বাছবো তার সিদ্ধান্ত।

### Why does it exist?
Reranker কোনো বিনামূল্যের উন্নতি না। প্রতিটা প্রশ্নে বাড়তি সময়, খরচ আর একটা নতুন dependency যোগ হয়।

### What problem does it solve?
"Reranker কি সত্যিই লাগবে, আর লাগলে কোনটা?" এই সিদ্ধান্ত data দিয়ে নেওয়া।

### How does it work?
| বিষয় | API reranker (Cohere, Voyage ইত্যাদি) | Local/open reranker (bge, Jina ইত্যাদি) |
|---|---|---|
| শুরু করা | খুব সহজ | Model load, hardware লাগে |
| Quality | সাধারণত ভালো | ভালো, model ভেদে আলাদা |
| Latency | Network সহ | Hardware নির্ভর, GPU তে দ্রুত |
| খরচ | প্রতি request/document | Hardware এর fixed খরচ |
| Privacy | Document provider এর কাছে যায় | নিজের কাছে থাকে |
| ভাষা | Model ভেদে, multilingual version দেখো | Multilingual model বাছতে হবে (বাংলার জন্য) |

### Important concepts
- **মূল lever হলো K:** কতগুলো document rerank হবে, সেটাই latency আর খরচ ঠিক করে।
- **Model এর size:** ছোট reranker দ্রুত, বড়টা নির্ভুল। একই family তে প্রায়ই কয়েকটা size থাকে।
- **Input limit:** Reranker এর একটা সর্বোচ্চ input দৈর্ঘ্য আছে। লম্বা chunk কেটে যায়, তাই chunk size এর সাথে মিলিয়ে বাছো।

### Trade-offs
| বেশি পাবে | বেশি দেবে |
|---|---|
| সঠিক document উপরে, LLM এর ভালো উত্তর | প্রতি প্রশ্নে বাড়তি latency |
| ছোট N, তাই কম LLM token | Reranker এর খরচ বা hardware |
| Negation, শর্তের মতো সূক্ষ্মতা ধরা | আরেকটা system চালানো আর monitor করার দায়িত্ব |

### Alternatives
প্রথম ধাপকে ভালো করা: hybrid search, ভালো chunking, ভালো embedding model। অনেক সময় এতেই যথেষ্ট উন্নতি হয়।

### When should I use it?
Eval এ দেখা গেছে সঠিক document top-20 এ আছে, কিন্তু top-3 এ নেই।

### When should I NOT use it?
- Voice assistant এর মতো খুব কঠোর latency এর জায়গায় (বা খুব ছোট K সহ)।
- Eval এ reranker যোগ করে উন্নতি দেখা যাচ্ছে না।

### Hands-on experiment
একটা API reranker (free tier থাকলে) আর একটা local reranker একই ২০টা বাংলা প্রশ্নে চালাও। Hit rate, গড় latency আর খরচ তিনটা মিলিয়ে একটা table বানাও।

### Production implementation
- Reranker ব্যর্থ হলে (timeout/error) পুরো উত্তর আটকে দিও না। Embedding এর ক্রমেই এগিয়ে যাও (fallback)।
- Reranker এর জন্য আলাদা timeout রাখো।

### Failure cases
Reranker service down হওয়ায় পুরো chatbot বন্ধ, কারণ fallback ছিল না।

### Security concerns
API reranker এ sensitive document পাঠানো মানে data বাইরে যাওয়া। Sensitive data হলে local reranker বাছো।

### Performance concerns
p95 latency মাপো। কিছু প্রশ্নে লম্বা document থাকায় reranking অনেক বেশি সময় নিতে পারে।

### Cost concerns
Reranker এর খরচ বনাম ছোট N এর কারণে LLM token সাশ্রয়, দুটো পাশাপাশি হিসাব করো।

### What did I learn?
- Reranker যোগ করা একটা engineering সিদ্ধান্ত। আগে মাপো, তারপর যোগ করো।

### What can I explain to someone else?
> "Reranker হলো quality control এর একজন অভিজ্ঞ লোক। সে থাকলে ভুল কম হয়, কিন্তু তার বেতন আছে, আর সে সব কিছু দেখলে কাজ ধীর হয়। তাই তাকে শুধু শেষ ধাপের বাছাই করা জিনিসগুলো দেখাও।"

### GitHub Evidence
- [ ] API vs local reranker comparison (hit rate, latency, cost)

### LinkedIn Post
- [ ] —

---

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Embeddings](embeddings.md) · [Root README](../README.md)

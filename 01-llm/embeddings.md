# Phase 1 — LLM Fundamentals: Embeddings (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Embedding model এর নাম, dimension আর দাম বদলায়। এখানের model নামগুলো উদাহরণ, ব্যবহারের আগে docs দেখবে।
>
> 🧪 Hands-on এ Ollama এর embedding model ব্যবহার করা হয়েছে (খরচ শূন্য)। বাংলার জন্য multilingual model লাগবে, যেমন `bge-m3` (`ollama pull bge-m3`)।
>
> 🔗 Embeddings হলো Phase 4 (RAG) এর ভিত্তি, আর পরের chapter Rerankers এর আগের ধাপ।

## Quick Cheat Sheet

| Topic | এক লাইনে |
|---|---|
| [Embedding fundamentals](#embedding-fundamentals) | Text কে সংখ্যার একটা list (vector) এ রূপান্তর, যেটা অর্থ ধরে রাখে |
| [Vector representations](#vector-representations) | অর্থ = জায়গা। কাছাকাছি অর্থের লেখা vector space এ কাছাকাছি থাকে |
| [Similarity](#similarity) | দুইটা vector কতটা কাছে, সেটা মেপে দুই লেখার অর্থ কতটা মেলে তা বোঝা |
| [Cosine similarity](#cosine-similarity) | দুই vector এর মাঝের কোণ দিয়ে মিল মাপা: 1 = একই দিক, 0 = সম্পর্কহীন |
| [Embedding dimensions](#embedding-dimensions) | Vector এ কয়টা সংখ্যা (384, 768, 1536…)। বেশি = বেশি detail, বেশি storage |
| [Embedding model selection](#embedding-model-selection) | ভাষা, quality, dimension, দাম আর input limit দেখে model বাছা |

```text
"পাসওয়ার্ড ভুলে গেছি"  ──► Embedding model ──► [0.12, -0.48, 0.91, ..., 0.05]  (যেমন 1024টা সংখ্যা)
"How to reset password?" ──► Embedding model ──► [0.10, -0.45, 0.88, ..., 0.07]
                                                          │
                                         Cosine similarity ≈ খুব কাছাকাছি ✅
                                (শব্দ এক না, ভাষাও এক না, কিন্তু অর্থ এক)
```

> **Chat model বনাম embedding model:** Chat model (GPT, Claude) text **লেখে**। Embedding model text **লেখে না**, শুধু text কে একটা vector বানায়। এরা আলাদা model, আলাদা API, আলাদা দাম (embedding অনেক সস্তা)।

---

<a id="embedding-fundamentals"></a>

## 1. Embedding Fundamentals

### What is it?
Embedding হলো একটা text (শব্দ, বাক্য, বা পুরো paragraph) এর **অর্থকে সংখ্যার একটা list** এ রূপান্তর করা। এই list কে বলে vector।

### Why does it exist?
Computer লেখার অর্থ বোঝে না, শুধু সংখ্যা বোঝে। আর সাধারণ keyword search অর্থ বোঝে না: "গাড়ি" খুঁজলে "car" বা "যানবাহন" পায় না।

### What problem does it solve?
**অর্থ দিয়ে খোঁজা (semantic search):** User যে শব্দেই প্রশ্ন করুক, অর্থ মিললেই সঠিক document পাওয়া যায়। এটাই RAG এর retrieval অংশের মূল।

### How does it work?
```text
Text ──► Tokenizer ──► Embedding model (Transformer) ──► একটা vector
```
- Model লাখ লাখ text জোড়া দেখে শেখে কোন লেখাগুলোর অর্থ কাছাকাছি।
- Training এর লক্ষ্য: কাছাকাছি অর্থের লেখার vector কাছাকাছি হবে, আলাদা অর্থের লেখার vector দূরে।
- একটা input থেকে **একটাই** vector আসে, input যত লম্বাই হোক (input limit এর মধ্যে)।

### Important concepts
- **Embedding model আলাদা:** `text-embedding-3-small` (OpenAI), `bge-m3`, `nomic-embed-text` এর মতো model শুধু vector বানায়।
- **একই model দিয়ে সব:** Document আর user এর প্রশ্ন, দুটোরই embedding **একই model** দিয়ে বানাতে হবে। আলাদা model এর vector তুলনা করা অর্থহীন।
- **Input limit:** প্রতিটা embedding model এর সর্বোচ্চ input token আছে (যেমন 512 বা 8192)। বেশি হলে কেটে যায়। এজন্যই RAG এ document কে chunk এ ভাগ করতে হয়।
- **Vector database:** লাখ লাখ vector store করে দ্রুত খোঁজার জন্য বিশেষ database (Qdrant, pgvector, Phase 4)।

### Trade-offs
অর্থ ধরে, কিন্তু exact জিনিস (product code, নাম, নম্বর) খোঁজায় keyword search এর চেয়ে দুর্বল।

### Alternatives
Keyword search (BM25), দুটো মিলিয়ে **hybrid search** (Phase 4)।

### When should I use it?
Semantic search, RAG, similar item খোঁজা (similar product, duplicate ticket), classification, clustering।

### When should I NOT use it?
Exact match দরকার: invoice number, phone number, SKU code খোঁজা। এখানে keyword/database query ভালো।

### Hands-on experiment
Ollama দিয়ে embedding বানাও আর দেখো vector দেখতে কেমন:

```ts
const res = await fetch("http://localhost:11434/api/embed", {
  method: "POST",
  body: JSON.stringify({ model: "bge-m3", input: "পাসওয়ার্ড ভুলে গেছি" }),
});
const { embeddings } = await res.json();
console.log(embeddings[0].length);      // vector এ কয়টা সংখ্যা (dimension)
console.log(embeddings[0].slice(0, 5)); // প্রথম ৫টা সংখ্যা
```

### Production implementation
- Document এর embedding একবার বানিয়ে store করো (indexing)। প্রশ্ন আসলে শুধু প্রশ্নের embedding বানাও।
- কোন model আর কোন version দিয়ে embedding বানানো, তা metadata তে রাখো।

### Failure cases
- Document একটা model দিয়ে, প্রশ্ন আরেকটা দিয়ে embed করা → search পুরোপুরি ভুল ফল দেয়, কিন্তু কোনো error দেয় না।
- লম্বা document input limit এ চুপচাপ কেটে গেছে, শেষের অংশ কখনো খুঁজে পাওয়া যায় না।

### Security concerns
- Embedding থেকে মূল text আংশিক পুনরুদ্ধার করা সম্ভব হতে পারে। তাই sensitive data এর embedding কেও sensitive data ধরো।
- Embedding API তে document পাঠানো মানে data provider এর কাছে যাওয়া।

### Performance concerns
Embedding বানানো দ্রুত, কিন্তু লাখ document একবারে embed করতে batch করে পাঠাও।

### Cost concerns
Embedding chat model এর চেয়ে অনেক সস্তা। মূল খরচ আসে বারবার re-embed করা আর vector storage থেকে।

### What did I learn?
- Embedding = অর্থের সংখ্যা রূপ। Search এর ভাষা শব্দ থেকে অর্থে চলে যায়।

### What can I explain to someone else?
> "Embedding হলো প্রতিটা লেখাকে একটা map এর ঠিকানা দেওয়া। একই রকম অর্থের লেখাগুলো একই পাড়ায় থাকে।"

### GitHub Evidence
- [ ] Ollama embedding demo (বাংলা + English)

### LinkedIn Post
- [ ] —

---

<a id="vector-representations"></a>

## 2. Vector Representations

### What is it?
একটা vector হলো সংখ্যার একটা list, যেটাকে একটা বহু-মাত্রিক space এর একটা **বিন্দু বা দিক** হিসেবে ভাবা যায়। Embedding এর ক্ষেত্রে সেই space এ **অবস্থানই অর্থ**।

### Why does it exist?
সংখ্যায় রূপ দিলে অর্থ নিয়ে গণিত করা যায়: দূরত্ব মাপা, কাছের জিনিস খোঁজা, দল বানানো।

### What problem does it solve?
"অর্থের কাছাকাছি" কথাটাকে মাপা যায় এমন জিনিসে রূপ দেয়।

### How does it work?
২-মাত্রার একটা খেলনা উদাহরণ (আসল embedding এ শত-হাজার মাত্রা):

```text
  খেলা ▲
       │      ● ক্রিকেট
       │   ● ফুটবল
       │
       │                       ● ভাত
       │                    ● বিরিয়ানি
       └──────────────────────────────► খাবার
```
"ক্রিকেট" আর "ফুটবল" কাছাকাছি, "বিরিয়ানি" আর "ভাত" কাছাকাছি, দুই দল একে অপরের থেকে দূরে।

### Important concepts
- **প্রতিটা সংখ্যার আলাদা অর্থ নেই:** "৫ নম্বর সংখ্যাটা খেলা বোঝায়" এমন না। অর্থ থাকে সব সংখ্যা মিলিয়ে পুরো দিকের মধ্যে।
- **Classic উদাহরণ:** পুরনো word embedding এ দেখা গিয়েছিল `king − man + woman ≈ queen`, মানে vector এ সম্পর্কও ধরা পড়ে।
- **Multilingual space:** ভালো multilingual model এ "বিড়াল" আর "cat" কাছাকাছি বসে, তাই বাংলা প্রশ্ন দিয়ে English document খোঁজা যায়।

### Trade-offs
পুরো অর্থ কয়েকশো সংখ্যায় চাপাতে গিয়ে কিছু সূক্ষ্মতা হারায় (যেমন negation, নিচে Similarity তে)।

### Alternatives
Sparse vector (keyword ভিত্তিক, যেমন BM25 এর হিসাব), যেখানে প্রতিটা শব্দের জন্য আলাদা জায়গা।

### When should I use it?
অর্থের ভিত্তিতে খোঁজা, দল বানানো, বা তুলনা করা লাগে।

### When should I NOT use it?
যেখানে একটা নির্দিষ্ট শব্দ বা ID থাকা না থাকাই আসল প্রশ্ন।

### Hands-on experiment
১০টা বাক্য নাও (৫টা খেলা, ৫টা খাবার নিয়ে, কিছু বাংলায় কিছু English এ)। সবগুলোর embedding বানিয়ে প্রতিটা জোড়ার similarity বের করো। দেখো খেলার বাক্যগুলো নিজেদের মধ্যে বেশি মেলে কিনা, ভাষা আলাদা হলেও।

### Production implementation
- Vector গুলো সাধারণত **normalize** করে রাখা হয় (দৈর্ঘ্য 1), তাতে similarity হিসাব সহজ আর দ্রুত হয়।
- অনেক model নিজেই normalized vector দেয়, docs দেখে নিশ্চিত হও।

### Failure cases
খুব আলাদা ধরনের data (যেমন code আর কবিতা) এক space এ মিশিয়ে একই model দিয়ে খুঁজলে ফল দুর্বল হয়।

### Security concerns
—

### Performance concerns
লাখ লাখ vector এর মধ্যে নিকটতম খোঁজা ধীর, তাই vector database বিশেষ index (ANN, approximate nearest neighbor) ব্যবহার করে (Phase 4)।

### Cost concerns
—

### What did I learn?
- Embedding এ অর্থ মানে জায়গা, আর মিল মানে কাছাকাছি থাকা।

### What can I explain to someone else?
> "লাইব্রেরিতে একই বিষয়ের বই একই তাকে থাকে। Embedding প্রতিটা লেখাকে তার বিষয় অনুযায়ী সঠিক তাকে বসিয়ে দেয়।"

### GitHub Evidence
- [ ] ১০ বাক্যের similarity matrix

### LinkedIn Post
- [ ] —

---

<a id="similarity"></a>

## 3. Similarity

### What is it?
দুইটা embedding কতটা কাছাকাছি, তার মাপ। মাপ বেশি মানে দুই লেখার অর্থ বেশি মেলে।

### Why does it exist?
Search এর মূল প্রশ্ন: "এই প্রশ্নের সাথে কোন document সবচেয়ে বেশি মেলে?" Similarity সেই উত্তরকে সংখ্যায় দেয়।

### What problem does it solve?
হাজার document কে প্রশ্নের সাথে মিল অনুযায়ী সাজানো (ranking), তারপর সবচেয়ে মেলা কয়েকটা (top-k) বেছে নেওয়া।

### How does it work?
```text
প্রশ্নের vector  ──┐
                   ├──► similarity score ──► সব document কে score অনুযায়ী সাজাও ──► top 5
Document vectors ──┘
```
প্রচলিত মাপ: **Cosine similarity** (পরের topic), **Dot product**, **Euclidean distance**।

### Important concepts
- **Similar ≠ সঠিক উত্তর:** "পাসওয়ার্ড কীভাবে reset করবো?" এর সবচেয়ে মেলা document হতে পারে আরেকজনের একই প্রশ্ন, উত্তর না। মিল বিষয়ে, সমাধানে না।
- **Negation সমস্যা:** "আমি কুকুর ভালোবাসি" আর "আমি কুকুর ঘৃণা করি" প্রায়ই খুব কাছাকাছি আসে। দুটো একই বিষয় নিয়ে, তাই embedding এর চোখে কাছে।
- **Score এর মান model ভেদে আলাদা:** এক model এ 0.8 মানে খুব মিল, আরেক model এ সাধারণ। Threshold প্রতিটা model এর জন্য আলাদা ঠিক করতে হয়।

### Trade-offs
দ্রুত আর সস্তা, কিন্তু সূক্ষ্ম relevance ধরতে দুর্বল। এজন্যই পরে **reranker** লাগে (পরের chapter)।

### Alternatives
Keyword matching; reranker (প্রশ্ন আর document একসাথে পড়ে relevance মাপে, ধীর কিন্তু নির্ভুল)।

### When should I use it?
প্রথম ধাপের খোঁজায়: লাখ document থেকে দ্রুত সম্ভাব্য ৫০টা বাছা।

### When should I NOT use it?
চূড়ান্ত "এটাই সঠিক উত্তর" সিদ্ধান্তে শুধু similarity score এর উপর ভরসা করবে না।

### Hands-on experiment
তিনটা জোড়ার similarity মাপো:
1. "আমি কুকুর ভালোবাসি" বনাম "আমি কুকুর ঘৃণা করি"
2. "আমি কুকুর ভালোবাসি" বনাম "I love dogs"
3. "আমি কুকুর ভালোবাসি" বনাম "আজ বৃষ্টি হবে"

কোন জোড়া সবচেয়ে কাছে আসে দেখে অবাক হতে পারো।

### Production implementation
- Top-k এর সাথে একটা minimum score threshold রাখো, যাতে কোনো মেলা document না থাকলে "পাইনি" বলা যায়।
- Threshold নিজের data দিয়ে test করে ঠিক করো।

### Failure cases
Threshold না থাকায় একদম অপ্রাসঙ্গিক document ও top-5 এ চলে আসে (কারণ সবসময় কেউ না কেউ "সবচেয়ে কাছে" থাকে), আর LLM সেটা দিয়েই উত্তর বানায়।

### Security concerns
কেউ ইচ্ছা করে এমন document ঢোকাতে পারে যা অনেক প্রশ্নের সাথে মেলে, যাতে তার লেখা (বা prompt injection) বারবার retrieve হয়।

### Performance concerns
Brute force এ প্রতিটা প্রশ্নে সব vector এর সাথে তুলনা লাগে। বড় data তে ANN index লাগবে।

### Cost concerns
Similarity হিসাব নিজে প্রায় বিনামূল্যে, খরচ হয় storage আর index এ।

### What did I learn?
- Similarity মানে "একই বিষয়", "একই অর্থ" বা "সঠিক উত্তর" নিশ্চিত না।

### What can I explain to someone else?
> "Similarity হলো দুই মানুষের পছন্দের মিল। দুইজনেরই ক্রিকেট নিয়ে কথা বলতে ভালো লাগে, কিন্তু একজন হয়তো বাংলাদেশ সমর্থক, আরেকজন অন্য দলের।"

### GitHub Evidence
- [ ] Negation আর cross-language similarity experiment

### LinkedIn Post
- [ ] "AI এর চোখে 'ভালোবাসি' আর 'ঘৃণা করি' প্রায় একই কেন?"

---

<a id="cosine-similarity"></a>

## 4. Cosine Similarity

### What is it?
দুইটা vector এর মাঝের **কোণ** দিয়ে মিল মাপার পদ্ধতি। Vector কত লম্বা তা না দেখে, কোন **দিকে** তাকিয়ে আছে তা দেখে।

### Why does it exist?
Embedding এ অর্থ থাকে দিকের মধ্যে। দৈর্ঘ্য অনেক সময় লেখার দৈর্ঘ্য বা অন্য অপ্রাসঙ্গিক জিনিসে প্রভাবিত হয়, তাই দিক মাপাই বেশি নির্ভরযোগ্য।

### What problem does it solve?
লেখা ছোট হোক বা বড়, শুধু অর্থের দিক মিলিয়ে তুলনা।

### How does it work?
```text
cosine(A, B) = (A · B) ÷ (|A| × |B|)

A · B  = প্রতিটা অবস্থানের সংখ্যা গুণ করে যোগ (dot product)
|A|    = vector এর দৈর্ঘ্য
```

ছোট উদাহরণ (২ মাত্রা):

| Vector জোড়া | হিসাব | Cosine | অর্থ |
|---|---|---|---|
| A = [1, 2], B = [2, 4] | একই দিক, B শুধু লম্বা | **1** | একদম একই দিক |
| A = [1, 2], C = [2, −1] | 1×2 + 2×(−1) = 0 | **0** | সম্পর্কহীন (৯০° কোণ) |
| A = [1, 2], D = [−1, −2] | উল্টো দিক | **−1** | সম্পূর্ণ বিপরীত |

### Important concepts
- **Range:** −1 থেকে 1। বাস্তব text embedding এ সাধারণত 0 থেকে 1 এর মধ্যেই থাকে।
- **Normalized vector এ cosine = dot product:** Vector এর দৈর্ঘ্য 1 হলে ভাগ করার দরকার নেই, তাই হিসাব দ্রুত। এজন্যই অনেক vector database dot product ব্যবহার করে।
- **Cosine distance = 1 − cosine similarity:** কিছু database "distance" দেখায়, সেখানে **কম** মানে বেশি মিল। গুলিয়ে ফেলো না।

### Trade-offs
সহজ আর দ্রুত, কিন্তু embedding model নিজে যা ধরতে পারেনি (যেমন negation), cosine তা ঠিক করতে পারে না।

### Alternatives
Dot product (normalized vector এ একই ফল), Euclidean distance।

### When should I use it?
Text embedding তুলনার default পদ্ধতি হিসেবে, যদি model এর docs অন্য কিছু না বলে।

### When should I NOT use it?
Model এর docs যদি নির্দিষ্ট metric (যেমন dot product) বলে দেয়, সেটাই ব্যবহার করো।

### Hands-on experiment
নিজের cosine function লিখে embedding এর সাথে চালাও:

```ts
function cosine(a: number[], b: number[]) {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}

console.log(cosine([1, 2], [2, 4]));  // 1
console.log(cosine([1, 2], [2, -1])); // 0

const res = await fetch("http://localhost:11434/api/embed", {
  method: "POST",
  body: JSON.stringify({
    model: "bge-m3",
    input: ["আমি কুকুর ভালোবাসি", "I love dogs", "আজ বৃষ্টি হবে"],
  }),
});
const { embeddings: [bn, en, rain] } = await res.json();
console.log("বাংলা vs English:", cosine(bn, en).toFixed(3));
console.log("কুকুর vs বৃষ্টি:", cosine(bn, rain).toFixed(3));
```

### Production implementation
নিজে হাতে লাখ vector এ cosine চালাবে না। Vector database এ metric (cosine/dot) সেট করো, সে index দিয়ে দ্রুত খুঁজবে।

### Failure cases
Database এর metric "cosine distance" কিন্তু code ধরে নিয়েছে "similarity", তাই সবচেয়ে **কম** মেলা document কে সবচেয়ে ভালো ভেবে নেওয়া।

### Security concerns
—

### Performance concerns
Normalized vector + dot product হলো সবচেয়ে দ্রুত উপায়।

### Cost concerns
—

### What did I learn?
- Cosine দেখে দিক, দৈর্ঘ্য না। অর্থও থাকে দিকে।

### What can I explain to someone else?
> "দুইজন মানুষ একই দিকে আঙুল তুলছে কিনা, সেটাই cosine। একজনের হাত লম্বা না খাটো, তাতে কিছু যায় আসে না।"

### GitHub Evidence
- [ ] Cosine function + বাংলা/English similarity test

### LinkedIn Post
- [ ] —

---

<a id="embedding-dimensions"></a>

## 5. Embedding Dimensions

### What is it?
একটা embedding vector এ কয়টা সংখ্যা আছে। যেমন 384, 768, 1024, 1536, 3072। প্রতিটা model এর dimension নির্দিষ্ট।

### Why does it exist?
বেশি সংখ্যা মানে অর্থের বেশি সূক্ষ্মতা ধরার জায়গা। কিন্তু প্রতিটা সংখ্যা জায়গা আর হিসাবের সময় খায়।

### What problem does it solve?
Quality আর খরচ/গতির মধ্যে balance ঠিক করার একটা lever।

### How does it work?
| Dimension (উদাহরণ) | উদাহরণ model | প্রতি vector এর size (float32) |
|---|---|---|
| 384 | `all-MiniLM-L6-v2` | ≈ 1.5 KB |
| 768 | `nomic-embed-text` | ≈ 3 KB |
| 1024 | `bge-m3` | ≈ 4 KB |
| 1536 | `text-embedding-3-small` | ≈ 6 KB |
| 3072 | `text-embedding-3-large` | ≈ 12 KB |

হিসাব: প্রতিটা সংখ্যা 4 byte (float32)। 1536 × 4 = 6,144 byte ≈ 6 KB। **১০ লাখ chunk × 6 KB ≈ 6 GB** শুধু vector এর জন্য।

### Important concepts
- **বেশি dimension = সবসময় ভালো না:** ভালোভাবে train করা ছোট model অনেক সময় বড় dimension এর দুর্বল model কে হারায়।
- **Dimension কমানো:** কিছু model এ output dimension কমিয়ে নেওয়া যায় (যেমন OpenAI এর `text-embedding-3` এ `dimensions` parameter), quality সামান্য কমে, storage অনেক কমে।
- **Fixed:** একবার একটা dimension দিয়ে index বানালে, অন্য dimension এর vector সেখানে রাখা যায় না।

### Trade-offs
বেশি dimension = বেশি quality সম্ভাবনা, কিন্তু বেশি storage, বেশি RAM, ধীর search।

### Alternatives
ছোট dimension এর model, dimension কমানো, quantization (vector এর সংখ্যা ছোট format এ রাখা)।

### When should I use it?
Storage আর search speed এর পরিকল্পনা করার সময়, বিশেষ করে লাখ লাখ document হলে।

### When should I NOT use it?
ছোট project (কয়েক হাজার document) এ dimension নিয়ে মাথা ঘামানোর দরকার নেই, quality দেখে model বাছো।

### Hands-on experiment
একই ২০টা প্রশ্ন-document জোড়ায় একটা ছোট (যেমন `nomic-embed-text`, 768) আর একটা বড় (`bge-m3`, 1024) model চালাও। সঠিক document top-3 এ কতবার এলো তুলনা করো, সাথে বাংলা প্রশ্নে কোনটা ভালো।

### Production implementation
- Document সংখ্যা × dimension × 4 byte দিয়ে আগেই storage হিসাব করো।
- Index এর dimension config এ রাখো, model এর সাথে মিলিয়ে।

### Failure cases
Model বদলে নতুন dimension এর vector পুরনো index এ ঢোকাতে গিয়ে error, বা পুরো index আবার বানাতে হওয়া।

### Security concerns
—

### Performance concerns
Dimension দ্বিগুণ = search এর হিসাব আর memory প্রায় দ্বিগুণ।

### Cost concerns
Managed vector database এ storage আর RAM এর বিল সরাসরি dimension এর সাথে বাড়ে।

### What did I learn?
- Dimension একটা খরচের সিদ্ধান্তও, শুধু quality এর না।

### What can I explain to someone else?
> "Dimension হলো ছবির resolution এর মতো। বেশি pixel এ বেশি detail, কিন্তু file বড়। সব কাজে 4K ছবি লাগে না।"

### GitHub Evidence
- [ ] Small vs large embedding model: retrieval accuracy table

### LinkedIn Post
- [ ] —

---

<a id="embedding-model-selection"></a>

## 6. Embedding Model Selection

### What is it?
নিজের কাজের জন্য সঠিক embedding model বেছে নেওয়া।

### Why does it exist?
Embedding model এর quality ভাষা আর domain ভেদে অনেক আলাদা। ভুল model = RAG এর retrieval দুর্বল = LLM ভুল context পায় = ভুল উত্তর।

### What problem does it solve?
Search আর RAG এর quality এর একদম গোড়ার সিদ্ধান্ত।

### How does it work?
যা দেখে বাছবে:

| বিষয় | প্রশ্ন |
|---|---|
| **ভাষা** | বাংলা support করে? বাংলা প্রশ্ন দিয়ে English document খুঁজতে পারে? |
| **Quality** | নিজের data তে সঠিক document খুঁজে পায়? |
| **Input limit** | Chunk এর সাইজ ধরে? (512 বনাম 8192 token) |
| **Dimension** | Storage আর speed এ মানায়? |
| **দাম / hosting** | API (প্রতি token) নাকি local/self-host (খরচ শূন্য, GPU লাগতে পারে)? |
| **Privacy** | Data বাইরে পাঠানো যাবে? |

### Important concepts
- **MTEB leaderboard:** Embedding model এর benchmark। Shortlist বানাতে কাজের, কিন্তু Benchmark post এর শিক্ষা এখানেও খাটে: চূড়ান্ত সিদ্ধান্ত নিজের data দিয়ে।
- **Model বদলানো দামি:** নতুন model এ গেলে সব document আবার embed করতে হয়।
- **Query/document prefix:** কিছু model এ প্রশ্ন আর document এর আগে আলাদা prefix লাগে (যেমন `query:` / `passage:`)। না দিলে quality কমে। Model card পড়ো।
- **সব provider এর নিজস্ব embedding নেই:** যেমন Anthropic নিজে embedding model দেয় না, অন্য provider (যেমন Voyage AI) এর পরামর্শ দেয়। Chat model আর embedding model আলাদা company র হতেই পারে।

### Trade-offs
API model = সহজ আর শক্তিশালী, কিন্তু খরচ আর data বাইরে যায়; local model = privacy আর শূন্য খরচ, কিন্তু quality আর infra নিজের দায়িত্ব।

### Alternatives
নিজের data দিয়ে embedding model fine-tune করা (advanced, Python Track E)।

### When should I use it?
RAG বা semantic search শুরু করার আগে, আর quality সমস্যা দেখা দিলে।

### When should I NOT use it?
৩টা model এর মধ্যে ছোটখাটো পার্থক্যে বারবার বদলাবে না, প্রতিবার re-embed এর খরচ আছে।

### Hands-on experiment
**ছোট retrieval eval:**
1. নিজের ২০টা বাংলা প্রশ্ন আর প্রতিটার সঠিক document বানাও।
2. ২–৩টা embedding model দিয়ে সব document আর প্রশ্ন embed করো।
3. প্রতিটা প্রশ্নে সঠিক document top-3 এ এলো কিনা গোনো।
4. Model প্রতি সফলতার হার (hit rate) তুলনা করো।

এটা Benchmark post এর "নিজের test" ধারণার বাস্তব রূপ, embedding এর জন্য।

### Production implementation
- Model নাম আর version config এ রাখো, প্রতিটা vector এর metadata তে লিখে রাখো।
- Model বদলালে নতুন index বানাও, পুরনোটা পাশে রেখে তুলনা করো, তারপর switch করো।

### Failure cases
- English-only model দিয়ে বাংলা document embed করা: সব বাংলা vector প্রায় একই জায়গায় জমা হয়, search অর্থহীন।
- Prefix না দেওয়ায় quality চুপচাপ কমে যাওয়া।

### Security concerns
Sensitive document হলে local embedding model বেছে নাও, data বাইরে যাবে না।

### Performance concerns
Local model এ CPU তে বড় batch embed করা ধীর। একবারের indexing background job এ চালাও।

### Cost concerns
API embedding এর দাম কম, কিন্তু লাখ document re-embed করলে বিল জমে। Local model এ শুধু হার্ডওয়্যার আর সময়।

### What did I learn?
- RAG এর quality শুরু হয় embedding model বাছাই থেকে, আর বাংলার জন্য multilingual model বাধ্যতামূলক।

### What can I explain to someone else?
> "Embedding model হলো লাইব্রেরিয়ান। বাংলা না জানা লাইব্রেরিয়ান বাংলা বইগুলো ঠিক তাকে রাখতে পারবে না, যত ভালো লাইব্রেরিয়ানই হোক।"

### GitHub Evidence
- [ ] বাংলা retrieval eval (২–৩ model, hit rate table)

### LinkedIn Post
- [ ] "বাংলা RAG বানাতে প্রথম ভুলটা হয় embedding model এ"

---

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Multimodal Inputs](multimodal-inputs.md) · [Root README](../README.md)

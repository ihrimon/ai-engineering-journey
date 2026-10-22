# Phase 4 — RAG & Knowledge Systems: Chunking (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Chunk size/overlap এর সংখ্যাগুলো **শুরু করার উদাহরণ**, সঠিক সংখ্যা নিজের data আর eval থেকে বের করতে হয়। Local practice: Ollama `bge-m3` (embedding) + `qwen2.5:3b`।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Fixed-size chunking](#fixed-size-chunking) | নির্দিষ্ট token সংখ্যায় কাটা: সহজ, দ্রুত, কিন্তু অর্থ ভাঙতে পারে |
| [Semantic chunking](#semantic-chunking) | Embedding similarity দেখে যেখানে topic বদলায় সেখানে কাটা |
| [Structural chunking](#structural-chunking) | Document এর নিজস্ব গঠন (heading, section, code block, table) মেনে কাটা |
| [Late chunking](#late-chunking) | আগে পুরো document embed (long-context model), পরে chunk এর vector বের করা |
| [Chunk overlap](#chunk-overlap) | পাশাপাশি chunk এর মাঝে কিছু token share, যাতে boundary তে অর্থ না হারায় |
| [Chunk metadata](#chunk-metadata) | Source, title, section, page, date, ACL: filter, citation আর context এর জন্য |

```text
Document ──► Parse ──► Chunk ──► (metadata যোগ) ──► Embed ──► Vector DB
                         ▲
        RAG quality এর সবচেয়ে বড় lever প্রায়ই এখানেই:
        "Garbage chunks in → garbage answers out"
```

---

<a id="fixed-size-chunking"></a>

## 1. Fixed-Size Chunking

### What is it?
Text কে নির্দিষ্ট সাইজের টুকরোয় কাটা (যেমন ৫০০ token), সাধারণত overlap সহ। জনপ্রিয় variant: **recursive character/token splitter**, যেটা আগে paragraph (`\n\n`), তারপর line, তারপর sentence boundary তে কাটার চেষ্টা করে।

### Why does it exist?
Embedding model এর input limit আছে, আর ছোট টুকরো বেশি focused vector দেয়। সবচেয়ে সহজ উপায় হলো সমান মাপে কাটা।

### What problem does it solve?
যেকোনো text কে দ্রুত, predictable সাইজের chunk এ ভাগ করে, কোনো model call ছাড়াই।

### How does it work?
```text
[────500────][────500────][────500────]
         [50 overlap]  [50 overlap]
Recursive: "\n\n" → "\n" → ". " → " " — বড় separator দিয়ে আগে চেষ্টা
```

### Important concepts
- **Token-based** মাপো, character না (বাংলায় পার্থক্য বিশাল)।
- **Chunk size trade-off**: ছোট (১০০–৩০০) = precise retrieval কিন্তু context কম; বড় (৮০০–১৫০০) = বেশি context কিন্তু noisy vector।
- **Recursive splitter** সাধারণত naive fixed split এর চেয়ে অনেক ভালো baseline।

### Trade-offs
সহজ ও দ্রুত, কিন্তু বাক্য, table বা code এর মাঝখানে কেটে অর্থ নষ্ট করতে পারে।

### Alternatives
Structural, semantic, late chunking।

### When should I use it?
প্রথম baseline হিসেবে, আর plain unstructured text (transcript, chat log) এ।

### When should I NOT use it?
Code, table, legal clause, structured markdown/HTML এ। এখানে structure ভাঙলে অর্থ হারায়।

### Hands-on experiment
```ts
import { getEncoding } from "js-tiktoken";
const enc = getEncoding("o200k_base");

function fixedChunks(text: string, size = 400, overlap = 50) {
  const ids = enc.encode(text);
  const out: string[] = [];
  for (let i = 0; i < ids.length; i += size - overlap) out.push(enc.decode(ids.slice(i, i + size)));
  return out;
}
```
একই document এ size = 200, 500, 1000 দিয়ে chunk বানিয়ে ২০টা প্রশ্নে retrieval hit rate তুলনা করো।

### Production implementation
- Chunk size config এ রাখো (document type অনুযায়ী আলাদা)।
- প্রতিটা chunk এর `token_count` store করো।
- Chunking strategy version করো (`chunker: "recursive-v2-500-50"`), বদলালে re-index।

### Failure cases
- Table এর header এক chunk এ, data অন্য chunk এ, ফলে data এর অর্থ নেই।
- "দাম ৫০০ টাকা নয়" বাক্যটা "দাম ৫০০ টাকা" | "নয়" এভাবে কাটা।

### Security concerns
Chunk boundary তে ACL হারাবে না। প্রতিটা chunk এ document এর permission metadata কপি করো।

### Performance concerns
খুব ছোট chunk মানে অনেক বেশি vector, তাই index বড় আর search ধীর।

### Cost concerns
Embedding cost ≈ মোট token × (1 + overlap ratio)। Chunk সংখ্যা বেশি হলে storage খরচ বাড়ে।

### What did I learn?
- Fixed-size = ভালো baseline, কিন্তু শেষ সমাধান না।

### What can I explain to someone else?
> "বই কে প্রতি ১০ পাতায় ছিঁড়ে আলাদা করা। সহজ, কিন্তু কোনো অধ্যায় মাঝখান থেকে ছিঁড়ে যেতে পারে।"

### GitHub Evidence
- [ ] Chunk size vs retrieval hit-rate experiment

### LinkedIn Post
- [ ] "RAG এর chunk size কত হবে? উত্তর: measure করো"

---

<a id="semantic-chunking"></a>

## 2. Semantic Chunking

### What is it?
প্রতিটা sentence (বা ছোট group) embed করে পাশাপাশি sentence এর similarity মাপা, আর যেখানে similarity হঠাৎ কমে যায় (topic বদলায়) সেখানে chunk boundary বসানো।

### Why does it exist?
Fixed size অর্থ বোঝে না। একটা topic এর মাঝখানে কাটতে পারে, বা দুইটা আলাদা topic এক chunk এ জুড়ে দিতে পারে।

### What problem does it solve?
প্রতিটা chunk এ একটাই coherent ধারণা থাকে, ফলে embedding পরিষ্কার আর retrieval আরও precise।

### How does it work?
```text
s1 s2 s3 | s4 s5 | s6 s7 s8
sim: 0.82 0.79 [0.31] 0.85 [0.28] 0.80 0.77
              ▲ breakpoint       ▲ breakpoint  (threshold বা percentile দিয়ে)
```

### Important concepts
- **Breakpoint threshold**: fixed (যেমন < 0.5) বা percentile (সবচেয়ে কম ৫% gap)।
- **Min/max size guard**: খুব ছোট বা বিশাল chunk এড়াতে।
- **Buffer**: sentence এর আগে-পরে ১টা sentence যোগ করে embed করলে noise কমে।
- LLM-based chunking (model কে বলো কোথায় কাটবে) আরেকটা variant, যা আরও দামি।

### Trade-offs
ভালো coherence, কিন্তু indexing এ প্রতিটা sentence embed করতে হয়, তাই ধীর ও দামি। আর সবসময় fixed এর চেয়ে ভালো ফল দেয় না, eval করে দেখতে হয়।

### Alternatives
Structural chunking (document এর heading ই যদি topic boundary হয়, তাহলে সেটাই সস্তা)।

### When should I use it?
Heading-less লম্বা text: transcript, article, essay, যেখানে topic ধীরে ধীরে বদলায়।

### When should I NOT use it?
ভালো structure আছে এমন document (markdown, docs site) বা খুব বড় corpus যেখানে indexing cost জরুরি।

### Hands-on experiment
```ts
import ollama from "ollama";
const cos = (a: number[], b: number[]) => {
  let d = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) { d += a[i] * b[i]; na += a[i] ** 2; nb += b[i] ** 2; }
  return d / Math.sqrt(na * nb);
};

async function semanticChunks(sentences: string[], threshold = 0.6) {
  const { embeddings } = await ollama.embed({ model: "bge-m3", input: sentences });
  const chunks: string[][] = [[sentences[0]]];
  for (let i = 1; i < sentences.length; i++) {
    if (cos(embeddings[i - 1], embeddings[i]) < threshold) chunks.push([]);
    chunks.at(-1)!.push(sentences[i]);
  }
  return chunks.map((c) => c.join(" "));
}
```
একটা podcast transcript এ fixed vs semantic chunking এর retrieval তুলনা করো।

### Production implementation
- Indexing pipeline এ batch embedding।
- Min 100 / max 800 token guard।
- Threshold eval set দিয়ে tune করো।

### Failure cases
- বাংলা sentence split ভুল (`।` চিনছে না), তাই পুরো paragraph একটা "sentence"।
- Threshold খুব বেশি, ফলে প্রতিটা sentence আলাদা chunk।

### Security concerns
Fixed chunking এর মতোই, ACL metadata প্রতিটা chunk এ।

### Performance concerns
Indexing এর সময় sentence-level embedding, তাই ৫–২০ গুণ বেশি embedding call।

### Cost concerns
API embedding হলে indexing খরচ বাড়ে। Local model (bge-m3) এ শুধু সময়।

### What did I learn?
- "Smart" chunking সবসময় জেতে না। Eval ছাড়া সিদ্ধান্ত নেবে না।

### What can I explain to someone else?
> "বই কে পাতা গুনে না, গল্পের দৃশ্য বদলানোর জায়গায় ভাগ করা।"

### GitHub Evidence
- [ ] Fixed vs semantic chunking eval

### LinkedIn Post
- [ ] "Semantic chunking কি সত্যিই ভালো? আমার experiment"

---

<a id="structural-chunking"></a>

## 3. Structural Chunking

### What is it?
Document এর নিজস্ব গঠন মেনে কাটা: Markdown/HTML heading, section, list, table, code এর function/class, PDF এর layout block। একে document-aware বা layout-aware chunking ও বলে।

### Why does it exist?
লেখক ইতিমধ্যে document কে অর্থপূর্ণ অংশে ভাগ করে রেখেছে (heading, section)। সেই structure ব্যবহার করা সবচেয়ে সস্তা আর নির্ভরযোগ্য "semantic" boundary।

### What problem does it solve?
Table, code block, clause, section অক্ষত থাকে, আর chunk এর সাথে heading path (context) যোগ করা যায়।

### How does it work?
```text
# Refund Policy               → heading path: "Refund Policy > Digital Products"
## Digital Products
  paragraph...                 → chunk = heading path + section content
  | table |                    → table অক্ষত একটা chunk (বা row-group)
```
Code এর জন্য AST-based (tree-sitter): function/class ধরে কাটা।

### Important concepts
- **Heading breadcrumb** chunk এর শুরুতে যোগ করো: chunk আলাদা থাকলেও বোঝা যায় এটা কোন section এর।
- **Section বড় হলে** ভেতরে আবার recursive split।
- **Atomic elements**: table, code block, list কখনো মাঝখানে কাটবে না (দরকার হলে header repeat করে row ভাগ)।
- Parsing quality এর উপর নির্ভরশীল (দেখো [document-parsing.md](document-parsing.md))।

### Trade-offs
Excellent quality, কিন্তু format-specific code লাগে, আর section সাইজ অসমান হয়।

### Alternatives
Fixed recursive (structure না থাকলে), semantic।

### When should I use it?
Docs site, markdown, HTML, legal/contract, code repository, well-formatted PDF।

### When should I NOT use it?
Structure নেই এমন text (chat log, OCR noise)।

### Hands-on experiment
```ts
function markdownChunks(md: string) {
  const out: { path: string; text: string }[] = [];
  const path: string[] = [];
  let buf: string[] = [];
  const flush = () => { if (buf.join("").trim()) out.push({ path: path.join(" > "), text: buf.join("\n") }); buf = []; };
  for (const line of md.split("\n")) {
    const h = /^(#{1,6})\s+(.*)/.exec(line);
    if (h) { flush(); path.splice(h[1].length - 1); path[h[1].length - 1] = h[2]; }
    else buf.push(line);
  }
  flush();
  return out.map((c) => `${c.path}\n\n${c.text}`);
}
```
এই repo র নিজের markdown file গুলোতে চালিয়ে দেখো।

### Production implementation
- Format অনুযায়ী chunker router: `.md` → markdown, `.ts/.py` → AST, PDF → layout parser।
- Heading path metadata হিসেবে ও রাখো (filter/citation এর জন্য)।

### Failure cases
- PDF parse এ heading চেনা যায়নি, ফলে পুরো ৫০ পাতা একটা section।
- বিশাল table কে এক chunk রাখায় embedding limit পার।

### Security concerns
Section-level permission থাকলে (যেমন "Internal only" section) সেটা metadata তে রাখো।

### Performance concerns
Parsing ধীর হতে পারে (PDF/OCR)। Chunking নিজে দ্রুত।

### Cost concerns
অতিরিক্ত LLM call নেই, তাই সস্তা।

### What did I learn?
- Structure আছে? তাহলে সেটাই সবচেয়ে ভালো chunk boundary।

### What can I explain to someone else?
> "বই কে অধ্যায় আর উপ-অধ্যায় ধরে ভাগ করা, প্রতিটা অংশের উপরে লিখে রাখা 'অধ্যায় ৩ > অনুচ্ছেদ ২'।"

### GitHub Evidence
- [ ] Markdown/heading-aware chunker + breadcrumb

### LinkedIn Post
- [ ] "Document এর structure ই সেরা chunking strategy"

---

<a id="late-chunking"></a>

## 4. Late Chunking

### What is it?
প্রচলিত পদ্ধতিতে আগে কাটো, তারপর প্রতিটা chunk আলাদা embed করো। **Late chunking** এ উল্টো: long-context embedding model দিয়ে **আগে পুরো document** (বা বড় অংশ) process করো, token-level embedding পাও, **তারপর** chunk boundary অনুযায়ী সেই token embedding গুলো pool (average) করে chunk vector বানাও। (Jina AI, 2024)

### Why does it exist?
আলাদা chunk embed করলে প্রতিটা chunk অন্যগুলোর context জানে না। যেমন "এটি ১৯৯৯ সালে প্রতিষ্ঠিত" chunk জানে না "এটি" মানে কোন company।

### What problem does it solve?
Pronoun/reference সমস্যা ("it", "এই নীতি") সমাধান করে। প্রতিটা chunk vector এ পুরো document এর context মিশে থাকে।

### How does it work?
```text
Naive:   chunk1 → embed   chunk2 → embed   (একে অপরকে দেখে না)
Late:    [whole doc] → long-context encoder → token embeddings
          → pool(tokens of chunk1), pool(tokens of chunk2)  (প্রতিটা অন্যদের context সহ)
```

### Important concepts
- **Long-context embedding model লাগে** (যেমন ৮K+ token) আর token-level output access।
- **Contextual retrieval** (Anthropic, 2024) একই সমস্যার আরেক সমাধান: প্রতিটা chunk এর আগে LLM দিয়ে ছোট context লেখা ("এই chunk টা Acme এর 2023 refund policy থেকে...")।
- Document model এর context এর চেয়ে বড় হলে বড় window ধরে late chunking।

### Trade-offs
ভালো contextual vector, ছোট chunk রেখেও। কিন্তু নির্দিষ্ট model ও pipeline লাগে, সব API তে token-level embedding পাওয়া যায় না।

### Alternatives
Contextual retrieval (LLM দিয়ে chunk prefix), heading breadcrumb যোগ করা (সস্তা ও প্রায়ই যথেষ্ট), parent-document retrieval।

### When should I use it?
অনেক cross-reference আছে এমন লম্বা document: report, paper, policy।

### When should I NOT use it?
ছোট independent document (FAQ entry, product listing), বা embedding API token-level output না দিলে।

### Hands-on experiment
Contextual retrieval এর সহজ version করো: প্রতিটা chunk এর আগে `"Document: <title>. Section: <heading>."` যোগ করে embed করো, আর ছাড়া embed করো। Pronoun-ভারী প্রশ্নে ("এর প্রতিষ্ঠা কবে?") retrieval তুলনা করো।

### Production implementation
- Embedding provider late chunking support করলে (যেমন Jina এর API তে option) সেটা ব্যবহার।
- নাহলে contextual prefix (LLM-generated বা template) + prompt caching দিয়ে খরচ কমানো।

### Failure cases
Document model এর context window এর চেয়ে বড়, তাই চুপচাপ কেটে গেছে, শেষের chunk গুলোর context নেই।

### Security concerns
Contextual prefix LLM দিয়ে বানালে document এর injection prefix এ ঢুকতে পারে। Template-based prefix নিরাপদ।

### Performance concerns
Long-context encoding ধীর, কিন্তু শুধু indexing এর সময়।

### Cost concerns
Late chunking এ embedding token প্রায় একই। LLM-based contextual prefix এ প্রতি chunk এ LLM call (caching দিয়ে কমানো যায়)।

### What did I learn?
- Chunk এর vector এ "আশেপাশের context" মেশানো retrieval quality বাড়ায়।

### What can I explain to someone else?
> "একটা বাক্য আলাদা করে পড়লে 'সে' কে বোঝা যায় না। পুরো গল্প পড়ে তারপর বাক্যটা আলাদা করলে মাথায় থাকে 'সে' মানে কে।"

### GitHub Evidence
- [ ] Contextual prefix vs plain chunk retrieval test

### LinkedIn Post
- [ ] "Chunk কে context দাও: late chunking আর contextual retrieval"

---

<a id="chunk-overlap"></a>

## 5. Chunk Overlap

### What is it?
পাশাপাশি দুইটা chunk এর মাঝে কিছু token (যেমন ১০–২০%) দুইটাতেই রাখা, যাতে boundary তে পড়া তথ্য অন্তত একটা chunk এ পুরো থাকে।

### Why does it exist?
Fixed boundary প্রায়ই একটা গুরুত্বপূর্ণ বাক্য বা ধারণাকে দুই ভাগ করে ফেলে।

### What problem does it solve?
Boundary তে তথ্য হারানো কমায়, আর প্রতিটা chunk একটু বেশি context পায়।

### How does it work?
```text
chunk1: [ A A A A A B B ]
chunk2:           [ B B C C C C C ]   ← "B B" দুইটাতেই
```

### Important concepts
- **সাধারণ শুরু**: chunk size এর ১০–২০%।
- **Structural chunking এ প্রায়ই overlap লাগে না**, কারণ boundary ইতিমধ্যে অর্থপূর্ণ।
- **Duplicate retrieval**: overlap এর কারণে পাশাপাশি দুই chunk একসাথে আসতে পারে। Dedup/merge করো।
- **Sentence-aligned overlap**: token এর বদলে শেষ ১–২ sentence repeat।

### Trade-offs
বেশি overlap = boundary safety কিন্তু বেশি storage, embedding cost আর duplicate result।

### Alternatives
Parent-document retrieval / sentence window (retrieve ছোট chunk, context এ আশেপাশের বড় window দাও), structural chunking।

### When should I use it?
Fixed/recursive chunking এ প্রায় সবসময়।

### When should I NOT use it?
Atomic unit (FAQ entry, product, table row) এ। এখানে overlap মানে অন্য item এর তথ্য মিশে যাওয়া।

### Hands-on experiment
Overlap 0%, 10%, 25% দিয়ে index বানাও। Boundary-crossing প্রশ্ন (যেগুলোর উত্তর ঠিক কাটার জায়গায়) বানিয়ে hit rate দেখো।

### Production implementation
- Retrieval এর পর adjacent chunk merge: একই document এর পাশাপাশি chunk একসাথে এলে জুড়ে দাও।
- `chunk_index` metadata রাখো যাতে neighbor বের করা যায়।

### Failure cases
২৫% overlap এ top-5 এর ৩টা আসলে একই অংশ, তাই diversity কম।

### Security concerns
নেই (overlap একই document এর ভেতরে)।

### Performance concerns
বেশি chunk মানে বড় index।

### Cost concerns
২০% overlap ≈ ২০% বেশি embedding আর storage।

### What did I learn?
- Overlap হলো boundary problem এর সস্তা bandage। ভালো boundary থাকলে লাগে না।

### What can I explain to someone else?
> "Panorama photo তোলার সময় প্রতিটা ছবি আগেরটার কিছু অংশ সহ তুলতে হয়, যাতে জোড়া লাগানোর সময় কিছু বাদ না পড়ে।"

### GitHub Evidence
- [ ] Overlap % vs boundary-question hit rate

### LinkedIn Post
- [ ] —

---

<a id="chunk-metadata"></a>

## 6. Chunk Metadata

### What is it?
প্রতিটা chunk এর সাথে রাখা structured তথ্য: `doc_id, source_url, title, heading_path, page, chunk_index, created_at/updated_at, language, doc_type, tenant_id, acl/roles, version`।

### Why does it exist?
Vector শুধু "অর্থ" ধরে। কিন্তু production এ filter করতে হয় (এই tenant, এই বছর, এই product), citation দিতে হয়, আর permission enforce করতে হয়।

### What problem does it solve?
- **Filtering**: "শুধু ২০২৬ এর policy" বা "শুধু আমার company এর doc"।
- **Citation**: উত্তরের সাথে source link/page।
- **Security**: ACL দিয়ে unauthorized chunk বাদ।
- **Freshness/re-index**: কোন chunk পুরনো তা বোঝা।

### How does it work?
```json
{
  "id": "policy-2026#12",
  "text": "Refund Policy > Digital Products\n\nডিজিটাল পণ্যে ৭ দিনের মধ্যে ...",
  "metadata": {
    "doc_id": "policy-2026", "title": "Refund Policy", "page": 3, "chunk_index": 12,
    "tenant_id": "acme", "acl": ["support", "admin"], "lang": "bn", "updated_at": "2026-09-01"
  }
}
```

### Important concepts
- **Filterable fields index করো** (vector DB তে payload index)।
- **Metadata in text vs metadata as field**: title/heading text এ যোগ করলে embedding এ ঢোকে; date/tenant field হিসেবে রাখলে filter হয়।
- **LLM-extracted metadata**: summary, keywords, entities, প্রশ্ন (hypothetical questions)।
- **Consistent schema** সব source এ।

### Trade-offs
বেশি metadata = শক্তিশালী filter/citation, কিন্তু extraction খরচ, storage, আর schema maintain করার কাজ।

### Alternatives
শুধু `doc_id` রেখে বাকি তথ্য আলাদা DB থেকে join।

### When should I use it?
সবসময়। অন্তত `doc_id, source, chunk_index, tenant/acl, updated_at`।

### When should I NOT use it?
Metadata এ sensitive তথ্য রাখবে না যা prompt এ গেলে leak হবে (যেমন internal reviewer note)।

### Hands-on experiment
একই index এ "২০২৪ vs ২০২৬ policy" দুই version রাখো। Metadata filter (`year = 2026`) ছাড়া আর সহ প্রশ্ন করো, পুরনো policy থেকে উত্তর আসছে কিনা দেখো।

### Production implementation
- Ingestion pipeline এ schema validation (zod)।
- Retrieval এ **mandatory filter**: `tenant_id` আর `acl` সবসময় (code এ, model এর হাতে না)।
- Citation format: `[title, p.3](source_url#page=3)`।

### Failure cases
- `tenant_id` filter ভুলে যাওয়ায় অন্য company এর document উত্তরে এসেছে (data breach)।
- Document update হয়েছে কিন্তু পুরনো chunk মুছে ফেলা হয়নি, তাই দুই version conflict।

### Security concerns
ACL filtering retrieval এর **আগে/সময়** (pre-filter) করো, LLM এর পরে না। Post-filter এ LLM ইতিমধ্যে দেখে ফেলেছে।

### Performance concerns
Selective filter (খুব কম match) কিছু vector index এ ধীর হতে পারে। DB এর filtered search behavior জেনে নাও।

### Cost concerns
LLM দিয়ে metadata extract করলে ingestion খরচ। Deterministic যা পাওয়া যায় আগে সেটা নাও।

### What did I learn?
- Metadata ছাড়া RAG = লাইব্রেরি যেখানে বইয়ের গায়ে নাম, তারিখ, তাক নম্বর কিছুই নেই।

### What can I explain to someone else?
> "প্রতিটা ফাইলের গায়ে label: কোন বিভাগের, কবেকার, কে দেখতে পারবে। খোঁজার সময় label দিয়ে ছাঁকো, তারপর ভেতরে পড়ো।"

### GitHub Evidence
- [ ] Metadata schema + tenant/ACL pre-filter test

### LinkedIn Post
- [ ] "RAG এর গোপন নায়ক: chunk metadata"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

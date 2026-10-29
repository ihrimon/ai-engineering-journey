# Phase 4 — RAG & Knowledge Systems: Multimodal RAG (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 🔗 ভিত্তি: [multimodal-inputs.md](../01-llm-fundamentals/multimodal-inputs.md) (Phase 1) আর [document-parsing.md](document-parsing.md)। Image token খরচ বেশি, তাই প্রতিটা experiment এ token/cost মাপবে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Image retrieval](#image-retrieval) | Text প্রশ্ন দিয়ে ছবি খোঁজা (বা ছবি দিয়ে ছবি): caption বা multimodal embedding |
| [Table retrieval](#table-retrieval) | প্রশ্নের জন্য সঠিক table (ও সঠিক row) খুঁজে আনা |
| [Chart understanding](#chart-understanding) | Retrieve করা chart থেকে answer time এ VLM দিয়ে প্রশ্নের উত্তর |
| [Document vision](#document-vision) | Parse না করে পুরো পাতার **ছবি** index ও retrieve (ColPali ধরনের) |
| [Multimodal embeddings](#multimodal-embeddings) | Text আর image একই vector space এ (CLIP, SigLIP, multimodal embed API) |

```text
                      তিনটা মূল কৌশল
1. Text-ify:   image/table/chart → text (caption, markdown) → সাধারণ text RAG
2. Shared space: multimodal embedding → text query দিয়ে সরাসরি image vector খোঁজা
3. Page-as-image: পুরো পাতা ছবি হিসেবে embed (ColPali) → retrieve page → VLM উত্তর দেয়

query ──► retrieve (text chunks + images + page images) ──► VLM (text + images) ──► answer
```

---

<a id="image-retrieval"></a>

## 1. Image Retrieval

### What is it?
প্রশ্নের (সাধারণত text) জন্য relevant image খুঁজে আনা: product photo, diagram, screenshot। দুই পথ: (১) image এর caption/description text এ index, (২) multimodal embedding দিয়ে সরাসরি image vector।

### Why does it exist?
অনেক উত্তর ছবিতে, আর অনেক সময় user ছবিই দেখতে চায় ("লাল রঙের sneakers দেখাও")।

### What problem does it solve?
Visual content এর search, আর উত্তরের সাথে প্রাসঙ্গিক ছবি দেখানো।

### How does it work?
```text
Path A (caption): image → VLM description → text embed → text search
Path B (shared):  image → CLIP/SigLIP image encoder → vector
                  query → same model's text encoder → vector → ANN
Reverse: image query → image encoder → similar images
```

### Important concepts
- **Caption path** text-only infra তে চলে, ব্যাখ্যাযোগ্য, কিন্তু caption এর মান এর উপর নির্ভরশীল।
- **Shared space path** caption ছাড়াই কাজ করে, কিন্তু সূক্ষ্ম text (diagram এর label) ভালো ধরে না।
- **Hybrid**: দুটো চালিয়ে RRF।
- Image এর সাথে metadata (source doc, page, alt text)।

### Trade-offs
Caption = ভালো জটিল diagram এ, কিন্তু indexing এ VLM খরচ। CLIP = সস্তা ও দ্রুত, কিন্তু text-heavy image এ দুর্বল।

### Alternatives
শুধু alt text/filename (দুর্বল), manual tagging।

### When should I use it?
E-commerce, manual/docs with screenshot, media library।

### When should I NOT use it?
Image গুলো decorative বা প্রশ্নের সাথে অপ্রাসঙ্গিক।

### Hands-on experiment
২০টা product image নাও। Path A (local VLM caption + bge-m3) আর Path B (CLIP, যেমন `@xenova/transformers` দিয়ে browser/Node এ) দুইভাবে index করো। ১০টা text query তে Recall@3 তুলনা করো।

### Production implementation
Image store (S3) + vector (caption vector + image vector, named vectors), thumbnail URL উত্তরে।

### Failure cases
CLIP "invoice with total 5000" query তে যেকোনো invoice দেয়, সংখ্যা পড়তে পারে না।

### Security concerns
User-uploaded image এ NSFW/sensitive content, moderation। Image এর ভেতরের text injection।

### Performance concerns
Image encode দ্রুত (CLIP)। VLM caption ধীর (offline)।

### Cost concerns
VLM caption = image token খরচ × image সংখ্যা।

### What did I learn?
ছবি খোঁজার দুই রাস্তা: ছবিকে কথায় বলা, অথবা কথা আর ছবিকে একই ভাষায় আনা।

### What can I explain to someone else?
> "Google Photos এ 'beach' লিখলে সমুদ্রের ছবি আসে। ছবিগুলোতে কেউ 'beach' লেখেনি, model ছবি আর শব্দকে একই মানচিত্রে বসিয়েছে।"

### GitHub Evidence
- [ ] Caption vs CLIP image retrieval eval

### LinkedIn Post
- [ ] —

---

<a id="table-retrieval"></a>

## 2. Table Retrieval

### What is it?
অনেক table এর মধ্যে প্রশ্নের জন্য সঠিক table, আর সম্ভব হলে সঠিক row/cell খুঁজে আনা। [Tables parsing](document-parsing.md#tables) এর পরের ধাপ।

### Why does it exist?
Table এর পুরো markdown embed করলে vector অস্পষ্ট হয় (সংখ্যার স্তূপ)। প্রশ্ন সাধারণত একটা নির্দিষ্ট cell চায়।

### What problem does it solve?
Spec, price, comparison, financial প্রশ্নের নির্ভুল উত্তর।

### How does it work?
```text
Index:
  table-level: caption + header + LLM summary → embed     ("কোন table?")
  row-level:   "Plan: Pro | Price: $20 | Storage: 100GB" → embed + BM25   ("কোন row?")
Query:
  retrieve table summary + matching rows → context এ পূর্ণ table (বা relevant rows + header)
Alternative:
  table → SQLite/DuckDB → text-to-SQL (বড়/numeric table এ)
```

### Important concepts
- **Multi-granularity index** (table + row)।
- **Header always**: row দিলে header ছাড়া অর্থহীন।
- **Numeric query** ("৫০ টাকার কম কোন plan?") vector search এ দুর্বল, SQL ভালো।
- **Units & footnotes** table এর সাথে রাখো।

### Trade-offs
Row-level index = precise কিন্তু index বড়। SQL path = exact, কিন্তু schema inference ও security কাজ।

### Alternatives
VLM কে table এর পাতার ছবি দিয়ে সরাসরি প্রশ্ন (document vision)।

### When should I use it?
Table-heavy corpus (pricing, catalog, report)।

### When should I NOT use it?
—(table থাকলে বিশেষ handling প্রায় সবসময় লাভজনক)

### Hands-on experiment
৫টা table (৩০০ row মোট) নিয়ে ১৫টা প্রশ্ন: (১) পুরো table chunk embed, (২) table+row multi-granularity, (৩) DuckDB + text-to-SQL। Accuracy table।

### Production implementation
Ingestion এ table detect হলে দুই path: vector index (summary+rows) আর structured store (DuckDB/Postgres)। Router প্রশ্নের ধরন দেখে বাছে।

### Failure cases
Row embed এ header ছিল না ("Pro | 20 | 100"), তাই retrieval অর্থহীন।

### Security concerns
Table এর sensitive column (salary) row-level ACL বা redact।

### Performance concerns
Row-level index অনেক vector, তবু দ্রুত।

### Cost concerns
Table summary তে LLM call (প্রতি table একবার)।

### What did I learn?
Table প্রশ্নে "কোন table" আর "কোন ঘর" দুটো আলাদা search সমস্যা।

### What can I explain to someone else?
> "বড় একটা রিপোর্টে আগে সঠিক টেবিলটা খুঁজে বের করো (সূচিপত্র দেখে), তারপর টেবিলে আঙুল দিয়ে সঠিক সারি।"

### GitHub Evidence
- [ ] Table retrieval: 3 approaches compared

### LinkedIn Post
- [ ] —

---

<a id="chart-understanding"></a>

## 3. Chart Understanding

### What is it?
Retrieval এর পরে, **answer time এ** chart image VLM কে দিয়ে প্রশ্নের উত্তর বের করা। (Parsing time এ chart-to-table: [document-parsing.md](document-parsing.md#charts))

### Why does it exist?
Indexing এর সময়ের description সব প্রশ্ন আগে থেকে জানে না। User এর নির্দিষ্ট প্রশ্নে ("মার্চ আর জুনের পার্থক্য কত?") আসল chart দেখা ভালো।

### What problem does it solve?
Chart-ভিত্তিক নির্দিষ্ট, অপ্রত্যাশিত প্রশ্নের উত্তর।

### How does it work?
```text
query → retrieve chunk (chart description/caption match) → chart image reference
     → VLM( [chart image] + surrounding text + question ) → answer (+ "approximate" disclaimer)
```

### Important concepts
- **Index text, answer with image**: retrieval এর জন্য description, উত্তরের জন্য আসল image।
- **Resolution**: chart image যথেষ্ট বড় রাখো (ছোট label পড়তে)।
- **Grounding**: VLM কে বলো কোন axis/value দেখে উত্তর দিচ্ছে বলতে।

### Trade-offs
নির্ভুলতর, কিন্তু প্রতি query তে image token (দামি) আর latency।

### Alternatives
শুধু indexing-time chart-to-table data দিয়ে উত্তর (সস্তা, কিন্তু সীমিত)।

### When should I use it?
Analytics report Q&A, financial document।

### When should I NOT use it?
High-volume, latency-sensitive path। Pre-extracted data ব্যবহার করো।

### Hands-on experiment
একটা report এর ৫টা chart নিয়ে ১০টা প্রশ্ন: (১) শুধু pre-extracted table, (২) answer time এ chart image VLM কে। Accuracy আর token cost তুলনা করো।

### Production implementation
Context এ max image সংখ্যা (যেমন ২), image resize policy, cost tracking per request।

### Failure cases
Retrieval ভুল chart এনেছে (একই রকম caption), VLM আত্মবিশ্বাসের সাথে ভুল chart থেকে উত্তর দিয়েছে।

### Security concerns
Chart image এর ভেতরে embedded text injection। Untrusted data হিসেবে treat করো।

### Performance concerns
Image input TTFT বাড়ায়।

### Cost concerns
প্রতিটা image শত থেকে হাজার token। Image সংখ্যা সীমিত রাখো।

### What did I learn?
Retrieve করো text দিয়ে, উত্তর দাও ছবি দেখে, এটাই multimodal RAG এর সাধারণ pattern।

### What can I explain to someone else?
> "Library catalogue এ লেখা দেখে বই খুঁজে বের করো, কিন্তু প্রশ্নের উত্তর দেওয়ার সময় বইয়ের ছবিটা নিজের চোখে দেখো।"

### GitHub Evidence
- [ ] Pre-extracted vs live chart VLM QA

### LinkedIn Post
- [ ] —

---

<a id="document-vision"></a>

## 4. Document Vision

### What is it?
Document parse না করে **পুরো পাতার screenshot/image** সরাসরি index ও retrieve করা, তারপর retrieved পাতার image VLM কে দিয়ে উত্তর। মূল উদাহরণ **ColPali** (2024): VLM-ভিত্তিক late-interaction (ColBERT-style) retriever যা পাতার image patch গুলোর multi-vector embedding বানায়। আরও আছে ColQwen, আর কিছু provider এর multimodal page embedding।

### Why does it exist?
Parsing pipeline (OCR → layout → table → chunk) জটিল ও ভঙ্গুর, প্রতিটা ধাপে তথ্য হারায়। VLM এখন পাতা "দেখে" বুঝতে পারে।

### What problem does it solve?
Visually rich document (slide, infographic, form, complex PDF) এ parsing ছাড়াই ভালো retrieval।

### How does it work?
```text
Index:  PDF page → image → ColPali → ~১০০০ patch vector per page → multi-vector index
Query:  text → token vectors → MaxSim (প্রতিটা query token এর সাথে সেরা patch) → page score
Answer: top-3 page images + question → VLM → answer (page number citation)
```

### Important concepts
- **Multi-vector** storage: প্রতি পাতায় অনেক vector, তাই storage বেশি (quantization/pooling দিয়ে কমানো)।
- **Late interaction (MaxSim)**।
- **Page-level citation** স্বাভাবিক।
- **Hybrid**: text-based RAG + page-image retrieval একসাথে।

### Trade-offs
Parsing ঝামেলা নেই আর visual element ভালো ধরে, কিন্তু storage বড়, multi-vector support লাগে (Qdrant, Vespa ইত্যাদি), আর answer time এ image token খরচ বেশি।

### Alternatives
Parse-then-text-RAG (সস্তা answer), VLM-to-markdown parsing।

### When should I use it?
Slide deck, brochure, scanned form, infographic-heavy report।

### When should I NOT use it?
Plain text-heavy document (parsing সস্তা ও যথেষ্ট), খুব বড় corpus যেখানে multi-vector storage খরচ বেশি।

### Hands-on experiment
Python এ `colpali-engine` দিয়ে একটা ২০-পাতার slide deck index করো (RAM সীমিত হলে ছোট deck বা cloud notebook)। Docling + text RAG এর সাথে ১০টা প্রশ্নে page-level Recall@3 তুলনা করো।

### Production implementation
Multi-vector-capable DB, page image storage, top-k page কম রাখো (৩), image resize।

### Failure cases
লম্বা dense text page এ (legal contract) patch-based retrieval text RAG এর চেয়ে খারাপ।

### Security concerns
Page image এ সব কিছু থাকে (signature, PII), তাই VLM provider এ পাঠানোর policy।

### Performance concerns
Indexing GPU-heavy, query time এ MaxSim হিসাব।

### Cost concerns
Storage (multi-vector) + answer time image token।

### What did I learn?
কখনো কখনো document "পড়ার" চেয়ে "দেখা" সহজ ও ভালো।

### What can I explain to someone else?
> "Slide এর লেখা আলাদা করে টাইপ না করে পুরো slide এর ছবি তুলে রাখা, আর খোঁজার সময় ছবির কোন অংশ প্রশ্নের সাথে মেলে সেটা দেখা।"

### GitHub Evidence
- [ ] ColPali vs parse-then-RAG on slide deck

### LinkedIn Post
- [ ] "Parsing ছাড়াই RAG: ColPali আর document vision"

---

<a id="multimodal-embeddings"></a>

## 5. Multimodal Embeddings

### What is it?
এমন embedding model যা ভিন্ন modality (text, image, কখনো audio/video) কে **একই vector space** এ বসায়, যাতে "একটা কুকুরের ছবি" text আর কুকুরের ছবি কাছাকাছি vector পায়। উদাহরণ: CLIP (OpenAI), SigLIP (Google), Jina CLIP, আর provider API এর multimodal embedding (যেমন Voyage multimodal, Cohere Embed এর image support, Google multimodal embedding)।

### Why does it exist?
আলাদা modality কে compare করতে একটা সাধারণ "ভাষা" দরকার।

### What problem does it solve?
Cross-modal search (text → image, image → text, image → image) caption ছাড়াই।

### How does it work?
```text
Contrastive training: লক্ষ লক্ষ (image, caption) জোড়া
  → ম্যাচিং জোড়ার vector কাছে টানো, অমিল জোড়া দূরে ঠেলো
Result: text encoder + image encoder, একই space এ output
```

### Important concepts
- **Same model for both sides**: index (image) আর query (text) একই model এর encoder।
- **Interleaved embedding**: কিছু নতুন model text+image মিশ্র input (যেমন screenshot + caption) একটা vector এ নেয়।
- **Text-in-image দুর্বলতা**: classic CLIP ছবির ভেতরের লেখা ভালো পড়ে না, নতুন model উন্নত।
- **Dimension ও normalize**, text embedding এর মতোই।

### Trade-offs
Caption pipeline ছাড়াই সরাসরি search, কিন্তু সূক্ষ্ম/domain-specific বোঝাপড়ায় caption+text embedding এর চেয়ে দুর্বল হতে পারে।

### Alternatives
Caption + text embedding, document vision (ColPali)।

### When should I use it?
বড় image collection, visual similarity search, e-commerce, media।

### When should I NOT use it?
Diagram/screenshot যেখানে ভেতরের text-ই আসল তথ্য। Caption/OCR path ভালো।

### Hands-on experiment
```ts
// npm i @huggingface/transformers   (CLIP Node এ চলে, local)
import { AutoProcessor, CLIPVisionModelWithProjection, AutoTokenizer, CLIPTextModelWithProjection, RawImage } from "@huggingface/transformers";
const id = "Xenova/clip-vit-base-patch32";
const [proc, vision, tok, text] = await Promise.all([
  AutoProcessor.from_pretrained(id), CLIPVisionModelWithProjection.from_pretrained(id),
  AutoTokenizer.from_pretrained(id), CLIPTextModelWithProjection.from_pretrained(id),
]);
const img = await vision(await proc(await RawImage.read("dog.jpg")));
const txt = await text(tok(["a photo of a dog", "a photo of a car"], { padding: true }));
// img.image_embeds আর txt.text_embeds এর cosine তুলনা করো
```
১০টা ছবি আর ১০টা বাংলা/English query দিয়ে ম্যাচিং দেখো (classic CLIP বাংলা বোঝে কিনা লক্ষ করো)।

### Production implementation
Named vectors: `text_vec` (caption) + `image_vec` (CLIP), hybrid fusion। Model version metadata।

### Failure cases
বাংলা query তে English-only CLIP অর্থহীন result দেয়। Multilingual model দরকার।

### Security concerns
Image similarity search দিয়ে মানুষের ছবি খোঁজা (face search) = privacy ঝুঁকি ও আইনি সমস্যা। Use case সীমিত রাখো।

### Performance concerns
CLIP ছোট ও দ্রুত, CPU তেও চলে।

### Cost concerns
Local model free। API multimodal embedding per-image।

### What did I learn?
Multimodal embedding = ছবি আর কথার জন্য এক অভিধান।

### What can I explain to someone else?
> "একটা অভিধান যেখানে 'কুকুর' শব্দ আর কুকুরের ছবি একই পাতায় থাকে। শব্দ দিয়ে খুঁজলে ছবি পাও, ছবি দিয়ে খুঁজলে শব্দ।"

### GitHub Evidence
- [ ] Local CLIP text→image search demo

### LinkedIn Post
- [ ] "ছবি আর লেখা একই ভাষায়: multimodal embeddings"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

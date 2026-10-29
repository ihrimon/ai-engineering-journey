# Phase 4 — RAG & Knowledge Systems: Document Parsing (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Parsing tool গুলো (Unstructured, LlamaParse, Docling) দ্রুত বদলায়। Feature/দাম official docs থেকে যাচাই করবে। সবচেয়ে ভালো tool হলো **তোমার নিজের document এ** যেটা ভালো করে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Unstructured](#unstructured) | Open-source + API: অনেক file type → typed "elements" (Title, NarrativeText, Table…) |
| [LlamaParse](#llamaparse) | LlamaIndex এর hosted, LLM/VLM-ভিত্তিক parser: জটিল PDF → clean markdown |
| [Docling](#docling) | IBM এর open-source local parser: layout + table structure model, markdown/JSON |
| [PDF extraction](#pdf-extraction) | PDF থেকে text/structure বের করা: digital বনাম scanned আলাদা সমস্যা |
| [OCR](#ocr) | Image/scanned page থেকে text চেনা (Tesseract, cloud OCR, VLM) |
| [Tables](#tables) | Row/column structure অক্ষত রেখে table বের করা ও represent করা |
| [Images](#images) | Document এর ভেতরের ছবি: caption/description বানিয়ে searchable করা |
| [Charts](#charts) | Chart থেকে data/insight বের করা (VLM বা chart-to-table) |
| [Layout-aware parsing](#layout-aware-parsing) | পাতার visual layout (column, heading, header/footer, reading order) বুঝে parse |

```text
                       "Garbage in, garbage out" এর আসল জায়গা
PDF/DOCX/HTML/scan ─► [ Parse ] ─► clean structured text (markdown + tables + captions)
                         │              │
                 layout · OCR · tables · images · charts
                         ▼
                   Chunk ─► Embed ─► Index
RAG এর অনেক "model ভুল উত্তর দিচ্ছে" সমস্যার মূল কারণ আসলে খারাপ parsing।
```

---

<a id="unstructured"></a>

## 1. Unstructured

### What is it?
`unstructured` হলো open-source Python library + hosted API/platform, যেটা PDF, DOCX, PPTX, HTML, email, image সহ অনেক format কে typed **elements** এর list এ ভাঙে: `Title`, `NarrativeText`, `ListItem`, `Table`, `Image`, `Header/Footer`, metadata (page, coordinates) সহ।

### Why does it exist?
প্রতিটা file format এর জন্য আলাদা parser লেখা ঝামেলা। একটা unified interface দরকার।

### What problem does it solve?
বিভিন্ন format কে একই structure এ আনা, আর element type দিয়ে chunking (title ধরে section ভাগ)।

### How does it work?
```text
partition(file) → strategy: "fast" (text layer) | "hi_res" (layout model + OCR) | "ocr_only"
→ [Title("Refund Policy"), NarrativeText("..."), Table(html=...), ...]
→ chunk_by_title()
```

### Important concepts
- **Strategy**: fast (সস্তা, digital PDF) বনাম hi_res (ধীর, জটিল layout/scan)।
- **Element metadata**: page_number, coordinates, `text_as_html` (table)।
- **Chunking helpers** (`by_title`)।

### Trade-offs
অনেক format একসাথে, কিন্তু hi_res ধীর ও dependency-heavy (Python, system libraries)।

### Alternatives
Docling (local), LlamaParse (hosted), format-specific library (pdfplumber, mammoth)।

### When should I use it?
Mixed file type এর ingestion pipeline।

### When should I NOT use it?
শুধু clean markdown/HTML হলে, তখন সহজ parser যথেষ্ট।

### Hands-on experiment
```bash
pip install "unstructured[pdf]"
python -c "from unstructured.partition.auto import partition; [print(type(e).__name__, '|', str(e)[:80]) for e in partition('sample.pdf')]"
```
একই PDF `fast` আর `hi_res` strategy তে চালিয়ে element তুলনা করো।

### Production implementation
Python microservice (FastAPI) হিসেবে, TypeScript app থেকে HTTP call। Queue-based async ingestion।

### Failure cases
Multi-column PDF fast strategy তে দুই column মিশে বাক্য এলোমেলো।

### Security concerns
Untrusted file parse = attack surface (malicious PDF)। Sandbox/container এ চালাও, file size limit দাও।

### Performance concerns
hi_res এ প্রতি পাতা সেকেন্ড লাগতে পারে।

### Cost concerns
Open-source free (compute খরচ), hosted API তে per-page।

### What did I learn?
Parser একটা ETL ধাপ, এটাকে আলাদা service হিসেবে ভাবো।

### What can I explain to someone else?
> "সব রকম কাগজ (চিঠি, রসিদ, বই) একটা মেশিনে ঢোকাও, সেটা সব কিছু একই ধরনের সাজানো কার্ডে রূপান্তর করে: শিরোনাম কার্ড, অনুচ্ছেদ কার্ড, টেবিল কার্ড।"

### GitHub Evidence
- [ ] Unstructured fast vs hi_res comparison

### LinkedIn Post
- [ ] —

---

<a id="llamaparse"></a>

## 2. LlamaParse

### What is it?
LlamaIndex (LlamaCloud) এর hosted document parsing service, যেটা LLM/VLM ব্যবহার করে জটিল document (table, chart, multi-column, scan) কে clean markdown/JSON এ রূপান্তর করে। Parsing instruction (prompt) দেওয়া যায়।

### Why does it exist?
Rule-based parser জটিল financial report, scientific paper, slide এ ব্যর্থ হয়। Vision-language model পাতা "দেখে" বুঝতে পারে।

### What problem does it solve?
জটিল layout ও table থেকে উচ্চমানের markdown, কম engineering এ।

### How does it work?
```text
upload file → mode (fast / balanced / premium-ধরনের tier) + optional parsing instruction
→ job → markdown / JSON (pages, tables, images)
```

### Important concepts
- **Tier/mode অনুযায়ী দাম ও quality** (credit per page)।
- **Parsing instructions**: "Output tables as markdown, ignore page footers"।
- TypeScript SDK/REST আছে।

### Trade-offs
উচ্চ quality, কম কাজ, কিন্তু hosted (data বাইরে যায়), per-page খরচ, আর LLM-based তাই মাঝে মাঝে hallucinated text (বিশেষ করে অস্পষ্ট scan এ)।

### Alternatives
Docling (local), Unstructured, সরাসরি VLM (Claude/GPT/Gemini) কে PDF পাতা দিয়ে markdown বানানো।

### When should I use it?
জটিল PDF, অল্প engineering সময়, data hosted parser এ পাঠানো যায়।

### When should I NOT use it?
Strict data residency, বিশাল volume (খরচ), বা পুরো determinism দরকার।

### Hands-on experiment
একটা জটিল table-ভারী PDF (annual report এর ২–৩ পাতা) LlamaParse free credit এ চালাও, আর একই পাতা Docling এ। Table accuracy হাতে মিলিয়ে দেখো।

### Production implementation
Async job + webhook/polling, result cache (file hash দিয়ে, একই file আবার parse করবে না)।

### Failure cases
অস্পষ্ট scan এ VLM একটা সংখ্যা "অনুমান" করে বসিয়েছে, ফলে financial data ভুল।

### Security concerns
Document third party তে যায়। Confidential doc এর জন্য policy/agreement দেখো।

### Performance concerns
Async, পাতা প্রতি কয়েক সেকেন্ড, বড় file এ মিনিট।

### Cost concerns
Per-page credit। File hash cache দিয়ে duplicate parse এড়াও।

### What did I learn?
VLM-based parsing শক্তিশালী, কিন্তু "পড়া" আর "অনুমান করা" এর সীমানা ঝাপসা। Number গুলো validate করো।

### What can I explain to someone else?
> "রোবট দিয়ে স্ক্যান না করে একজন মানুষকে দিয়ে পাতাটা দেখে টাইপ করানো। বেশি বুদ্ধিমান, কিন্তু মানুষের মতো মাঝে মাঝে ভুল পড়ে।"

### GitHub Evidence
- [ ] LlamaParse vs Docling table accuracy

### LinkedIn Post
- [ ] —

---

<a id="docling"></a>

## 3. Docling

### What is it?
IBM Research এর open-source (MIT) document conversion toolkit। Local এ চলে, layout analysis model আর table structure model (TableFormer) দিয়ে PDF, DOCX, PPTX, HTML, image কে structured `DoclingDocument` → markdown/JSON/HTML এ রূপান্তর করে। Vision-language model (যেমন SmolDocling/granite-docling) pipeline ও আছে।

### Why does it exist?
উচ্চমানের parsing local এ, data বাইরে না পাঠিয়ে, আর free।

### What problem does it solve?
Private document এর জন্য production-grade layout + table extraction।

### How does it work?
```text
PDF → page images + text layer → layout model (heading, paragraph, table, figure, reading order)
    → TableFormer (row/col structure) → optional OCR (scanned) → DoclingDocument → export_to_markdown()
```

### Important concepts
- **Local, GPU optional** (CPU তে ধীর কিন্তু চলে)।
- **Chunker** built-in (HybridChunker, heading-aware)।
- LangChain/LlamaIndex integration।
- OCR engine plug-in (EasyOCR, Tesseract ইত্যাদি)।

### Trade-offs
Free ও private, কিন্তু CPU তে ধীর, model download লাগে, আর মাঝে মাঝে জটিল layout এ hosted VLM এর চেয়ে দুর্বল।

### Alternatives
Marker, MinerU (open), Unstructured, LlamaParse।

### When should I use it?
Self-hosted pipeline, confidential document, ভালো table extraction দরকার।

### When should I NOT use it?
খুব কম resource (৮GB RAM এ বড় batch কষ্টকর হতে পারে), বা JS-only stack যেখানে Python service চালাতে চাও না।

### Hands-on experiment
```bash
pip install docling
docling sample.pdf --to md --output out/
```
Output markdown খুলে table, heading, reading order যাচাই করো। একটা page এ কত সময় লাগছে মাপো।

### Production implementation
Python worker (queue থেকে job), GPU node বড় volume এ, output markdown + JSON দুটোই store।

### Failure cases
Scanned বাংলা PDF এ default OCR ভাষা English, তাই আবর্জনা text। OCR language config করো।

### Security concerns
Local, তাই data বাইরে যায় না। তবু untrusted file sandbox এ parse।

### Performance concerns
CPU তে প্রতি পাতা কয়েক সেকেন্ড। GPU তে অনেক দ্রুত।

### Cost concerns
Software free, শুধু compute।

### What did I learn?
Local parsing এখন hosted এর কাছাকাছি মানের, privacy-sensitive project এ প্রথম পছন্দ।

### What can I explain to someone else?
> "নিজের বাসায় একটা ভালো স্ক্যানার আর টাইপিস্ট রাখা, কাগজ বাইরে পাঠাতে হয় না।"

### GitHub Evidence
- [ ] Docling local conversion + timing

### LinkedIn Post
- [ ] "Private document এর জন্য local parsing: Docling"

---

<a id="pdf-extraction"></a>

## 4. PDF Extraction

### What is it?
PDF থেকে text আর structure বের করা। দুই ধরনের PDF: **digital/born-digital** (ভেতরে text layer আছে) আর **scanned/image** (শুধু ছবি, OCR লাগে)। অনেক PDF মিশ্র।

### Why does it exist?
PDF হলো "print format": পাতায় কোন glyph কোথায় বসবে, সেটা বলে, "এটা heading", "এটা table" না। Reading order, paragraph, table, সব পুনর্গঠন করতে হয়।

### What problem does it solve?
Enterprise knowledge এর বিশাল অংশ PDF এ, তাই RAG এর প্রবেশদ্বার।

### How does it work?
```text
1. Detect: text layer আছে? (ছোট sample এ extracted text length দেখো)
2. Digital → text extraction (pdf.js/pdfplumber/PyMuPDF) + layout heuristics
3. Scanned → render page image → OCR
4. Clean: header/footer/page number বাদ, hyphenation জোড়া, ligature ঠিক
```

### Important concepts
- **Reading order** (multi-column, sidebar)।
- **Header/footer repetition** প্রতিটা chunk কে দূষিত করে, বাদ দাও।
- **Unicode issues**: বাংলা PDF এ font encoding এর কারণে text layer থাকলেও ভাঙা character (legacy Bijoy font ইত্যাদি) আসতে পারে।
- **Page number metadata** citation এর জন্য রাখো।

### Trade-offs
Text layer দ্রুত ও নির্ভুল (যদি ঠিক থাকে)। OCR/VLM সব ক্ষেত্রে কাজ করে কিন্তু ধীর ও ভুলের সম্ভাবনা।

### Alternatives
Source format (DOCX/HTML) পাওয়া গেলে PDF এড়াও। LLM কে সরাসরি PDF দেওয়া (Claude/Gemini PDF input), যা ছোট volume এ সহজ।

### When should I use it?
PDF corpus এ সবসময়।

### When should I NOT use it?
Original source (Google Doc, Confluence, HTML) পাওয়া গেলে সেখান থেকে নাও।

### Hands-on experiment
```ts
// npm i unpdf   (pdf.js ভিত্তিক, Node এ চলে)
import { extractText, getDocumentProxy } from "unpdf";
import { readFile } from "node:fs/promises";
const pdf = await getDocumentProxy(new Uint8Array(await readFile("sample.pdf")));
const { text } = await extractText(pdf, { mergePages: false });
text.forEach((t, i) => console.log(`--- page ${i + 1} (${t.length} chars)`, t.slice(0, 200)));
```
একটা English আর একটা বাংলা PDF এ চালাও। বাংলায় text ঠিকঠাক আসে নাকি ভাঙা? Page প্রতি char সংখ্যা খুব কম হলে সেটা scanned page।

### Production implementation
- Per-page routing: text layer ভালো হলে fast path, না হলে OCR/VLM path।
- Quality check: extracted text এ অদ্ভুত character এর ratio বেশি হলে fallback।

### Failure cases
- Header "Confidential – Acme Corp – Page 3" প্রতিটা chunk এ, তাই সব chunk এর embedding এ noise।
- বাংলা PDF এ ভাঙা যুক্তাক্ষর, ফলে retrieval শূন্য।

### Security concerns
PDF এ hidden text layer (সাদা রঙে লেখা injection) থাকতে পারে, যা চোখে দেখা যায় না কিন্তু extracted text এ আসে।

### Performance concerns
Text layer: ms/page। OCR: সেকেন্ড/page।

### Cost concerns
Fast path সস্তা, শুধু দরকারি পাতায় দামি path ব্যবহার করো।

### What did I learn?
"PDF থেকে text" সহজ শোনায়, কিন্তু PDF আসলে text format না।

### What can I explain to someone else?
> "PDF হলো ছাপানো পোস্টার: অক্ষর কোথায় বসবে জানে, কিন্তু কোনটা শিরোনাম কোনটা অনুচ্ছেদ জানে না। আমাদের আবার সেই গঠন খুঁজে বের করতে হয়।"

### GitHub Evidence
- [ ] Per-page digital/scanned router

### LinkedIn Post
- [ ] "বাংলা PDF থেকে text বের করা কেন এত কঠিন"

---

<a id="ocr"></a>

## 5. OCR

### What is it?
**Optical Character Recognition**: image (scan, photo, screenshot) থেকে text চেনা। Classic engine (Tesseract), cloud OCR (Google Document AI, AWS Textract, Azure Document Intelligence), আর এখন VLM-based OCR (multimodal LLM সরাসরি পাতা পড়ে)।

### Why does it exist?
Scanned document, photo করা রসিদ, পুরনো archive এ কোনো text layer নেই।

### What problem does it solve?
Image-only content কে searchable/RAG-যোগ্য text এ রূপান্তর।

### How does it work?
```text
image → preprocess (deskew, denoise, binarize, DPI 300) → text detection (কোথায় লেখা)
      → recognition (কী লেখা) → confidence score + bounding box
VLM OCR: page image → "Transcribe this page as markdown" → text (layout সহ)
```

### Important concepts
- **Language pack**: বাংলার জন্য `ben` (Tesseract), বা multilingual cloud OCR।
- **Confidence score**: কম confidence অংশ flag করো।
- **Handwriting** classic OCR এ দুর্বল, VLM/cloud ভালো।
- **VLM hallucination**: অস্পষ্ট অংশে বানানো text, যেটা classic OCR এ সাধারণত হয় না (সেখানে ভুল character হয়)।

### Trade-offs
Tesseract = free, local, কিন্তু জটিল layout ও বাংলায় দুর্বল। Cloud = ভালো, কিন্তু খরচ ও privacy। VLM = সবচেয়ে বুদ্ধিমান, কিন্তু hallucination ও খরচ।

### Alternatives
Source digital file খুঁজে পাওয়া (সবচেয়ে ভালো), manual data entry (ছোট volume)।

### When should I use it?
Scanned PDF, image document, receipt, ID card, handwritten form।

### When should I NOT use it?
Digital PDF এর text layer ঠিক থাকলে। OCR অপ্রয়োজনীয় আর ভুল যোগ করে।

### Hands-on experiment
```ts
// npm i tesseract.js
import Tesseract from "tesseract.js";
const { data } = await Tesseract.recognize("bangla-scan.png", "ben+eng");
console.log(data.confidence, data.text.slice(0, 300));
```
একই image Tesseract আর একটা VLM (যেমন Claude বা local `qwen2.5vl`) দিয়ে পড়াও। Character error হাতে গুনে তুলনা করো।

### Production implementation
Preprocessing pipeline, confidence threshold, কম confidence পাতা human review queue তে।

### Failure cases
Low DPI (৭২) scan এ বাংলা OCR প্রায় অপাঠ্য।

### Security concerns
ID card/medical scan = sensitive PII। Cloud OCR এ পাঠানোর আগে policy দেখো, output redact।

### Performance concerns
Tesseract CPU তে প্রতি পাতা ১–৫s। VLM আরও ধীর।

### Cost concerns
Cloud OCR per-page, VLM per-token (image token)। Volume বেশি হলে local।

### What did I learn?
OCR quality শুরু হয় scan quality থেকে। খারাপ ছবি থেকে ভালো text আশা করা যায় না।

### What can I explain to someone else?
> "ঝাপসা ছবির লেখা পড়ার চেষ্টা: ভালো চোখ (model) সাহায্য করে, কিন্তু পরিষ্কার ছবি (300 DPI scan) সবচেয়ে বেশি সাহায্য করে।"

### GitHub Evidence
- [ ] Tesseract vs VLM on Bangla scan

### LinkedIn Post
- [ ] "বাংলা OCR: Tesseract বনাম AI vision model"

---

<a id="tables"></a>

## 6. Tables

### What is it?
Document এর table কে row/column structure অক্ষত রেখে বের করা, আর RAG এর জন্য উপযুক্তভাবে represent করা (markdown, HTML, JSON, বা row-wise text)।

### Why does it exist?
Plain text extraction এ table ভেঙে সংখ্যার স্রোত হয়ে যায়: কোন সংখ্যা কোন column এর বোঝা যায় না। অথচ গুরুত্বপূর্ণ তথ্য (দাম, তুলনা, specs) প্রায়ই table এ।

### What problem does it solve?
"Pro plan এর মাসিক দাম কত?" এর মতো প্রশ্নের নির্ভুল উত্তর।

### How does it work?
```text
Detect table region → structure recognition (rows, cols, merged cells, header)
→ represent:
   Markdown table  (ছোট-মাঝারি table, LLM ভালো পড়ে)
   HTML table      (merged cell/complex header)
   Row-as-sentence ("Plan: Pro | Monthly: $20 | Storage: 100GB") — retrieval এর জন্য ভালো
   Table summary   (LLM: "এই table টা plan অনুযায়ী দাম দেখায়") — embedding এর জন্য
```

### Important concepts
- **Header repeat**: বড় table ভাগ করলে প্রতিটা chunk এ header রাখো।
- **Dual representation**: embed করো summary/row-text, context এ দাও পূর্ণ table।
- **Numeric precision**: সংখ্যা validate (মোট যোগফল মেলে কিনা)।
- **Spreadsheet** (Excel/CSV) হলে structured retrieval (SQL) প্রায়ই ভালো।

### Trade-offs
সঠিক table extraction কঠিন ও ধীর (model লাগে)। Simple extraction দ্রুত কিন্তু ভাঙা।

### Alternatives
Table কে DB তে load করে SQL query, VLM কে পাতার ছবি দিয়ে সরাসরি প্রশ্ন।

### When should I use it?
Financial report, pricing, spec sheet, research paper।

### When should I NOT use it?
—(table থাকলে সবসময় আলাদা যত্ন লাগে)

### Hands-on experiment
একটা pricing table এর PDF নাও। (১) plain text extraction, (২) Docling markdown table, (৩) row-as-sentence। প্রতিটা দিয়ে ১০টা নির্দিষ্ট cell-প্রশ্নের accuracy মাপো।

### Production implementation
```ts
const rowSentences = (headers: string[], rows: string[][]) =>
  rows.map((r) => headers.map((h, i) => `${h}: ${r[i]}`).join(" | "));
// embed: rowSentences + table caption;  context: পুরো markdown table
```

### Failure cases
- Merged header cell ভুল parse হয়ে সব column এক ঘর সরে গেছে, ফলে প্রতিটা দাম ভুল plan এর।
- ১০০ row table এক chunk এ, তাই embedding অর্থহীন।

### Security concerns
Table এ PII column (phone, email) থাকলে ingestion এ redact।

### Performance concerns
Table structure model ধীর, শুধু table page এ চালাও।

### Cost concerns
VLM দিয়ে table parse দামি, deterministic আগে চেষ্টা করো।

### What did I learn?
Table = ছোট database। Text হিসেবে না, structure হিসেবে treat করো।

### What can I explain to someone else?
> "Excel sheet কে লম্বা একটা বাক্যে লিখে ফেললে কোন সংখ্যা কোন কলামের বোঝা যায় না। Table এর ঘরগুলো ঘর হিসেবেই রাখতে হবে।"

### GitHub Evidence
- [ ] Table representation accuracy test

### LinkedIn Post
- [ ] "RAG এ table: সবচেয়ে বেশি ভুল হয় যেখানে"

---

<a id="images"></a>

## 7. Images

### What is it?
Document এর ভেতরের ছবি (diagram, screenshot, photo, logo) handle করা: extract করা, VLM দিয়ে **caption/description** বানানো, আর সেটা text হিসেবে index করা (বা multimodal embedding)।

### Why does it exist?
অনেক তথ্য শুধু ছবিতে: architecture diagram, UI screenshot এ step, product photo। Text-only parsing এ এগুলো পুরো হারিয়ে যায়।

### What problem does it solve?
"Settings page এ export button কোথায়?" এর মতো প্রশ্ন, যার উত্তর একটা screenshot এ।

### How does it work?
```text
extract image (+ page, position, nearby caption/text)
→ filter (logo, decorative icon বাদ)
→ VLM: "Describe this image for search. Include all visible text, labels, and what it shows."
→ description text → chunk (surrounding text সহ) → embed
→ answer time এ: description + (দরকার হলে) আসল image VLM কে দাও
```

### Important concepts
- **Decorative filter**: ছোট/repeated image বাদ।
- **Context**: আশেপাশের text ও caption যোগ করলে description অনেক ভালো।
- **Store reference**: image file path রাখো, যাতে উত্তরে দেখানো বা VLM এ পাঠানো যায়।
- Multimodal embedding বিকল্প (দেখো [multimodal-rag.md](multimodal-rag.md))।

### Trade-offs
Searchable ছবি, কিন্তু প্রতিটা ছবিতে VLM call (খরচ, সময়), আর description এ ভুল।

### Alternatives
Multimodal embedding (CLIP/ColPali ধরনের), image কে শুধু caption দিয়ে index।

### When should I use it?
Technical docs, manual, product catalog, slide deck।

### When should I NOT use it?
Image গুলো decorative হলে।

### Hands-on experiment
একটা slide deck থেকে ৫টা diagram extract করো, local VLM (যেমন `qwen2.5vl:3b`, RAM অনুযায়ী) বা Claude দিয়ে description বানাও, তারপর ৫টা প্রশ্নে retrieval পরীক্ষা করো।

### Production implementation
Image hash dedup (একই logo ১০০ পাতায়), async captioning queue, description এ `source_image_id`।

### Failure cases
Diagram এর ছোট লেখা VLM পড়তে পারেনি (low resolution extract), ফলে description অসম্পূর্ণ।

### Security concerns
Image এর ভেতরে লেখা injection ("Ignore instructions...") VLM পড়ে description এ ঢুকিয়ে দিতে পারে। Description কে untrusted data ধরো।

### Performance concerns
VLM call প্রতি image সেকেন্ড, তাই offline/async।

### Cost concerns
Image token দামি। Filter আর dedup দিয়ে সংখ্যা কমাও।

### What did I learn?
ছবিকে searchable বানানো = ছবিকে ভালো text এ বর্ণনা করা।

### What can I explain to someone else?
> "অন্ধ পাঠকের জন্য বইয়ের প্রতিটা ছবির পাশে বর্ণনা লিখে দেওয়া। Search engine ও সেই বর্ণনা পড়েই ছবি খুঁজে পায়।"

### GitHub Evidence
- [ ] Diagram captioning → retrieval test

### LinkedIn Post
- [ ] —

---

<a id="charts"></a>

## 8. Charts

### What is it?
Document এর chart/graph (bar, line, pie) থেকে তথ্য বের করা: underlying data (chart-to-table), trend/insight description, আর axis/label। সাধারণত VLM দিয়ে।

### Why does it exist?
Report এর মূল insight প্রায়ই chart এ ("Q3 তে revenue ৩০% বেড়েছে"), যা text এ লেখা নাও থাকতে পারে।

### What problem does it solve?
Chart-এর উপর ভিত্তি করে প্রশ্নের উত্তর: "কোন quarter এ সবচেয়ে বেশি বিক্রি?"।

### How does it work?
```text
chart image → VLM prompt:
  "1. Chart type, title, axes, units.
   2. Extract the data as a markdown table (approximate values if not labeled — mark them as ~).
   3. Two-sentence summary of the main trend."
→ table + summary → index; image reference রাখো
```

### Important concepts
- **Labeled vs unlabeled values**: label না থাকলে পড়া মান **approximate**, সেটা চিহ্নিত করো।
- **Source data better**: chart এর পেছনের Excel/data পাওয়া গেলে সেটাই নাও।
- **Validation**: pie chart এর যোগফল ~১০০% কিনা।

### Trade-offs
দরকারি insight, কিন্তু VLM এর সংখ্যা অনুমান ভুল হতে পারে।

### Alternatives
Chart এর source data, specialized chart-to-table model।

### When should I use it?
Financial/analytics report, research paper, dashboard export।

### When should I NOT use it?
যেখানে exact সংখ্যা জরুরি আর label নেই। তখন source data খোঁজো, অনুমানের উপর সিদ্ধান্ত নেবে না।

### Hands-on experiment
৩টা chart নাও (একটা labeled bar, একটা unlabeled line, একটা pie)। VLM দিয়ে table বানিয়ে আসল data এর সাথে error % মাপো।

### Production implementation
Extracted value এ `approximate: true` flag, উত্তরে "chart থেকে আনুমানিক" disclaimer।

### Failure cases
Log scale axis না বুঝে VLM linear ধরে সংখ্যা পড়েছে, ফলে বিশাল ভুল।

### Security concerns
Misleading chart থেকে ভুল সিদ্ধান্ত। Approximate disclaimer দাও।

### Performance concerns
VLM call, তাই offline।

### Cost concerns
Image token।

### What did I learn?
Chart থেকে পড়া সংখ্যা "মাপা" না, "অনুমান"। সেটা স্পষ্ট করে বলতে হয়।

### What can I explain to someone else?
> "Graph দেখে বলা 'মোটামুটি ৪০ হবে'। কাজের জন্য ঠিক আছে, কিন্তু হিসাবের খাতায় বসানোর আগে আসল সংখ্যা খুঁজে নাও।"

### GitHub Evidence
- [ ] Chart-to-table error measurement

### LinkedIn Post
- [ ] —

---

<a id="layout-aware-parsing"></a>

## 9. Layout-Aware Parsing

### What is it?
পাতার **visual layout** বুঝে parse করা: কোনটা heading, paragraph, column, sidebar, header/footer, table, figure, caption, আর সঠিক **reading order**। Layout detection model (Docling, Unstructured hi_res, LayoutLM ধরনের) বা VLM দিয়ে।

### Why does it exist?
মানুষ পাতা দেখে বুঝে কোনটা আগে পড়তে হবে। Text layer শুধু অক্ষরের অবস্থান দেয়, অর্থপূর্ণ গঠন না।

### What problem does it solve?
Multi-column paper, newsletter, form, invoice, slide, যেখানে সরল top-to-bottom পড়া ভুল ক্রম দেয়।

### How does it work?
```text
page image → layout model → blocks [{type: "title", bbox}, {type: "text", bbox}, {type: "table", bbox}, ...]
→ reading order (column detect, top-bottom, left-right)
→ per-block: text extract / OCR / table model / figure caption
→ structured output (markdown with headings)
```

### Important concepts
- **Bounding box** + **block type**।
- **Reading order** algorithm।
- **Headers/footers** চিহ্নিত করে বাদ।
- **Heading hierarchy** recovery → structural chunking এর ভিত্তি।

### Trade-offs
সেরা structure, কিন্তু model inference লাগে, ধীর।

### Alternatives
Text-layer heuristics (font size দিয়ে heading অনুমান), VLM-to-markdown।

### When should I use it?
Complex PDF, scan, form, slide, যেখানে structure গুরুত্বপূর্ণ।

### When should I NOT use it?
Simple single-column digital PDF বা ইতিমধ্যে structured source (HTML/markdown)।

### Hands-on experiment
একটা দুই-column research paper এর পাতা নাও: (১) plain text extraction, (২) Docling layout-aware। দুটোর output পাশাপাশি রেখে reading order ঠিক আছে কিনা দেখো।

### Production implementation
Parser router: file type + quality signal অনুযায়ী simple বা layout-aware path, output schema একই রাখো।

### Failure cases
Sidebar এর "Related articles" main text এর মাঝে ঢুকে গেছে, ফলে chunk এ অপ্রাসঙ্গিক তথ্য।

### Security concerns
Hidden/off-page block (bbox পাতার বাইরে) ফেলে দাও, injection লুকানোর জায়গা।

### Performance concerns
GPU তে দ্রুত, CPU তে ধীর।

### Cost concerns
Compute, শুধু দরকারি document এ চালাও।

### What did I learn?
ভালো parsing = পাতাকে মানুষের চোখে দেখা।

### What can I explain to someone else?
> "পত্রিকা পড়ার সময় তুমি জানো কোন কলাম আগে, কোনটা বিজ্ঞাপন। Layout-aware parser কম্পিউটারকে সেই চোখ দেয়।"

### GitHub Evidence
- [ ] Plain vs layout-aware on 2-column paper

### LinkedIn Post
- [ ] "RAG এর গুণমান শুরু হয় parsing থেকে"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

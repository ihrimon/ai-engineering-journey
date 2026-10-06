# Phase 1 — LLM Fundamentals: Multimodal Inputs (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** কোন model কোন ধরনের input (image, audio, PDF) নেয়, file size ও page limit কত, image কত token খায়, এগুলো provider ভেদে আলাদা আর দ্রুত বদলায়। ব্যবহারের আগে provider এর docs দেখবে।
>
> 🧪 Hands-on এ local vision model (Ollama) আর Whisper ব্যবহার করা হয়েছে, তাই খরচ শূন্য। Vision model এর জন্য সাধারণত 8B text model এর চেয়ে বেশি RAM লাগে।

## Quick Cheat Sheet

| Topic | এক লাইনে |
|---|---|
| [Vision](#vision) | Model এর ছবি "দেখে" বোঝার ক্ষমতা: ছবিকে token এ ভেঙে text এর পাশে বসানো |
| [Images](#images) | বাস্তবে ছবি পাঠানো: format, size, resolution, token খরচ |
| [Audio](#audio) | কথা বোঝা: হয় আগে text এ রূপান্তর (STT), নয়তো সরাসরি audio model |
| [PDFs](#pdfs) | PDF আসলে text না, "ছাপানো পাতা"। Text বের করা, নাকি পাতার ছবি পাঠানো |
| [Documents](#documents) | DOCX, HTML, email ইত্যাদিকে model এর পড়ার মতো text (সাধারণত Markdown) এ রূপান্তর |
| [Tables](#tables) | ২-মাত্রার table কে ১-মাত্রার token এ ভাঙলে কাঠামো হারায়, তাই সাবধানে represent করতে হয় |
| [Charts](#charts) | Chart থেকে trend বোঝা সহজ, কিন্তু exact সংখ্যা পড়া অনির্ভরযোগ্য |

```text
                          সব input শেষে token হয়
                                   │
    ┌──────────┬───────────┬───────┴───────┬────────────┐
  Text       Image       Audio            PDF        Document
    │          │           │               │            │
Tokenizer  Image encoder  STT → text   Text layer +   Parser →
    │      (patch → token) বা audio     পাতার ছবি     Markdown
    │          │          encoder          │            │
    └──────────┴───────────┴───────┬───────┴────────────┘
                                   ▼
                     একই context window (একই বিল)
```

> **Core Concepts এর সাথে যোগ:** ছবি, audio, PDF সবই শেষে token হয়ে একই context window এ ঢোকে। তাই এগুলোও **context জায়গা খায়, cost বাড়ায়, latency বাড়ায়**। একটা বড় PDF পাঠানো মানে হাজার হাজার token।

---

<a id="vision"></a>

## 1. Vision

### What is it?
LLM এর ছবি বোঝার ক্ষমতা: ছবিতে কী আছে বর্ণনা করা, ছবির লেখা পড়া (OCR), ছবি নিয়ে প্রশ্নের উত্তর দেওয়া। এমন model কে বলে **vision-language model (VLM)**।

### Why does it exist?
বাস্তব দুনিয়ার অনেক তথ্য text এ না: receipt, screenshot, form, ID card, diagram, পণ্যের ছবি।

### What problem does it solve?
আলাদা OCR, আলাদা image classifier ছাড়াই একটা model দিয়ে ছবি বোঝা আর তা নিয়ে যুক্তি করা।

### How does it work?
```text
ছবি ──► ছোট ছোট টুকরো (patch) ──► Image encoder ──► "image token" ──┐
                                                                     ├──► LLM
Text ──► Tokenizer ────────────────────────────────► text token ────┘
```
- ছবিকে ছোট ছোট বর্গে (patch) ভাগ করা হয়, যেমন text কে token এ ভাগ করা হয়।
- একটা image encoder প্রতিটা patch কে LLM বোঝে এমন সংখ্যায় (embedding) রূপান্তর করে।
- তারপর LLM এর কাছে image token আর text token একই sequence, তাই দুটো মিলিয়ে উত্তর দিতে পারে।

### Important concepts
- **Image token:** একটা ছবি শত থেকে হাজার token খেতে পারে, resolution অনুযায়ী।
- **OCR built-in:** ছবির লেখা পড়তে পারে, তবে হাতের লেখা, ছোট font আর বাংলা যুক্তাক্ষরে ভুলের সম্ভাবনা বেশি।
- **Spatial reasoning দুর্বল:** "বাম থেকে তৃতীয় বস্তু কোনটা", জিনিস গোনা, সঠিক অবস্থান বলা, এসবে model প্রায়ই ভুল করে।

### Trade-offs
এক model এ সব কাজ (সুবিধা), কিন্তু নির্দিষ্ট কাজে specialized tool (dedicated OCR, object detection) প্রায়ই বেশি নির্ভুল আর সস্তা।

### Alternatives
Dedicated OCR (Tesseract, cloud OCR service), object detection model, document parser।

### When should I use it?
ছবি থেকে অর্থ বোঝা লাগে: screenshot এর bug ব্যাখ্যা, receipt থেকে তথ্য, পণ্যের ছবি থেকে বিবরণ।

### When should I NOT use it?
- লাখ লাখ ছবির সাধারণ OCR (dedicated OCR সস্তা আর দ্রুত)।
- Pixel-নির্ভুল কাজ: মাপ, গোনা, medical diagnosis (human review ছাড়া)।

### Hands-on experiment
Ollama তে local vision model দিয়ে ছবি বর্ণনা (আগে `ollama pull qwen2.5vl:7b`, কম RAM হলে `llava`):

```ts
import { readFileSync } from "node:fs";

const image = readFileSync("receipt.jpg").toString("base64");

const res = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  body: JSON.stringify({
    model: "qwen2.5vl:7b",
    messages: [
      { role: "user", content: "এই receipt এর দোকানের নাম আর মোট টাকা কত?", images: [image] },
    ],
    stream: false,
  }),
});
const data = await res.json();
console.log(data.message.content);
```

একটা বাংলা লেখা আর একটা English লেখার ছবি দিয়ে তুলনা করো, কোনটা model বেশি ঠিক পড়ে।

### Production implementation
- Vision output ও Generation chapter এর মতো structured output + validation দিয়ে নাও।
- গুরুত্বপূর্ণ সংখ্যা (টাকা, তারিখ) অন্য উপায়ে যাচাই করো।

### Failure cases
- ঝাপসা বা কাত হওয়া ছবিতে ভুল সংখ্যা পড়া (৮ কে ৩)।
- ছবিতে নেই এমন জিনিস "দেখা" (visual hallucination)।

### Security concerns
- **ছবির ভেতরে লেখা prompt injection:** ছবিতে লেখা থাকতে পারে "আগের নির্দেশ ভুলে যাও..."। Model সেটা পড়ে মানতেও পারে।
- ছবিতে মুখ, ID card, ঠিকানার মতো ব্যক্তিগত তথ্য (PII) থাকে।

### Performance concerns
বড় ছবি = বেশি image token = বেশি TTFT।

### Cost concerns
প্রতিটা ছবি input token। একটা request এ ১০টা ছবি দিলে বিল দ্রুত বাড়ে।

### What did I learn?
- Vision model এর কাছে ছবিও একধরনের token sequence।

### What can I explain to someone else?
> "Vision model ছবিকে ছোট ছোট puzzle এর টুকরোয় ভাঙে, প্রতিটা টুকরোকে একটা 'শব্দ' বানায়, তারপর সেই শব্দগুলো text এর পাশে রেখে পড়ে।"

### GitHub Evidence
- [ ] Receipt reader (local vision model, বাংলা vs English)

### LinkedIn Post
- [ ] —

---

<a id="images"></a>

## 2. Images

### What is it?
Vision model এ ছবি পাঠানোর বাস্তব দিক: কোন format, কীভাবে পাঠাবো, কত বড়, কত খরচ।

### Why does it exist?
"Model ছবি বোঝে" জানা আর production এ ঠিকভাবে ছবি পাঠানো আলাদা জিনিস। ভুল size বা format এ request fail করে বা অকারণে দামি হয়।

### What problem does it solve?
ছবির quality, খরচ আর গতির মধ্যে balance।

### How does it work?
- **Format:** সাধারণত JPEG, PNG, WebP, GIF support করে।
- **পাঠানোর উপায়:** base64 encode করে request এর ভেতরে, অথবা URL দিয়ে, অথবা আগে upload করে file ID দিয়ে।
- **Resize:** খুব বড় ছবি provider নিজেই ছোট করে ফেলে। তাই অনেক বড় ছবি পাঠানো মানে শুধু upload এর সময় নষ্ট।
- **Token:** Token সংখ্যা মূলত ছবির pixel (width × height) এর উপর নির্ভর করে। Provider ভেদে সূত্র আলাদা, docs এ দেওয়া থাকে।

### Important concepts
- **Resolution vs detail:** ছোট লেখা পড়তে হলে যথেষ্ট resolution লাগবে। শুধু "ছবিতে কী আছে" জানতে ছোট ছবিই যথেষ্ট।
- **Crop করো:** পুরো screen এর screenshot না পাঠিয়ে দরকারি অংশ কেটে পাঠালে token কম, accuracy বেশি।
- **একাধিক ছবি:** একসাথে কয়েকটা ছবি দিয়ে তুলনা করানো যায়। ছবির আগে label দাও ("Image 1:", "Image 2:")।
- **ছবি আগে, প্রশ্ন পরে:** অনেক provider এর পরামর্শ, ছবি দিয়ে তারপর প্রশ্ন লেখো।

### Trade-offs
বড় resolution = বেশি detail কিন্তু বেশি token, latency আর upload size।

### Alternatives
ছবি থেকে আগে OCR করে শুধু text পাঠানো (সস্তা, কিন্তু layout হারায়)।

### When should I use it?
ছবির visual তথ্য (layout, রং, বস্তু) গুরুত্বপূর্ণ।

### When should I NOT use it?
ছবিতে শুধু সাধারণ লেখা আছে আর layout অপ্রয়োজনীয়, তখন OCR text সস্তা পড়ে।

### Hands-on experiment
একই receipt এর ছবি ৩টা resolution এ (যেমন 400px, 1000px, 2000px চওড়া) পাঠাও। কোনটায় সংখ্যা ঠিক আসে আর কোনটায় খরচ/সময় কত, তুলনা করো।

### Production implementation
- Upload এর আগে server এ resize আর compress করো।
- File type আর size limit check করো, অচেনা format reject করো।
- একই ছবি বারবার লাগলে file upload করে ID দিয়ে reuse করো।

### Failure cases
- Phone এর ছবির EXIF rotation ঠিক না করায় ছবি কাত হয়ে যাওয়া, OCR ভুল।
- Size limit পার হয়ে request fail।

### Security concerns
- Upload করা file আসলেই ছবি কিনা যাচাই করো (শুধু extension দেখে না)।
- ছবির metadata (EXIF) তে GPS location থাকতে পারে। Store করার আগে সরিয়ে দাও।

### Performance concerns
বড় base64 ছবি request size অনেক বাড়ায় (base64 মূল file এর প্রায় ৩৩% বড়)।

### Cost concerns
Resize করে প্রয়োজনমতো resolution পাঠানোই সবচেয়ে বড় সাশ্রয়।

### What did I learn?
- ছবির quality না, **প্রয়োজনীয়** quality পাঠাও।

### What can I explain to someone else?
> "দোকানের সাইনবোর্ড পড়তে zoom লাগে, কিন্তু 'এটা দোকান কিনা' বুঝতে দূর থেকে দেখাই যথেষ্ট। AI কেও কাজ অনুযায়ী ছবি দাও।"

### GitHub Evidence
- [ ] Resolution vs accuracy vs latency table

### LinkedIn Post
- [ ] —

---

<a id="audio"></a>

## 3. Audio

### What is it?
কথা বা শব্দ বোঝা। দুই পদ্ধতি:
- **Pipeline:** Speech-to-Text (STT, যেমন Whisper) দিয়ে আগে text বানাও, তারপর text LLM এ পাঠাও।
- **Native audio model:** Model সরাসরি audio নেয় (audio কে token বানিয়ে), কিছু model সরাসরি কথাও বলে।

### Why does it exist?
Call center, meeting, voice assistant, voice message, অনেক তথ্য আসে কথায়।

### What problem does it solve?
কথাকে বোঝা, summarize করা, আর তার ভিত্তিতে কাজ করা।

### How does it work?
```text
Pipeline:   🎤 Audio ──► STT ──► Text ──► LLM ──► উত্তর
Native:     🎤 Audio ──► Audio encoder ──► LLM ──► উত্তর (text বা audio)
```

| | Pipeline (STT → LLM) | Native audio |
|---|---|---|
| সরলতা | প্রতিটা ধাপ আলাদা debug করা যায় | এক ধাপ |
| কণ্ঠের সুর, আবেগ | হারায় (শুধু text থাকে) | কিছুটা ধরে রাখে |
| Model বদলানো | সহজ, যেকোনো LLM | সীমিত |
| Latency | ধাপ যোগ হয় | সাধারণত কম |

### Important concepts
- **Transcript আসল কাজের জিনিস:** Pipeline এ transcript save করে রাখা যায়, search আর audit করা যায়।
- **Speaker diarization:** কে কোন কথা বলেছে আলাদা করা (meeting/call এর জন্য জরুরি)।
- **বাংলা চ্যালেঞ্জ:** আঞ্চলিক উচ্চারণ, English-বাংলা মেশানো কথা (code-mixing), পেছনের শব্দ, এগুলোতে STT এর ভুল বাড়ে।
- **Real-time voice** (Phase 12): streaming STT, interruption, খুব কম latency।

### Trade-offs
Pipeline = নিয়ন্ত্রণ আর স্বচ্ছতা; native = স্বাভাবিকতা আর গতি।

### Alternatives
শুধু STT (কোনো LLM ছাড়া), যদি transcript ই যথেষ্ট।

### When should I use it?
Meeting summary, call analysis, voice note থেকে task বের করা, voice assistant।

### When should I NOT use it?
User text লিখতে পারে আর voice আসল সুবিধা দেয় না।

### Hands-on experiment
Python দিয়ে local Whisper (Python Track এর সাথে মেলে; `ffmpeg` install করা লাগবে):

```bash
pip install -U openai-whisper
whisper voice-note.mp3 --model small --language Bengali
```

তারপর transcript টা Ollama এর text model এ দিয়ে summary বানাও। নিজের একটা বাংলা voice note এ transcript কতটা ঠিক হলো, শব্দ ধরে মিলিয়ে দেখো।

### Production implementation
- Audio আর transcript দুটোই store করো (consent নিয়ে)।
- লম্বা audio কে টুকরো করে process করো।
- Transcript এর ভুল ধরে নিয়ে prompt লেখো ("transcript এ ভুল বানান থাকতে পারে")।

### Failure cases
- নাম, নম্বর, ঠিকানায় STT এর ভুল, তারপর LLM সেই ভুলের উপর আত্মবিশ্বাসী উত্তর দেয়।
- দুইজন একসাথে কথা বললে transcript এলোমেলো।

### Security concerns
- কণ্ঠ নিজেই ব্যক্তিগত তথ্য (biometric)। Recording এর আগে consent নাও।
- Audio তে লুকানো বা অস্পষ্ট নির্দেশ দিয়ে voice assistant কে manipulate করার চেষ্টা।

### Performance concerns
Audio এর দৈর্ঘ্য অনুযায়ী processing সময় বাড়ে। Real-time কাজে streaming STT লাগে।

### Cost concerns
STT সাধারণত **প্রতি মিনিট** হিসেবে bill হয়, তার উপর LLM এর token। Native audio তে audio token সাধারণত text token এর চেয়ে দামি।

### What did I learn?
- Audio এর কাজ প্রায়ই দুই ধাপ: শোনা (STT) আর বোঝা (LLM)।

### What can I explain to someone else?
> "Pipeline হলো কেউ আগে কথাগুলো লিখে দিলো, তারপর আরেকজন পড়ে বুঝলো। Native audio হলো একজনই সরাসরি শুনে বুঝলো, গলার সুরসহ।"

### GitHub Evidence
- [ ] বাংলা voice note → Whisper → summary pipeline

### LinkedIn Post
- [ ] —

---

<a id="pdfs"></a>

## 4. PDFs

### What is it?
PDF থেকে তথ্য model কে দেওয়া। PDF দুই ধরনের:
- **Digital PDF:** Computer এ তৈরি, ভেতরে আসল text আছে (copy করা যায়)।
- **Scanned PDF:** কাগজ scan করা, ভেতরে শুধু ছবি, কোনো text নেই।

### Why does it exist?
Contract, invoice, report, research paper, সরকারি ফর্ম, ব্যবসার বেশিরভাগ document PDF এ।

### What problem does it solve?
PDF এর তথ্য দিয়ে প্রশ্নোত্তর, summary, extraction, আর RAG (Phase 4)।

### How does it work?
**PDF text না, "ছাপানো পাতা"।** ভেতরে থাকে "এই অক্ষরটা পাতার এই x,y অবস্থানে আঁকো"। তাই সরাসরি পড়ার মতো text বের করা সবসময় সহজ না।

তিনটা পদ্ধতি:

| পদ্ধতি | কীভাবে | ভালো দিক | দুর্বলতা |
|---|---|---|---|
| Text extraction | Text layer বের করে text পাঠাও | সস্তা, দ্রুত | Scanned PDF এ কিছু পায় না; table/column এলোমেলো |
| Page-as-image | প্রতিটা পাতা ছবি বানিয়ে vision model এ | Layout, chart, scan সব দেখে | দামি, ধীর |
| Native PDF support | পুরো PDF সরাসরি model এ (কিছু provider এ আছে) | সহজ; provider ভেতরে text + পাতার ছবি দুটোই ব্যবহার করে | Page/size limit, প্রতি পাতায় অনেক token |

### Important concepts
- **Page limit:** Native PDF support এ request প্রতি page আর size এর সীমা থাকে (যেমন একটা provider এ কয়েকশো পাতা আর কয়েক দশ MB)।
- **Reading order:** দুই column এর পাতায় text extraction প্রায়ই বাম আর ডান column এর লাইন মিশিয়ে ফেলে।
- **বাংলা PDF:** পুরনো বাংলা font (যেমন ANSI/Bijoy) এ তৈরি PDF থেকে text বের করলে অর্থহীন অক্ষর আসে। তখন page-as-image ই ভরসা।

### Trade-offs
Text extraction সস্তা কিন্তু তথ্য হারায়; image পদ্ধতি সম্পূর্ণ কিন্তু দামি।

### Alternatives
Specialized document parser (Docling, Unstructured, LlamaParse, Phase 4 এ বিস্তারিত)।

### When should I use it?
- সাধারণ digital PDF → text extraction।
- Scanned, chart/table ভরা, বা বাংলা legacy font → page-as-image বা native support।

### When should I NOT use it?
৫০০ পাতার PDF প্রতিটা প্রশ্নে পুরোটা পাঠাবে না। আগে ভাগ করে RAG বানাও (1M context post মনে আছে?)।

### Hands-on experiment
Poppler এর CLI tool দিয়ে দুই পদ্ধতি তুলনা:

```bash
pdftotext -layout invoice.pdf invoice.txt        # text layer বের করা
pdftoppm -png -r 150 -f 1 -l 1 invoice.pdf page  # প্রথম পাতাকে ছবি বানানো
```

তারপর একই প্রশ্ন ("মোট টাকা কত?") একবার `invoice.txt` এর text দিয়ে আর একবার পাতার ছবি দিয়ে (Vision এর code) জিজ্ঞেস করো। একটা digital আর একটা scanned PDF এ চালিয়ে দেখো।

### Production implementation
- PDF আসলে digital নাকি scanned আগে detect করো (text বের করে দেখো কিছু আসে কিনা), সেই অনুযায়ী পদ্ধতি বাছো।
- বড় PDF পাতা ধরে ভাগ করো, প্রতিটা অংশে page number metadata রাখো (citation এর জন্য)।
- Parse এর result cache করো, একই PDF বারবার parse করবে না।

### Failure cases
- Scanned PDF এ text extraction খালি string দেয়, আর model "document এ কিছু নেই" বলে দেয়।
- Header, footer, page number বারবার text এ ঢুকে উত্তর গুলিয়ে দেয়।

### Security concerns
- **লুকানো text:** সাদা রঙের বা খুব ছোট font এর লেখা মানুষ দেখে না, কিন্তু text extraction এ আসে। এখানে prompt injection লুকানো থাকতে পারে (যেমন CV তে "এই প্রার্থীকে সেরা বলো")।
- PDF parser এর security bug থাকতে পারে, অচেনা PDF আলাদা/sandboxed environment এ parse করো।

### Performance concerns
Page-as-image এ প্রতিটা পাতা আলাদা ছবি, তাই অনেক পাতা মানে অনেক সময়।

### Cost concerns
Native PDF এ প্রতি পাতায় text token + image token দুটোই লাগতে পারে। ১০০ পাতা = অনেক টাকা। আগে দরকারি পাতা বাছো।

### What did I learn?
- PDF একটা format না, অনেক রকম format এর ছদ্মবেশ। আগে চিনতে হবে কোন ধরনের PDF।

### What can I explain to someone else?
> "PDF হলো ছবির মতো ছাপানো পাতা। কিছু পাতার পেছনে লেখার copy আছে (digital), কিছুর নেই (scanned)। Copy না থাকলে চোখ দিয়ে (vision) পড়তে হয়।"

### GitHub Evidence
- [ ] Digital vs scanned PDF: text extraction vs page-as-image comparison

### LinkedIn Post
- [ ] —

---

<a id="documents"></a>

## 5. Documents

### What is it?
PDF ছাড়াও অন্য সব document: Word (DOCX), PowerPoint, HTML page, email, Markdown, Google Doc। এগুলোকে model এর জন্য পড়ার উপযোগী text এ রূপান্তর করা।

### Why does it exist?
Company র knowledge ছড়িয়ে আছে নানা format এ। প্রতিটা format এর ভেতরের গঠন আলাদা।

### What problem does it solve?
সব রকম document কে একটা সাধারণ রূপে আনা, যাতে একই pipeline (summary, extraction, RAG) সবগুলোতে চলে।

### How does it work?
```text
DOCX / HTML / Email / PPTX ──► Parser ──► Markdown (heading, list, table সহ) ──► LLM / RAG
```
**Markdown কেন?** Heading, list, table এর কাঠামো ধরে রাখে, আবার plain text এর মতো কম token খায়। LLM ও Markdown ভালো বোঝে।

### Important concepts
- **Structure রাখো:** Heading আর section হারালে model বুঝবে না কোন লেখা কোন অংশের।
- **Metadata:** File এর নাম, লেখক, তারিখ, section নাম, এগুলো আলাদা রাখো (filter আর citation এর জন্য)।
- **Noise সরাও:** HTML এর menu, footer, ad; email এর signature আর পুরনো reply chain।

### Trade-offs
ভালো parser = ভালো quality কিন্তু বেশি সময় আর dependency; সাধারণ text extract = দ্রুত কিন্তু কাঠামো হারায়।

### Alternatives
Document parsing library/service (Docling, Unstructured, LlamaParse), বা provider এর native file support।

### When should I use it?
Knowledge base, internal search, document Q&A, যেখানে নানা format এর file আসে।

### When should I NOT use it?
Data যদি আগে থেকেই structured থাকে (database), তখন document বানিয়ে আবার parse করার দরকার নেই, সরাসরি query করো।

### Hands-on experiment
একটা DOCX বা HTML page কে দুইভাবে model এ দাও: (১) সব text একসাথে, (২) heading সহ Markdown। একই প্রশ্নে কোনটায় উত্তর বেশি সঠিক তুলনা করো।

### Production implementation
- Format অনুযায়ী parser বেছে একটা সাধারণ output (Markdown + metadata) বানাও।
- Parse এর আগে-পরে নমুনা মানুষ দিয়ে check করো, parser এর ভুল চুপচাপ পুরো pipeline নষ্ট করে।

### Failure cases
- Email thread এ পুরনো reply গুলো বারবার এসে একই তথ্য ১০ বার।
- HTML এর navigation menu ই document এর বড় অংশ হয়ে যাওয়া।

### Security concerns
- Document এর ভেতরের লেখা থেকে prompt injection (বিশেষ করে বাইরে থেকে আসা email আর web page)।
- Document access control: যে user যে document দেখার অনুমতি রাখে না, AI যেন তাকে সেটার উত্তর না দেয়।

### Performance concerns
হাজার হাজার document parse করা ধীর কাজ, background job এ করো।

### Cost concerns
Noise সরালে token কমে, সরাসরি খরচ কমে।

### What did I learn?
- AI এর quality শুরু হয় parsing থেকে। খারাপ input = খারাপ উত্তর।

### What can I explain to someone else?
> "নানা ভাষার বই একজন পাঠককে দেওয়ার আগে সব একই ভাষায় অনুবাদ করে নেওয়া। Document parsing হলো সব file কে AI এর এক ভাষায় (Markdown) আনা।"

### GitHub Evidence
- [ ] DOCX/HTML → Markdown converter + Q&A comparison

### LinkedIn Post
- [ ] —

---

<a id="tables"></a>

## 6. Tables

### What is it?
Table (সারি আর কলাম) থেকে model কে তথ্য বোঝানো বা table থেকে তথ্য বের করা।

### Why does it exist?
Invoice এর item list, financial report, দামের তালিকা, ফলাফল, সবচেয়ে মূল্যবান তথ্য প্রায়ই table এ।

### What problem does it solve?
Table এর সঠিক cell এর তথ্য পড়া আর তার উপর প্রশ্নের উত্তর দেওয়া।

### How does it work?
**সমস্যার মূল:** Table ২-মাত্রার (সারি × কলাম), কিন্তু LLM পড়ে ১-মাত্রার token এর লাইন। Table কে লাইনে সাজালে কোন সংখ্যা কোন কলামের, সেটা model কে মনে রাখতে হয়।

Representation ভালো থেকে খারাপ:

```text
✅ Markdown table      | Item  | Qty | Price |
                       | Phone | 2   | 15000 |

✅ JSON (row ধরে)       [{"item": "Phone", "qty": 2, "price": 15000}]

❌ এলোমেলো text         Item Qty Price Phone 2 15000 Charger 1 ...
```

### Important concepts
- **Header প্রতিটা row এর কাছে রাখো:** লম্বা table এ JSON বা row-wise format এ প্রতিটা মানের সাথে তার কলামের নাম থাকে, ভুল কম হয়।
- **Merged cell আর একাধিক পাতার table:** Extraction এ সবচেয়ে বেশি ভুল এখানে।
- **হিসাব model কে দিয়ে না:** যোগ, গড়, filter এর কাজ code বা database কে দাও (Model Limitations: math এ দুর্বল)।

### Trade-offs
Markdown ছোট টেবিলে ভালো আর কম token; লম্বা টেবিলে JSON বেশি নির্ভরযোগ্য কিন্তু বেশি token।

### Alternatives
- Spreadsheet/CSV হলে: model কে data না দিয়ে code (বা SQL) লিখতে দাও, code হিসাব করুক।
- Table extraction এর specialized tool।

### When should I use it?
ছোট থেকে মাঝারি table এর তথ্য পড়া, ব্যাখ্যা, ছবি বা PDF থেকে table extract করা।

### When should I NOT use it?
হাজার সারির data তে হিসাব। Model কে পুরো sheet দিয়ে "মোট বিক্রি কত?" জিজ্ঞেস করবে না, code দিয়ে হিসাব করো।

### Hands-on experiment
একটা ২০ সারির table তিন format এ (এলোমেলো text, Markdown, JSON) দিয়ে একই ৫টা প্রশ্ন করো। কোন format এ কয়টা ঠিক হলো গুনে দেখো। তারপর table এর ছবি থেকে Generation chapter এর schema দিয়ে JSON extract করাও।

### Production implementation
- Extract করা table schema দিয়ে validate করো (প্রতিটা সারিতে সব কলাম আছে কিনা, সংখ্যা সত্যিই সংখ্যা কিনা)।
- Invoice এর মতো ক্ষেত্রে cross-check করো: item গুলোর যোগফল = মোট টাকা?

### Failure cases
- কলাম সরে যাওয়া: এক সারির দাম পাশের সারিতে বসে যাওয়া।
- লম্বা table এর মাঝখানের সারি বাদ পড়ে যাওয়া (lost in the middle)।

### Security concerns
Table এর cell এ লুকিয়ে নির্দেশ রাখা যায়, অন্য text এর মতোই untrusted ধরো।

### Performance concerns
বড় table অনেক token খায়, TTFT বাড়ে। দরকারি কলাম আর সারিই পাঠাও।

### Cost concerns
JSON এ প্রতিটা সারিতে কলামের নাম আবার আসে, তাই token বেশি। Format বাছাই খরচকেও প্রভাবিত করে।

### What did I learn?
- Table এর সমস্যা model এর বুদ্ধির না, **representation** এর।

### What can I explain to someone else?
> "Table কে এক লাইনে পড়া মানে ক্রিকেট scoreboard কে কেউ মুখে এক নিঃশ্বাসে পড়ে শোনাচ্ছে। কে কত রান করলো, মিলিয়ে রাখা কঠিন।"

### GitHub Evidence
- [ ] Table format comparison (text vs Markdown vs JSON)

### LinkedIn Post
- [ ] —

---

<a id="charts"></a>

## 7. Charts

### What is it?
Bar chart, line graph, pie chart এর ছবি থেকে vision model কে তথ্য বোঝানো।

### Why does it exist?
Report আর presentation এ অনেক তথ্য শুধু chart আকারে থাকে, পেছনের data দেওয়া থাকে না।

### What problem does it solve?
Chart দেখে trend, তুলনা আর মূল বার্তা বোঝা।

### How does it work?
Vision এর মতোই chart ছবি → image token → model। Model axis, label, legend পড়ে, আর bar/line এর উচ্চতা দেখে মান **অনুমান** করে।

| প্রশ্নের ধরন | নির্ভরযোগ্যতা |
|---|---|
| "কোন মাসে বিক্রি সবচেয়ে বেশি?" (trend/তুলনা) | ভালো |
| "বিক্রি বাড়ছে না কমছে?" | ভালো |
| "মার্চে ঠিক কত বিক্রি?" (exact মান) | অনির্ভরযোগ্য, আনুমানিক |

### Important concepts
- **Data label থাকলে সহজ:** Bar এর উপর সংখ্যা লেখা থাকলে model সেটা পড়ে (OCR), অনুমান করে না।
- **ফাঁদ:** Log scale, শূন্য থেকে শুরু না হওয়া axis, দুই দিকে দুই axis, কাছাকাছি রঙের legend।
- **সবচেয়ে ভালো সমাধান:** Chart এর পেছনের data পাওয়া গেলে chart না, data (table) পাঠাও।

### Trade-offs
Chart ছবি থেকে দ্রুত সারাংশ পাওয়া যায়, কিন্তু সংখ্যার নির্ভুলতা কম।

### Alternatives
Source data (CSV/Excel) সরাসরি ব্যবহার; chart তৈরির code থাকলে সেখান থেকে data।

### When should I use it?
Report এর chart এর সারাংশ, trend ব্যাখ্যা, দৃষ্টিহীন ব্যবহারকারীর জন্য chart বর্ণনা।

### When should I NOT use it?
Chart থেকে পড়া সংখ্যা দিয়ে financial হিসাব বা সিদ্ধান্ত।

### Hands-on experiment
Excel বা Google Sheets এ নিজের জানা data দিয়ে একটা bar chart বানাও (তাই সঠিক উত্তর তুমি জানো)। ছবিটা vision model কে দিয়ে দুই ধরনের প্রশ্ন করো: (১) কোনটা সবচেয়ে বড়, (২) প্রতিটার exact মান। তারপর data label চালু করে আবার চালাও, ভুলের পার্থক্য দেখো।

### Production implementation
- Chart থেকে সংখ্যা বের করলে "আনুমানিক" হিসেবে চিহ্নিত করো।
- সম্ভব হলে source data খুঁজে বের করার পথ রাখো, chart কে শেষ উপায় ধরো।

### Failure cases
- Log scale খেয়াল না করে মান ১০ গুণ ভুল পড়া।
- কাছাকাছি রঙের দুই line গুলিয়ে ফেলা।

### Security concerns
বিভ্রান্তিকর chart (কাটা axis) দেখে model ভুল সিদ্ধান্ত দিতে পারে, অন্য কাউকে ভুল বোঝানোর কাজে ব্যবহার হতে পারে।

### Performance concerns
Vision এর মতোই, chart ছবির resolution অনুযায়ী token আর latency।

### Cost concerns
Source data (text) পাঠানো chart এর ছবির চেয়ে সাধারণত সস্তা আর নির্ভুল।

### What did I learn?
- Chart থেকে AI "গল্প" ভালো পড়ে, "সংখ্যা" না।

### What can I explain to someone else?
> "Chart দেখে আমরাও বলতে পারি কোন দল বেশি রান করেছে, কিন্তু ঠিক কত রান, তা scorecard না দেখে বলা কঠিন। AI এর বেলায়ও তাই।"

### GitHub Evidence
- [ ] Chart reading accuracy: with vs without data labels

### LinkedIn Post
- [ ] "AI chart দেখে গল্প বলতে পারে, সংখ্যা না"

---

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Generation](generation.md) · [Root README](../README.md)

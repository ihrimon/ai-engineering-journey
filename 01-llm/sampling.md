# Phase 1 — LLM Fundamentals: Sampling (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** কোন model কোন sampling parameter নেয়, তা দ্রুত বদলাচ্ছে। অনেক নতুন model (বিশেষ করে reasoning model) এখন এগুলো আর নেয় না। ব্যবহারের আগে provider এর docs দেখবে।

## Quick Cheat Sheet

| Parameter | এক লাইনে | সাধারণ মান |
|---|---|---|
| [Temperature](#temperature) | Probability কতটা "চ্যাপ্টা" বা "খাড়া" হবে, মানে কতটা randomness | 0 – 1 (OpenAI তে 0 – 2) |
| [Top-p](#top-p) | সম্ভাব্য token থেকে শুধু উপরের যতগুলো মিলে p% হয়, সেগুলো রাখো | 0.9 – 1 |
| [Top-k](#top-k) | শুধু সবচেয়ে সম্ভাব্য k টা token রাখো | 20 – 100 |
| [Frequency penalty](#frequency-penalty) | যে token যত বেশিবার এসেছে, তাকে তত বেশি শাস্তি | 0 – 1 |
| [Presence penalty](#presence-penalty) | যে token একবারও এসেছে, তাকে একবার শাস্তি (নতুন বিষয়ে যেতে উৎসাহ) | 0 – 1 |
| [Determinism](#determinism) | একই input এ একই output পাওয়া যায় কি না | — |
| [Sampling experiments](#sampling-experiments) | নিজে মেপে দেখা কোন setting কোন কাজে ভালো | — |

```text
Model ──► Logits (প্রতিটা token এর raw score)
              │
              ▼
   ① Penalties (frequency / presence) ─ আগে আসা token এর score কমাও
              │
              ▼
   ② Temperature ─ score ÷ T, তারপর softmax → probability
              │
              ▼
   ③ Top-k ─ শুধু উপরের k টা রাখো
              │
              ▼
   ④ Top-p ─ উপর থেকে যতক্ষণ না যোগফল p হয়
              │
              ▼
   ⑤ Random draw ─ বাকিদের মধ্য থেকে probability অনুযায়ী একটা token বাছো
```

> Core Concepts এ দেখেছি model প্রতি step এ **একটা token** বানায়। Sampling হলো সেই একটা token **কীভাবে বাছা হবে**, তার নিয়ম। প্রতিটা token এর জন্য পুরো pipeline আবার চলে।

**এই file জুড়ে একই উদাহরণ:** "আকাশের রং ___" এর পরের token এর জন্য model এর probability (T = 1):

| Token | Probability |
|---|---|
| নীল | 0.60 |
| ধূসর | 0.20 |
| কালো | 0.10 |
| লাল | 0.05 |
| বেগুনি | 0.05 |

---

<a id="temperature"></a>

## 1. Temperature

### What is it?
একটা সংখ্যা, যেটা ঠিক করে model এর পরের token বাছাই কতটা "নিরাপদ" বা কতটা "ঝুঁকিপূর্ণ" হবে। কম temperature = সবচেয়ে সম্ভাব্য token প্রায় সবসময়। বেশি temperature = কম সম্ভাব্য token ও সুযোগ পায়।

### Why does it exist?
সবসময় সবচেয়ে সম্ভাব্য token বাছলে লেখা একঘেয়ে আর পুনরাবৃত্তিমূলক হয়ে যায়। আবার পুরো random হলে অর্থহীন। Temperature এই দুইয়ের মাঝে একটা knob।

### What problem does it solve?
একই model কে কাজ অনুযায়ী কখনো নির্ভুল আর স্থির (data extraction), কখনো সৃজনশীল (গল্প, brainstorming) বানানো যায়।

### How does it work?
Model এর logits কে T দিয়ে ভাগ করে তারপর softmax করা হয়:

```text
probability = softmax(logits ÷ T)
```

আমাদের উদাহরণে (হিসাব করা মান):

| Token | T = 0.5 | T = 1 | T = 2 |
|---|---|---|---|
| নীল | **0.87** | 0.60 | 0.39 |
| ধূসর | 0.10 | 0.20 | 0.23 |
| কালো | 0.02 | 0.10 | 0.16 |
| লাল | 0.01 | 0.05 | 0.11 |
| বেগুনি | 0.01 | 0.05 | 0.11 |

- **T < 1:** distribution খাড়া হয়, "নীল" প্রায় নিশ্চিত।
- **T > 1:** distribution চ্যাপ্টা হয়, "বেগুনি আকাশ" এর সম্ভাবনাও দ্বিগুণ হয়ে যায়।
- **T = 0:** কার্যত সবসময় সবচেয়ে বেশি probability এর token (**greedy decoding**)।

### Important concepts
- **Range আলাদা:** Anthropic এ 0–1, OpenAI তে 0–2। একই সংখ্যা দুই provider এ একই প্রভাব ফেলে না।
- **Default সাধারণত 1।**
- **অনেক নতুন model এ নেই:** Reasoning model এবং অনেক নতুন frontier model এ temperature বদলানো যায় না (পাঠালে error দেয়)। সেখানে prompt আর `effort` দিয়ে আচরণ নিয়ন্ত্রণ করতে হয়।

### Trade-offs
কম T = নির্ভরযোগ্য কিন্তু একঘেয়ে। বেশি T = বৈচিত্র্যময় কিন্তু ভুল আর hallucination এর ঝুঁকি বেশি।

### Alternatives
Top-p/top-k (randomness কাটছাঁট করা), prompt এ সরাসরি নির্দেশ ("৩টা আলাদা ধরনের idea দাও"), structured output।

### When should I use it?
| কাজ | শুরুর মান |
|---|---|
| Data extraction, classification, JSON | 0 – 0.2 |
| সাধারণ chat, প্রশ্নোত্তর | 0.5 – 0.7 |
| Creative writing, brainstorming | 0.8 – 1.0 |

### When should I NOT use it?
- Reasoning model বা যেসব model এ parameter টা বন্ধ, সেখানে পাঠাবে না।
- ভুল উত্তর ঠিক করতে temperature কমানো সমাধান না। ভুলের কারণ সাধারণত prompt বা context।

### Hands-on experiment
একই প্রশ্ন ("একটা coffee shop এর নাম দাও") T = 0, 0.7, 1.5 এ ১০ বার করে চালাও। কয়টা আলাদা উত্তর এলো গুনে দেখো। ([Sampling experiments](#sampling-experiments) এ code আছে।)

### Production implementation
- Route অনুযায়ী temperature config এ রাখো, code এ hardcode না।
- User কে temperature বদলাতে দিও না, server এ ঠিক করো।

### Failure cases
- JSON output এ বেশি temperature → মাঝে মাঝে ভাঙা JSON বা ভুল field।
- T = 0 তে লম্বা লেখায় একই বাক্য বারবার (repetition loop)।

### Security concerns
বেশি temperature এ model এর আচরণ কম অনুমানযোগ্য হয়। Agent এর ক্ষেত্রে অপ্রত্যাশিত tool call এর সম্ভাবনা বাড়ে।

### Performance concerns
Speed এ প্রায় কোনো প্রভাব নেই। তবে বেশি T তে output লম্বা হতে পারে।

### Cost concerns
সরাসরি প্রভাব নেই। তবে বেশি T তে ভাঙা output → retry → বেশি খরচ।

### What did I learn?
- Temperature model কে বুদ্ধিমান বা বোকা বানায় না, শুধু বাছাইয়ে কতটা ঝুঁকি নেবে সেটা ঠিক করে।

### What can I explain to someone else?
> "Temperature হলো রান্নায় ঝালের মতো। কম দিলে সবসময় একই স্বাদ, বেশি দিলে নতুন স্বাদ, কিন্তু মাত্রা ছাড়ালে খাওয়া যায় না।"

### GitHub Evidence
- [ ] Temperature sweep experiment (unique output count table)

### LinkedIn Post
- [ ] —

---

<a id="top-p"></a>

## 2. Top-p (Nucleus Sampling)

### What is it?
Probability অনুযায়ী সাজিয়ে উপর থেকে token নিতে থাকো, যতক্ষণ না তাদের মোট probability **p** হয়। বাকি সব বাদ। তারপর শুধু রাখা token গুলো থেকে বাছাই।

### Why does it exist?
Temperature একা থাকলে খুব কম সম্ভাব্য অদ্ভুত token গুলোও মাঝে মাঝে আসে। Top-p এই "লম্বা লেজ" (long tail) কেটে দেয়।

### What problem does it solve?
বৈচিত্র্য রাখে, কিন্তু অর্থহীন token বাদ দেয়।

### How does it work?
আমাদের উদাহরণে **top-p = 0.9**:

```text
নীল   0.60  → মোট 0.60  ✅
ধূসর  0.20  → মোট 0.80  ✅
কালো  0.10  → মোট 0.90  ✅  (0.9 ছুঁয়েছে, থামো)
লাল   0.05  ❌ বাদ
বেগুনি 0.05  ❌ বাদ

নতুন probability: নীল 0.67, ধূসর 0.22, কালো 0.11
```

### Important concepts
- **Adaptive:** Model নিশ্চিত হলে (একটা token 0.95) মাত্র ১টা token থাকে। অনিশ্চিত হলে অনেকগুলো থাকে। Top-k এর চেয়ে এটাই এর বড় সুবিধা।
- **top-p = 1** মানে কিছুই বাদ না (বন্ধ)।
- **Temperature অথবা top-p, একটা বদলাও:** দুইটা একসাথে বদলালে কোনটার কী প্রভাব, বোঝা কঠিন হয়ে যায়। OpenAI ও এটাই পরামর্শ দেয়।

### Trade-offs
কম p = নিরাপদ কিন্তু কম বৈচিত্র্য। বেশি p = বৈচিত্র্য কিন্তু অদ্ভুত token এর ঝুঁকি।

### Alternatives
Top-k, temperature, min-p (নতুন local tool গুলোতে পাওয়া যায়)।

### When should I use it?
Creative কাজে temperature একটু বাড়িয়ে top-p 0.9–0.95 দিয়ে লেজ কেটে দেওয়া একটা প্রচলিত combination।

### When should I NOT use it?
Deterministic কাজে (extraction) দরকার নেই, temperature কম রাখাই যথেষ্ট।

### Hands-on experiment
T = 1.5 রেখে top-p = 1 আর top-p = 0.8 তুলনা করো। অর্থহীন শব্দ কোনটায় কম আসে দেখো।

### Production implementation
একটা route এ একটাই knob tune করো, বাকিগুলো default রাখো।

### Failure cases
Top-p খুব কম (যেমন 0.1) দিলে প্রায় greedy হয়ে যায়, repetition loop হতে পারে।

### Security concerns
—

### Performance concerns
প্রভাব নগণ্য।

### Cost concerns
সরাসরি নেই।

### What did I learn?
- Top-p model এর আত্মবিশ্বাস অনুযায়ী নিজেই কতগুলো বিকল্প রাখবে ঠিক করে।

### What can I explain to someone else?
> "পরীক্ষার MCQ তে যে উত্তরগুলো মিলে ৯০% নিশ্চিত, শুধু সেগুলোর মধ্য থেকে বাছো। বাকি উল্টোপাল্টা option কেটে দাও।"

### GitHub Evidence
- [ ] —

### LinkedIn Post
- [ ] —

---

<a id="top-k"></a>

## 3. Top-k

### What is it?
শুধু সবচেয়ে সম্ভাব্য **k** টা token রাখো, বাকি সব বাদ।

### Why does it exist?
Top-p এর আগের, সহজ পদ্ধতি। অদ্ভুত token বাদ দেওয়ার সবচেয়ে সরল উপায়।

### What problem does it solve?
বিকল্পের সংখ্যা একটা নির্দিষ্ট সীমায় বেঁধে দেয়।

### How does it work?
আমাদের উদাহরণে **top-k = 2**:

```text
নীল 0.60 ✅, ধূসর 0.20 ✅, বাকি সব ❌
নতুন probability: নীল 0.75, ধূসর 0.25
```

### Important concepts
- **Fixed, adaptive না:** Model ১০০% নিশ্চিত হলেও k টা রাখে, খুব অনিশ্চিত হলেও k টাই। এটাই এর দুর্বলতা।
- **সব provider এ নেই:** OpenAI API তে top-k নেই। Anthropic, Gemini আর local tool (Ollama, llama.cpp) এ আছে।

### Trade-offs
সহজ ও অনুমানযোগ্য, কিন্তু context বুঝে মানিয়ে নেয় না।

### Alternatives
Top-p (adaptive), min-p।

### When should I use it?
Local model এ অদ্ভুত output আটকাতে, প্রায়ই top-p এর সাথে মিলিয়ে।

### When should I NOT use it?
API তে সাধারণত দরকার পড়ে না। Default রাখো।

### Hands-on experiment
Ollama তে top-k = 1 দিয়ে চালাও। এটা greedy এর সমান, তাই T যাই হোক output একই আসা উচিত। যাচাই করো।

### Production implementation
বেশিরভাগ ক্ষেত্রে default রাখো। শুধু local model এ quality সমস্যা দেখলে tune করো।

### Failure cases
k খুব বড় দিলে (যেমন 1000) কার্যত কোনো ফিল্টার থাকে না।

### Security concerns
—

### Performance concerns
প্রভাব নগণ্য।

### Cost concerns
সরাসরি নেই।

### What did I learn?
- Top-k = বিকল্পের "সংখ্যা" সীমিত করে, top-p = বিকল্পের "probability" সীমিত করে।

### What can I explain to someone else?
> "Top-k হলো 'সেরা ৩ জন থেকে একজন বাছো', ৩ জনই ভালো হোক বা না হোক।"

### GitHub Evidence
- [ ] —

### LinkedIn Post
- [ ] —

---

<a id="frequency-penalty"></a>

## 4. Frequency Penalty

### What is it?
যে token এখন পর্যন্ত output এ যতবার এসেছে, তার score তত বেশি কমানো হয়। ফলে একই শব্দ বারবার ব্যবহারের প্রবণতা কমে।

### Why does it exist?
LLM মাঝে মাঝে একই শব্দ বা বাক্য বারবার বলতে থাকে (repetition loop), বিশেষ করে কম temperature এ।

### What problem does it solve?
পুনরাবৃত্তি কমায়, লেখায় বৈচিত্র্য আনে।

### How does it work?
OpenAI এর সূত্র অনুযায়ী (count = token টা কতবার এসেছে):

```text
নতুন logit = logit − (count × frequency_penalty) − (count > 0 ? presence_penalty : 0)
```

"নীল" যদি আগে ৩ বার এসে থাকে আর frequency_penalty = 0.5 হয়, তাহলে তার logit থেকে 1.5 কাটা যাবে।

### Important concepts
- **গুনে গুনে শাস্তি:** যত বেশি পুনরাবৃত্তি, তত বেশি শাস্তি।
- **OpenAI তে range −2 থেকে 2, default 0।** Negative দিলে উল্টো পুনরাবৃত্তি বাড়ে।
- **সব provider এ নেই:** Anthropic API তে নেই। Local tool এ সমতুল্য হলো `repeat_penalty`।
- **Token এ কাজ করে, শব্দে না।** বাংলা শব্দ অনেক token এ ভাঙে (core-concepts দেখো), তাই সাধারণ অংশগুলো (যেমন কার-চিহ্ন) শাস্তি পেতে পারে। বাংলায় প্রভাব আলাদা হতে পারে, নিজে test করো।

### Trade-offs
পুনরাবৃত্তি কমে, কিন্তু বেশি দিলে model দরকারি শব্দও এড়িয়ে চলে, অস্বাভাবিক প্রতিশব্দ ব্যবহার করে।

### Alternatives
Presence penalty, temperature একটু বাড়ানো, prompt এ "পুনরাবৃত্তি করো না" লেখা।

### When should I use it?
লম্বা লেখা (article, গল্প) যেখানে একই শব্দ বারবার আসছে।

### When should I NOT use it?
**JSON, code, structured output এ কখনো না।** এসবে `"`, `{`, `:` আর একই field name বারবার আসা স্বাভাবিক। শাস্তি দিলে structure ভেঙে যায়।

### Hands-on experiment
"AI নিয়ে ৩০০ শব্দের লেখা দাও" prompt penalty 0 আর 1 দিয়ে চালাও। সবচেয়ে বেশি আসা ৫টা শব্দ গুনে তুলনা করো।

### Production implementation
Default 0 রাখো। শুধু নির্দিষ্ট creative route এ পুনরাবৃত্তির সমস্যা মেপে পেলে অল্প বাড়াও (0.2–0.5)।

### Failure cases
Technical লেখায় penalty দেওয়ায় model "database" শব্দটা এড়িয়ে "data store", "data repository" লিখতে থাকে, পাঠক বিভ্রান্ত হয়।

### Security concerns
—

### Performance concerns
প্রভাব নগণ্য।

### Cost concerns
পুনরাবৃত্তি কমলে output একটু ছোট হতে পারে।

### What did I learn?
- Penalty হলো মোটা দাগের হাতিয়ার। Structured output এ এটা ক্ষতি করে।

### What can I explain to someone else?
> "Frequency penalty হলো বিতর্ক প্রতিযোগিতার নিয়ম: একই যুক্তি যতবার বলবে, তত নম্বর কাটা যাবে।"

### GitHub Evidence
- [ ] Penalty 0 vs 1: word frequency comparison

### LinkedIn Post
- [ ] —

---

<a id="presence-penalty"></a>

## 5. Presence Penalty

### What is it?
যে token output এ **একবারও** এসেছে, তাকে একটা নির্দিষ্ট শাস্তি দেওয়া হয়, কতবার এসেছে তাতে কিছু যায় আসে না।

### Why does it exist?
Model কে নতুন শব্দ আর নতুন বিষয়ে যেতে উৎসাহ দিতে।

### What problem does it solve?
একই বিষয়ে আটকে না থেকে topic এর বৈচিত্র্য বাড়ায়।

### How does it work?
উপরের সূত্রের দ্বিতীয় অংশ: `count > 0` হলেই একবার `presence_penalty` কাটা যায়। "নীল" ১ বার এলেও যতটা শাস্তি, ১০ বার এলেও ততটা।

| | Frequency penalty | Presence penalty |
|---|---|---|
| শাস্তি কখন | প্রতিবার পুনরাবৃত্তিতে বাড়ে | একবার এলেই, একবারই |
| প্রভাব | একই শব্দের পুনরাবৃত্তি কমায় | নতুন শব্দ/বিষয়ে ঠেলে দেয় |
| উদাহরণ কাজ | লম্বা লেখায় একঘেয়েমি কমানো | Brainstorming এ ভিন্ন ভিন্ন idea |

### Important concepts
- **OpenAI তে range −2 থেকে 2, default 0।**
- Frequency penalty এর মতো Anthropic API তে নেই।

### Trade-offs
বেশি বৈচিত্র্য, কিন্তু মূল বিষয় থেকে সরে যাওয়ার ঝুঁকি।

### Alternatives
Prompt এ সরাসরি বলা: "প্রতিটা idea আলাদা category থেকে দাও।" প্রায়ই এটাই বেশি কাজের।

### When should I use it?
Brainstorming, নাম/slogan এর তালিকা যেখানে প্রতিটা আলাদা হওয়া দরকার।

### When should I NOT use it?
Summary, প্রশ্নোত্তর, structured output, যেখানে মূল বিষয়ে থাকা জরুরি।

### Hands-on experiment
"১০টা startup idea দাও" presence penalty 0 আর 1 দিয়ে চালাও। কয়টা আলাদা industry এলো গুনে দেখো।

### Production implementation
Default 0। প্রয়োজনে শুধু ideation feature এ।

### Failure cases
বেশি penalty তে summary এ মূল keyword বাদ পড়ে যায়।

### Security concerns
—

### Performance concerns
প্রভাব নগণ্য।

### Cost concerns
সরাসরি নেই।

### What did I learn?
- Frequency = "কতবার", Presence = "এসেছে কি না"।

### What can I explain to someone else?
> "Presence penalty হলো আড্ডায় 'এই বিষয়ে তো বলা হয়েছে, নতুন কিছু বলো' নিয়ম।"

### GitHub Evidence
- [ ] —

### LinkedIn Post
- [ ] —

---

<a id="determinism"></a>

## 6. Determinism

### What is it?
একই input দিলে প্রতিবার হুবহু একই output পাওয়া। LLM স্বভাবতই **non-deterministic**।

### Why does it exist?
Sampling এ random draw থাকে (ধাপ ⑤)। আর temperature 0 তেও hardware ও server এর কারণে ছোট পার্থক্য আসে।

### What problem does it solve?
Testing, debugging, audit আর reproducibility এর জন্য একই output পাওয়া জরুরি।

### How does it work?
Determinism এর কাছাকাছি যাওয়ার উপায়:
- **Temperature 0 (greedy):** random draw কার্যত বন্ধ।
- **Seed:** random number generator এর শুরুর মান ঠিক করে দেওয়া। Local tool (Ollama) এ ভালো কাজ করে। OpenAI তে `seed` আছে, কিন্তু শুধু "best effort"।

**T = 0 তেও কেন পুরোপুরি একই না?**
- GPU তে floating-point হিসাবের ক্রম বদলালে ফলাফল সামান্য বদলায়।
- Server এ তোমার request অন্যদের সাথে batch হয়, batch অনুযায়ী হিসাব বদলায়।
- MoE model এ router এর সিদ্ধান্তও এতে প্রভাবিত হয়।
- Provider চুপচাপ backend বা model update করতে পারে।

দুইটা token এর probability খুব কাছাকাছি হলে এই সামান্য পার্থক্যেই অন্য token বাছা হয়, আর তারপর পুরো বাকি উত্তর আলাদা পথে চলে যায়।

### Important concepts
- **API তে ১০০% determinism এর guarantee নেই।**
- **Model version pin করো:** "latest" alias ব্যবহার করলে model বদলে গেলে output ও বদলাবে।
- **Reasoning model এ temperature নিয়ন্ত্রণই নেই**, তাই determinism আরও কঠিন।

### Trade-offs
বেশি determinism = test সহজ, কিন্তু বৈচিত্র্য শূন্য।

### Alternatives
Output এর হুবহু মিল না খুঁজে **অর্থের মিল** যাচাই করা: schema validation, LLM-as-judge, semantic similarity (Phase 9)।

### When should I use it?
Unit test, regression test, debugging, audit trail।

### When should I NOT use it?
Determinism এর উপর ভরসা করে business logic বানাবে না ("এই prompt এ সবসময় 'YES' আসবে")।

### Hands-on experiment
একই prompt T = 0 তে ২০ বার চালাও, কয়টা আলাদা output এলো গুনে দেখো। তারপর Ollama তে `seed` দিয়ে একই test করো।

### Production implementation
- Model এর exact version ব্যবহার করো।
- Prompt, model, parameter আর output log করো, যাতে পরে কারণ খোঁজা যায়।
- Test এ exact string match না করে schema বা judge দিয়ে যাচাই করো।

### Failure cases
CI তে snapshot test (exact output match) মাঝে মাঝে কোনো code change ছাড়াই fail করে (flaky test)।

### Security concerns
Audit এর ক্ষেত্রে "AI কেন এই সিদ্ধান্ত নিল" দেখাতে পুরো input/output log রাখা দরকার, কারণ পরে আবার চালালে একই উত্তর নাও আসতে পারে।

### Performance concerns
—

### Cost concerns
Flaky test বারবার চালানো মানে বাড়তি API খরচ।

### What did I learn?
- T = 0 মানে "প্রায় একই", "হুবহু একই" না। Reliability আসে validation থেকে।

### What can I explain to someone else?
> "একই রেসিপি দিয়ে দুইবার রান্না করলেও স্বাদ হুবহু এক হয় না। তাই 'স্বাদ ঠিক আছে কি না' যাচাই করো, 'হুবহু এক কি না' না।"

### GitHub Evidence
- [ ] T = 0 consistency test (20 runs)

### LinkedIn Post
- [ ] "Temperature 0 দিলেও AI কেন একই উত্তর দেয় না?"

---

<a id="sampling-experiments"></a>

## 7. Sampling Experiments

### What is it?
নিজের কাজের জন্য কোন sampling setting সবচেয়ে ভালো, তা অনুমান না করে মেপে বের করা।

### Why does it exist?
"Temperature 0.7 ভালো" এর মতো সাধারণ পরামর্শ সব কাজে খাটে না। কাজ, model আর ভাষা অনুযায়ী ফল বদলায়।

### What problem does it solve?
Guess এর বদলে data দিয়ে setting বাছাই।

### How does it work?
1. একটা কাজ আর কিছু prompt ঠিক করো।
2. একটা parameter বদলাও (বাকিগুলো স্থির রাখো)।
3. প্রতি setting এ একাধিকবার চালাও।
4. মাপো: কয়টা আলাদা output, JSON valid কতবার, quality কেমন।

### Important concepts
- **একবারে একটা variable বদলাও।**
- **একাধিকবার চালাও:** sampling random, একবারের ফল দিয়ে সিদ্ধান্ত নেওয়া যায় না।
- **Local model দিয়ে শুরু করো:** খরচ শূন্য, আর সব parameter বদলানো যায়।

### Trade-offs
বেশি run = বিশ্বাসযোগ্য ফল, কিন্তু বেশি সময় আর (API তে) বেশি খরচ।

### Alternatives
Provider এর playground এ হাতে হাতে চেষ্টা করা (দ্রুত, কিন্তু মাপা যায় না)।

### When should I use it?
নতুন feature এর setting ঠিক করার সময়, আর model বদলানোর পর।

### When should I NOT use it?
যে model এ sampling parameter বন্ধ (reasoning model), সেখানে এই experiment অর্থহীন। সেখানে prompt আর effort নিয়ে experiment করো।

### Hands-on experiment
Ollama দিয়ে local এ temperature sweep (আগে `ollama run qwen2.5:7b` দিয়ে model নামিয়ে নাও):

```ts
const MODEL = "qwen2.5:7b";
const PROMPT = "একটা coffee shop এর নাম দাও। শুধু নামটা লেখো।";
const RUNS = 10;

async function generate(temperature: number) {
  const res = await fetch("http://localhost:11434/api/generate", {
    method: "POST",
    body: JSON.stringify({
      model: MODEL,
      prompt: PROMPT,
      stream: false,
      options: { temperature },
    }),
  });
  const data = await res.json();
  return (data.response as string).trim();
}

for (const temperature of [0, 0.7, 1.5]) {
  const outputs: string[] = [];
  for (let i = 0; i < RUNS; i++) outputs.push(await generate(temperature));
  const unique = new Set(outputs).size;
  console.log(`T=${temperature} → ${unique}/${RUNS} unique`, [...new Set(outputs)]);
}
```

একই script এ `options` বদলে বাকি experiment গুলোও করা যায়: `top_k`, `top_p`, `seed`, `repeat_penalty`।

### Production implementation
Experiment এর ফল (setting, prompt, model version, result) একটা file এ রাখো। পরে model বদলালে একই experiment আবার চালিয়ে তুলনা করো।

### Failure cases
Local ছোট model এর ফল দেখে ধরে নেওয়া যে বড় API model ও একই আচরণ করবে।

### Security concerns
Experiment এ real user data ব্যবহার করবে না।

### Performance concerns
Local এ CPU তে চালালে ধীর। RUNS কম রেখে শুরু করো।

### Cost concerns
Local এ শূন্য। API তে চালালে run সংখ্যা × prompt এর খরচ আগে হিসাব করো।

### What did I learn?
- Sampling নিয়ে মতামত না, measurement দরকার।

### What can I explain to someone else?
> "রান্নার রেসিপিতে 'লবণ পরিমাণমতো' লেখা থাকে। পরিমাণটা জানতে হলে চেখে দেখতে হয়। Sampling experiment হলো সেই চেখে দেখা।"

### GitHub Evidence
- [ ] Sampling sweep script + result table (temperature, top-k, seed)

### LinkedIn Post
- [ ] —

---

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Model Families](model-families.md) · [Root README](../README.md)

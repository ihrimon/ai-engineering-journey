# Phase 1 — LLM Fundamentals: Model Families (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Model এর নাম, version, দাম আর context size খুব দ্রুত বদলায়। এখানে যে version গুলোর নাম আছে সেগুলো **উদাহরণ**, সবচেয়ে নতুন নাও হতে পারে। কাজে লাগানোর আগে সবসময় provider এর official docs দেখবে।

## Quick Cheat Sheet

| Family | Company | Weights | এক লাইনে পরিচয় |
|---|---|---|---|
| [GPT](#gpt) | OpenAI | Closed | সবচেয়ে বড় ecosystem, reasoning model এর পথপ্রদর্শক |
| [Claude](#claude) | Anthropic | Closed | Coding, agent, লম্বা document, instruction following এ শক্তিশালী |
| [Gemini](#gemini) | Google | Closed | Natively multimodal (video/audio), খুব লম্বা context, Google ecosystem |
| [Llama](#llama) | Meta | Open-weight | Self-hosting আর fine-tuning এর জন্য সবচেয়ে পরিচিত open family |
| [Mistral](#mistral) | Mistral AI (France) | Open + Closed | ছোট কিন্তু efficient model, Europe/data residency |
| [Qwen](#qwen) | Alibaba | Open-weight | অনেক size, শক্তিশালী multilingual ও coding |
| [DeepSeek](#deepseek) | DeepSeek (China) | Open-weight | MoE + RL reasoning, খুব কম দাম |

**Understand (concept গুলো):** [Architecture differences](#architecture-differences) · [Capability trade-offs](#capability-trade-offs) · [Cost trade-offs](#cost-trade-offs) · [Latency trade-offs](#latency-trade-offs) · [Context window differences](#context-window-differences) · [Open-weight vs closed](#open-weight-vs-closed)

```text
                        LLM Model Families
                               │
          ┌────────────────────┴────────────────────┐
     Closed (API only)                      Open-weight (download করা যায়)
          │                                         │
  GPT · Claude · Gemini                Llama · Qwen · DeepSeek · Mistral*
  (Mistral এর কিছু model)              (* Mistral এর একাংশ)
          │                                         │
  সহজ শুরু, সর্বোচ্চ quality,              নিজের server, data control,
  vendor এর উপর নির্ভরতা                   fine-tuning, কিন্তু infra নিজের
```

---

# Part A — Model Families

<a id="gpt"></a>

## 1. GPT (OpenAI)

### What is it?
OpenAI এর closed model family। ChatGPT এর পেছনের model। উদাহরণ: GPT-4o, GPT-5 family, আর "o-series" reasoning model (o1, o3)। সাধারণত বড়/ছোট tier থাকে (যেমন full, mini, nano)।

### Why does it exist?
ChatGPT দিয়ে LLM কে mainstream বানিয়েছে। Developer দের জন্য API এর মাধ্যমে একই model বিক্রি করে।

### What problem does it solve?
General-purpose যেকোনো ভাষার কাজ, একটা ভালো documented API আর বিশাল ecosystem সহ।

### How does it work?
Transformer-based (internal details public না)। API তে পাঠাও, response পাও। Reasoning model গুলো উত্তর দেওয়ার আগে ভেতরে "চিন্তা" করে (reasoning tokens)।

### Important concepts
- **Ecosystem effect:** বেশিরভাগ tutorial, library আর framework প্রথমে OpenAI support করে।
- **OpenAI-compatible API format:** অনেক provider (OpenRouter, Groq, vLLM) একই format copy করেছে।
- **Tokenizer:** `o200k_base` (core-concepts এ আমরা এটাই মেপেছিলাম)।
- **Tier:** একই family তে বড় model (smart, দামি) আর mini/nano (দ্রুত, সস্তা)।

### Trade-offs
Ecosystem আর tooling সবচেয়ে ভালো, কিন্তু closed: weights পাবে না, model বদলালে behavior বদলে যেতে পারে।

### Alternatives
Claude, Gemini (closed); self-host চাইলে Llama/Qwen/DeepSeek।

### When should I use it?
দ্রুত prototype, প্রচুর tutorial/library দরকার, structured output ও function calling লাগবে।

### When should I NOT use it?
Data নিজের server এর বাইরে যাওয়া নিষেধ হলে, অথবা খুব বেশি volume এ cost বেশি হয়ে গেলে।

### Hands-on experiment
একই prompt GPT এর বড় আর mini model এ চালিয়ে quality, latency ও `usage` তুলনা করো।

### Production implementation
সরাসরি SDK call ছড়িয়ে না রেখে **provider abstraction layer** বানাও (নিচে [Part B](#capability-trade-offs) এ code আছে), যাতে পরে model বদলানো সহজ হয়।

### Failure cases
Model version update হলে আগের prompt এর output বদলে যায় → eval ছাড়া ধরা পড়ে না।

### Security concerns
তোমার data OpenAI এর server এ যায়। Data retention ও usage policy পড়ে নাও।

### Performance concerns
Reasoning model এ TTFT অনেক বেশি হতে পারে।

### Cost concerns
Mini/nano tier অনেক সস্তা। সব কাজে বড় model লাগে না।

### What did I learn?
GPT এর বড় শক্তি শুধু model না, তার ecosystem।

### What can I explain to someone else?
> "GPT হলো AI দুনিয়ার 'default'। সবাই এটা দিয়ে শুরু করে কারণ সব tool এটা support করে। কিন্তু default মানেই সবসময় best না।"

### GitHub Evidence
- [ ] Model Comparison Lab এ GPT adapter

### LinkedIn Post
- [ ] —

---

<a id="claude"></a>

## 2. Claude (Anthropic)

### What is it?
Anthropic এর closed model family। Tier গুলো (অক্টোবর ২০২৬ অনুযায়ী):

| Tier | উদাহরণ model | Context | দাম (input / output per 1M) |
|---|---|---|---|
| Haiku (ছোট, দ্রুত) | Claude Haiku 4.5 | 200K | $1 / $5 |
| Sonnet (balanced) | Claude Sonnet 5.5 | 1M | $2 / $10 |
| Opus (শক্তিশালী) | Claude Opus 5.5 | 1M | $4 / $20 |
| Fable (সবচেয়ে capable) | Claude Fable 5.1 | 1M | $10 / $50 |

### Why does it exist?
Anthropic safety-focused AI company। Claude তাদের commercial product।

### What problem does it solve?
জটিল coding, লম্বা agentic কাজ, বড় document analysis, আর নির্ভরযোগ্যভাবে instruction মানা।

### How does it work?
Transformer-based। নতুন model গুলোতে **adaptive thinking**: model নিজেই ঠিক করে কতটা চিন্তা করবে, আর `effort` (low থেকে max) দিয়ে সেটা নিয়ন্ত্রণ করা যায়।

### Important concepts
- **Tier system:** Haiku → Sonnet → Opus → Fable, ছোট থেকে বড়।
- **Prompt caching:** বারবার পাঠানো context এ অনেক কম দাম (cache read)।
- **Effort control:** একই model এ কম effort দিলে কম token আর কম খরচ।
- **Token counting API:** Claude এর tokenizer public library হিসেবে নেই, তাই count করতে `count_tokens` API লাগে (`tiktoken` দিয়ে হবে না)।

### Trade-offs
Coding ও agent এ শক্তিশালী, কিন্তু closed। উপরের tier এ দাম বেশি।

### Alternatives
GPT, Gemini; open-weight চাইলে Qwen/DeepSeek।

### When should I use it?
Coding agent, লম্বা document বা codebase analysis, multi-step tool use।

### When should I NOT use it?
খুব সহজ classification এ বড় tier ব্যবহার করবে না (Haiku বা অন্য ছোট model যথেষ্ট)। Self-hosting বাধ্যতামূলক হলেও না।

### Hands-on experiment
একই কাজ Haiku আর Sonnet এ চালাও। ছোট model কোথায় যথেষ্ট আর কোথায় ব্যর্থ, তা নোট করো।

### Production implementation
Stable system prompt আগে রেখে prompt caching চালু করো। `stop_reason` সবসময় check করো।

### Failure cases
Tokenizer আলাদা হওয়ায় OpenAI এর হিসাব দিয়ে Claude এর cost estimate করলে ভুল হয়।

### Security concerns
Data Anthropic এর server এ যায়। Retention policy দেখে নাও। Tool use দিলে permission সীমিত রাখো।

### Performance concerns
Thinking/effort বাড়ালে quality বাড়ে, latency ও বাড়ে।

### Cost concerns
Prompt caching আর effort tuning হলো দুইটা সবচেয়ে বড় cost lever।

### What did I learn?
একই family তে tier আর effort বেছে নেওয়াই আসল cost-quality decision।

### What can I explain to someone else?
> "Claude এর tier গুলো একই কোম্পানির junior থেকে principal engineer এর মতো। কাজ যত কঠিন, তত সিনিয়র কাউকে দাও, কিন্তু সহজ কাজে সিনিয়র লাগিও না।"

### GitHub Evidence
- [ ] Model Comparison Lab এ Claude adapter

### LinkedIn Post
- [ ] —

---

<a id="gemini"></a>

## 3. Gemini (Google)

### What is it?
Google DeepMind এর closed model family। Tier: **Pro** (শক্তিশালী), **Flash** (দ্রুত, সস্তা), **Flash-Lite** (সবচেয়ে সস্তা)। উদাহরণ: Gemini 2.5 Pro/Flash।

### Why does it exist?
Google এর search, Android, Workspace আর Cloud এর AI ভিত্তি।

### What problem does it solve?
একই model এ text, image, **audio ও video** বোঝা, সাথে খুব লম্বা context।

### How does it work?
শুরু থেকেই **natively multimodal** হিসেবে train করা, মানে text ছাড়াও অন্য media সরাসরি input নিতে পারে।

### Important concepts
- **Native multimodal:** video/audio সরাসরি input দেওয়া যায়।
- **Long context:** ১M বা তার বেশি token এর context।
- **Access:** Google AI Studio (সহজ শুরু) আর Vertex AI (enterprise)।

### Trade-offs
Multimodal আর context এ শক্তিশালী, কিন্তু Google ecosystem এর উপর নির্ভরতা বাড়ে।

### Alternatives
GPT/Claude (vision), multimodal কাজে আলাদা specialized model (STT, OCR)।

### When should I use it?
Video/audio বিশ্লেষণ, বিশাল document একবারে, সস্তা high-volume কাজ (Flash)।

### When should I NOT use it?
Data residency বা vendor নীতির কারণে Google ব্যবহার করা না গেলে।

### Hands-on experiment
একটা ছোট video বা audio দিয়ে summary চাও, তারপর অন্য model এর সাথে quality তুলনা করো।

### Production implementation
Enterprise এ Vertex AI দিয়ে IAM, region আর quota নিয়ন্ত্রণ করো।

### Failure cases
লম্বা context থাকলেও মাঝের তথ্য হারাতে পারে (lost in the middle)।

### Security concerns
Free tier এ data training এ ব্যবহার হতে পারে। Production এ paid/enterprise terms দেখে নাও।

### Performance concerns
Flash tier খুব দ্রুত। Pro আর thinking mode ধীর।

### Cost concerns
Flash/Flash-Lite high-volume কাজে খুব সাশ্রয়ী।

### What did I learn?
Multimodal বা খুব লম্বা context এর কাজে Gemini প্রথমে বিবেচনা করার মতো।

### What can I explain to someone else?
> "Gemini এমন একজন, যে লেখা পড়ার পাশাপাশি ভিডিও দেখতে আর অডিও শুনতেও পারে, আর একবারে অনেক বড় বই মাথায় রাখতে পারে।"

### GitHub Evidence
- [ ] Model Comparison Lab এ Gemini adapter

### LinkedIn Post
- [ ] —

---

<a id="llama"></a>

## 4. Llama (Meta)

### What is it?
Meta এর **open-weight** model family। Model এর weights download করে নিজের computer বা server এ চালানো যায়। উদাহরণ: Llama 3.x (8B, 70B, 405B), Llama 4 (MoE architecture)।

### Why does it exist?
Meta open-weight model ছেড়ে নিজেকে AI ecosystem এর কেন্দ্রে রাখতে চায়, আর community এটার উপর build করে।

### What problem does it solve?
নিজের infra তে model চালানো, data বাইরে না পাঠানো, আর নিজের data দিয়ে fine-tune করা।

### How does it work?
Weights download → inference engine (vLLM, Ollama, llama.cpp) দিয়ে চালাও। অথবা Groq, Together, AWS Bedrock এর মতো hosted provider থেকে API হিসেবে নাও।

### Important concepts
- **Parameter size (8B, 70B…):** B = billion parameter। বড় size মানে সাধারণত বেশি smart, কিন্তু বেশি GPU memory লাগে।
- **License:** "Llama Community License"। Open-weight, কিন্তু পুরোপুরি open source (OSI) না, কিছু শর্ত আছে।
- **Fine-tuning:** LoRA/QLoRA দিয়ে নিজের task এ বিশেষায়িত করা যায় (Python Track E)।

### Trade-offs
পূর্ণ নিয়ন্ত্রণ পাও, কিন্তু GPU, scaling, monitoring সব নিজের দায়িত্ব।

### Alternatives
Qwen, DeepSeek, Mistral (open-weight)।

### When should I use it?
Privacy বা compliance, offline/on-premise deployment, fine-tuning, বা বিশাল volume এ API cost কমানো।

### When should I NOT use it?
ছোট team এ GPU ops সামলানোর লোক নেই, আর frontier-level quality দরকার।

### Hands-on experiment
`ollama run llama3.2` দিয়ে নিজের laptop এ ছোট model চালাও। Token/sec আর quality নোট করো।

### Production implementation
vLLM দিয়ে serve করো, quantization (Phase 7) দিয়ে memory কমাও।

### Failure cases
ছোট model জটিল reasoning বা structured output এ বেশি ভুল করে।

### Security concerns
Data নিজের কাছে থাকে (সুবিধা)। তবে নিজের server এর security আর model এর safety tuning তোমার দায়িত্ব।

### Performance concerns
নিজের GPU এর ক্ষমতাই latency ঠিক করে।

### Cost concerns
Per-token দাম নেই, কিন্তু GPU (সবসময় চালু থাকলে) এর fixed খরচ আছে। কম volume এ API প্রায়ই সস্তা পড়ে।

### What did I learn?
Open-weight মানে স্বাধীনতা, সাথে ops এর দায়িত্বও।

### What can I explain to someone else?
> "Closed model হলো রেস্টুরেন্টে খাওয়া, Llama হলো রেসিপি নিয়ে নিজে রান্না করা। স্বাদ নিজের মতো বদলাতে পারো, কিন্তু রান্নাঘর তোমার।"

### GitHub Evidence
- [ ] Local Llama via Ollama + benchmark

### LinkedIn Post
- [ ] —

---

<a id="mistral"></a>

## 5. Mistral (Mistral AI)

### What is it?
France এর Mistral AI এর model family। কিছু model **open-weight** (যেমন Mistral 7B, Mixtral, Apache 2.0 license), কিছু **commercial/closed** (যেমন Mistral Large)। Codestral নামে coding model ও আছে।

### Why does it exist?
Europe এর নিজস্ব শক্তিশালী AI provider, আর efficient ছোট model এর উপর ফোকাস।

### What problem does it solve?
কম resource এ ভালো performance, আর EU data residency বা sovereignty দরকার এমন কোম্পানির চাহিদা।

### How does it work?
Mixtral জনপ্রিয় করেছে **Mixture of Experts (MoE)**: প্রতিটা token এ পুরো model না, শুধু কয়েকটা "expert" অংশ কাজ করে, তাই efficient।

### Important concepts
- **Apache 2.0 model:** commercial ব্যবহারে প্রায় কোনো বাধা নেই।
- **MoE:** মোট parameter বেশি, কিন্তু প্রতি token এ active parameter কম।
- **La Plateforme:** Mistral এর নিজস্ব API।

### Trade-offs
Efficient আর flexible license, কিন্তু frontier closed model এর তুলনায় top quality কিছুটা কম হতে পারে।

### Alternatives
Llama, Qwen (open); GPT/Claude (closed)।

### When should I use it?
EU compliance, cost-sensitive production, ছোট model self-host।

### When should I NOT use it?
সবচেয়ে কঠিন reasoning বা agentic কাজে, যেখানে frontier model লাগে।

### Hands-on experiment
Mistral 7B আর Llama 8B একই local setup এ চালিয়ে তুলনা করো।

### Production implementation
API বা self-host দুইটাই সম্ভব। Provider abstraction এর মাধ্যমে রাখো।

### Failure cases
কোন model কোন license এর তা না দেখে ব্যবহার করলে legal ঝামেলা হতে পারে।

### Security concerns
Self-host করলে data নিজের কাছে থাকে।

### Performance concerns
ছোট আর MoE model দ্রুত।

### Cost concerns
ছোট model এর API দাম কম, self-host ও সস্তা।

### What did I learn?
"Open" model এর মধ্যেও license আলাদা। প্রতিটা model এর license আলাদা করে পড়তে হয়।

### What can I explain to someone else?
> "Mistral হলো ছোট কিন্তু দক্ষ টিম। সব কাজে বিশাল কোম্পানি লাগে না।"

### GitHub Evidence
- [ ] —

### LinkedIn Post
- [ ] —

---

<a id="qwen"></a>

## 6. Qwen (Alibaba)

### What is it?
Alibaba এর model family, বেশিরভাগ **open-weight** (অনেকগুলো Apache 2.0)। খুব ছোট (০.৫B) থেকে বিশাল পর্যন্ত অনেক size। বিশেষ variant: Qwen-Coder (coding), Qwen-VL (vision)। উদাহরণ: Qwen2.5, Qwen3।

### Why does it exist?
চীনের শীর্ষ AI lab গুলোর একটা, global open-weight ecosystem এ শক্ত অবস্থান নিয়েছে।

### What problem does it solve?
প্রায় সব size এ ভালো open model, শক্তিশালী multilingual আর coding ক্ষমতা।

### How does it work?
Dense আর MoE দুই ধরনের model আছে। Open-weight হওয়ায় Ollama/vLLM দিয়ে চালানো যায়।

### Important concepts
- **Size ladder:** device এ চালানো যায় এমন ছোট model থেকে server-grade বড় model।
- **Multilingual:** অনেক এশীয় ভাষা সহ শক্তিশালী। বাংলায় কেমন, তা নিজে মেপে দেখার মতো।
- **Fine-tuning base:** community অনেক fine-tune এর base হিসেবে Qwen ব্যবহার করে।

### Trade-offs
শক্তিশালী আর flexible, কিন্তু কিছু প্রতিষ্ঠানে চীনা model ব্যবহারে policy বাধা থাকতে পারে।

### Alternatives
Llama, DeepSeek, Mistral।

### When should I use it?
Self-host, coding assistant, multilingual app, ছোট device এ model।

### When should I NOT use it?
Organization এর vendor/country policy এ অনুমোদিত না হলে।

### Hands-on experiment
Qwen আর Llama এর ছোট model দিয়ে একই বাংলা প্রশ্নের উত্তর আর token count তুলনা করো (core-concepts এর experiment এর পরের ধাপ)।

### Production implementation
vLLM দিয়ে serve করো। Coding কাজে Qwen-Coder variant দেখো।

### Failure cases
ছোট size এ hallucination বেশি।

### Security concerns
Self-host করলে weights থেকে data বাইরে যায় না। Hosted API হলে provider এর অবস্থান ও policy দেখো।

### Performance concerns
ছোট model খুব দ্রুত, বড় model এ ভালো GPU লাগে।

### Cost concerns
Open-weight, তাই খরচ শুধু infra।

### What did I learn?
বাংলা app এর জন্য open-weight model এর মধ্যেও তুলনা করা দরকার।

### What can I explain to someone else?
> "Qwen হলো সব সাইজের জামা পাওয়া যায় এমন দোকান, ছোট phone থেকে বড় server, সবার জন্য একটা মাপ আছে।"

### GitHub Evidence
- [ ] বাংলা prompt এ Qwen vs Llama comparison

### LinkedIn Post
- [ ] —

---

<a id="deepseek"></a>

## 7. DeepSeek

### What is it?
চীনের DeepSeek এর model family, **open-weight (MIT license)**। উদাহরণ: DeepSeek-V3 (general), DeepSeek-R1 (reasoning)।

### Why does it exist?
খুব কম training cost এ frontier-এর কাছাকাছি performance দেখিয়ে ২০২৫ সালে AI industry কে চমকে দিয়েছিল।

### What problem does it solve?
উচ্চমানের reasoning খুব কম দামে, আর open-weight হিসেবে।

### How does it work?
- **MoE architecture:** বিশাল মোট parameter, কিন্তু প্রতি token এ অল্প অংশ active।
- **R1:** Reinforcement Learning (RL) দিয়ে reasoning শেখানো। উত্তরের আগে লম্বা চিন্তা করে।

### Important concepts
- **MIT license:** খুবই permissive।
- **Distilled models:** বড় R1 এর জ্ঞান ছোট model (Qwen/Llama base) এ "distill" করা, তাই ছোট hardware এও চলে।
- **API বনাম self-host:** অফিশিয়াল API এর data চীনের server এ যায়, কিন্তু weights নিয়ে নিজে চালালে যায় না।

### Trade-offs
দাম আর openness অসাধারণ, কিন্তু অফিশিয়াল API ব্যবহারে data residency আর compliance প্রশ্ন আছে।

### Alternatives
o-series (GPT), Claude/Gemini thinking mode, Qwen reasoning model।

### When should I use it?
Reasoning-heavy কাজ কম দামে, অথবা self-hosted reasoning model দরকার হলে।

### When should I NOT use it?
Sensitive data অফিশিয়াল API তে পাঠানো যাবে না এমন অবস্থায় (তখন self-host বা অন্য hosted provider)।

### Hands-on experiment
একটা ছোট distilled R1 model local এ চালিয়ে তার "thinking" অংশ পড়ে দেখো, model কীভাবে সমস্যা ভাঙে।

### Production implementation
Bedrock বা Together এর মতো trusted hosted provider, অথবা নিজের vLLM।

### Failure cases
Reasoning model সহজ প্রশ্নেও অনেক বেশি চিন্তা করে → অপ্রয়োজনীয় latency আর token খরচ।

### Security concerns
Data residency; model এর ভেতরে থাকা bias/censorship নিয়ে সচেতন থাকো।

### Performance concerns
Reasoning token বেশি, তাই উত্তর আসতে সময় লাগে।

### Cost concerns
Per-token দাম খুব কম, কিন্তু reasoning token যোগ করে মোট খরচ হিসাব করো।

### What did I learn?
Architecture (MoE) আর training পদ্ধতি (RL) দিয়ে খরচ নাটকীয়ভাবে কমানো সম্ভব।

### What can I explain to someone else?
> "DeepSeek দেখিয়েছে, বুদ্ধিমান model বানাতে সবসময় সবচেয়ে বেশি টাকা লাগে না, বুদ্ধিমান design লাগে।"

### GitHub Evidence
- [ ] Local distilled R1 experiment

### LinkedIn Post
- [ ] —

---

# Part B — Understand

<a id="architecture-differences"></a>

## 8. Model Architecture Differences

### What is it?
প্রায় সব আধুনিক LLM **Transformer** ভিত্তিক, কিন্তু ভেতরের design এ পার্থক্য আছে:
- **Dense vs MoE:** Dense এ প্রতি token এ পুরো model কাজ করে। MoE তে শুধু কিছু expert কাজ করে।
- **Tokenizer:** প্রতিটা family এর আলাদা (core-concepts এ দেখেছি)।
- **Native multimodal vs text-only:** কিছু model শুরু থেকেই image/audio বোঝে।
- **Standard vs reasoning:** কিছু model উত্তরের আগে আলাদা "thinking" ধাপ চালায়।

### Why does it exist?
প্রতিটা lab আলাদা লক্ষ্যে optimize করে: quality, speed, cost, বা multimodality।

### What problem does it solve?
Architecture বুঝলে বোঝা যায় কোন model কেন দ্রুত, সস্তা বা কোন কাজে ভালো।

### How does it work?
```text
Dense:  token ──► [ পুরো 70B parameter কাজ করে ] ──► output
MoE:    token ──► Router ──► [Expert 3] + [Expert 7] ──► output
                    (মোট অনেক বড়, কিন্তু প্রতি token এ অল্প অংশ active)
```

### Important concepts
- **Total vs active parameters (MoE):** memory লাগে total এর জন্য, speed নির্ভর করে active এর উপর।
- Closed model এর architecture সাধারণত public না।

### Trade-offs
MoE দ্রুত আর সস্তা inference দেয়, কিন্তু পুরো model memory তে রাখতে হয় (বেশি VRAM)।

### Alternatives
— (এটা নিজেই একটা comparison concept)

### When should I use it?
Self-host করার সময় hardware plan করতে, আর latency/cost আচরণ বুঝতে।

### When should I NOT use it?
API ব্যবহার করলে architecture নিয়ে বেশি ভাবার দরকার নেই। Output, cost আর latency মাপাই যথেষ্ট।

### Hands-on experiment
Ollama তে একটা dense আর একটা MoE model চালিয়ে memory usage আর token/sec তুলনা করো।

### Production implementation
Self-host এ GPU sizing করো total parameter আর quantization ধরে (Phase 7)।

### Failure cases
MoE model এর active parameter দেখে ছোট GPU কিনলে পুরো model memory তে ধরবে না।

### Security concerns
—

### Performance concerns
Architecture সরাসরি TPS আর memory ঠিক করে।

### Cost concerns
MoE সাধারণত per-token এ সস্তা।

### What did I learn?
"৭০০B parameter" শুনে ধীর ভাবা ভুল হতে পারে। MoE তে active অংশ অনেক ছোট।

### What can I explain to someone else?
> "Dense model এ প্রতিটা প্রশ্নে পুরো অফিস কাজ করে। MoE তে receptionist (router) শুধু সংশ্লিষ্ট ২-৩ জন expert কে ডাকে।"

### GitHub Evidence
- [ ] Dense vs MoE local benchmark

### LinkedIn Post
- [ ] —

---

<a id="capability-trade-offs"></a>

## 9. Capability Trade-offs

### What is it?
কোনো একটা model সব কাজে সেরা না। একটায় coding ভালো, অন্যটায় multimodal, আরেকটায় multilingual বা সস্তা।

### Why does it exist?
Training data, size আর optimization লক্ষ্য আলাদা।

### What problem does it solve?
বোঝায় কেন "best model" বলে কিছু নেই, আছে "এই কাজের জন্য best model"।

### How does it work?
নিজের task অনুযায়ী capability matrix বানাও:

| কাজ | প্রথমে কোথায় দেখবো (শুরুর অনুমান, eval দিয়ে যাচাই করবে) |
|---|---|
| জটিল coding / agent | Claude, GPT |
| Video/audio বিশ্লেষণ | Gemini |
| Self-host / privacy | Llama, Qwen, DeepSeek, Mistral |
| সস্তা reasoning | DeepSeek, ছোট reasoning model |
| High-volume সহজ কাজ | Haiku, Flash, mini, ছোট open model |

### Important concepts
- **Benchmark ≠ তোমার task।** নিজের eval set সবচেয়ে বিশ্বাসযোগ্য।
- **Provider abstraction:** model বদলানো যেন এক লাইনের config change হয়।

### Trade-offs
বেশি model ব্যবহার = বেশি flexibility, কিন্তু বেশি complexity আর testing।

### Alternatives
একটা model এর সাথে effort/tier বদলানো (অনেক সময় যথেষ্ট)।

### When should I use it?
Model selection এর সময় আর **LLM Model Comparison Lab** project এ।

### When should I NOT use it?
শুরুতেই ৫টা provider integrate করবে না। একটা দিয়ে শুরু করো, abstraction রাখো।

### Hands-on experiment
Provider-neutral একটা harness বানাও:
```ts
interface ModelProvider {
  name: string;
  generate(prompt: string): Promise<{ text: string; inputTokens: number; outputTokens: number; ms: number }>;
}

async function compare(providers: ModelProvider[], prompts: string[]) {
  for (const p of providers) {
    for (const prompt of prompts) {
      const r = await p.generate(prompt);
      console.table({ model: p.name, ms: r.ms, in: r.inputTokens, out: r.outputTokens });
    }
  }
}
```
প্রতিটা provider এর জন্য তার official SDK দিয়ে আলাদা adapter লিখবে।

### Production implementation
Model নাম আর provider config/feature flag এ রাখো, code এ hardcode না।

### Failure cases
এক model এর জন্য লেখা prompt অন্য model এ একইভাবে কাজ না-ও করতে পারে।

### Security concerns
প্রতিটা নতুন provider = নতুন data processor, তাই privacy review লাগবে।

### Performance concerns
—

### Cost concerns
Capability অনুযায়ী routing (সহজ কাজ ছোট model এ) বড় সাশ্রয় দেয় (Phase 10)।

### What did I learn?
Model selection হলো eval-driven engineering decision, ব্যক্তিগত পছন্দ না।

### What can I explain to someone else?
> "Cricket team এ সবাই opener না। কাজ অনুযায়ী player নামাও।"

### GitHub Evidence
- [ ] LLM Model Comparison Lab harness

### LinkedIn Post
- [ ] —

---

<a id="cost-trade-offs"></a>

## 10. Cost Trade-offs

### What is it?
বিভিন্ন family ও tier এর per-token দাম অনেক আলাদা। একই family এর মধ্যেও সবচেয়ে ছোট আর সবচেয়ে বড় tier এ প্রায় **দশ গুণ** পার্থক্য (উপরের Claude table: $1 বনাম $10 per 1M input)।

### Why does it exist?
বড় model চালাতে বেশি GPU লাগে।

### What problem does it solve?
বোঝায় কোথায় টাকা যায় আর কোথায় বাঁচানো যায়।

### How does it work?
মোট খরচ = per-token দাম × token সংখ্যা। Token সংখ্যা নির্ভর করে **tokenizer**, **reasoning token** আর **output length** এর উপর।

### Important concepts
- **Per-token price ≠ per-task cost।** সস্তা model বেশি retry লাগালে মোট খরচ বেশি।
- **Tokenizer পার্থক্য:** একই text বিভিন্ন model এ আলাদা সংখ্যক token।
- **Self-host:** per-token দাম নেই, কিন্তু GPU এর fixed খরচ।

### Trade-offs
সস্তা model vs quality; API (variable cost) vs self-host (fixed cost)।

### Alternatives
Caching, batching, routing, effort কমানো।

### When should I use it?
Feature এর unit economics হিসাব করার সময়।

### When should I NOT use it?
শুধু দাম দেখে model বাছবে না। Quality bar আগে ঠিক করো।

### Hands-on experiment
১০টা real task এ ৩টা model চালিয়ে **cost per successful task** বের করো।

### Production implementation
Usage log এ model, tokens আর cost রাখো। মাসিক রিপোর্ট বানাও।

### Failure cases
Reasoning model এর hidden thinking token হিসাবে না ধরায় বাজেট ভুল হওয়া।

### Security concerns
—

### Performance concerns
সস্তা ছোট model প্রায়ই দ্রুতও হয়।

### Cost concerns
এটাই topic: **measure first, optimize second।**

### What did I learn?
Cost তুলনা করতে হয় একই কাজের পুরো খরচে, শুধু price table দেখে না।

### What can I explain to someone else?
> "সস্তা জুতা দুই মাসে ছিঁড়ে গেলে দামি জুতার চেয়ে বেশি খরচ পড়ে।"

### GitHub Evidence
- [ ] Cost per task report

### LinkedIn Post
- [ ] —

---

<a id="latency-trade-offs"></a>

## 11. Latency Trade-offs

### What is it?
Model ভেদে TTFT আর tokens/sec আলাদা। বড় model আর reasoning model সাধারণত ধীর।

### Why does it exist?
Model size, architecture, provider এর hardware আর server load।

### What problem does it solve?
UX অনুযায়ী (chat, voice, batch) সঠিক model বাছতে সাহায্য করে।

### How does it work?
একই model আলাদা provider এ চালালেও latency আলাদা হয় (যেমন Groq/Cerebras এর মতো specialized hardware খুব দ্রুত)।

### Important concepts
- **Model choice আর provider choice দুটোই latency ঠিক করে।**
- Reasoning/effort বাড়ালে TTFT বাড়ে।

### Trade-offs
দ্রুত = সাধারণত ছোট model বা দামি infra।

### Alternatives
Streaming, caching, ছোট model, প্রয়োজন হলে fast mode।

### When should I use it?
Voice AI, real-time chat, autocomplete এর মতো latency-sensitive feature এ।

### When should I NOT use it?
Overnight batch job এ latency নিয়ে বেশি ভাবার দরকার নেই।

### Hands-on experiment
৩টা model এর TTFT আর TPS মাপো (core-concepts এর latency script ব্যবহার করে)।

### Production implementation
Route অনুযায়ী model: real-time route এ দ্রুত model, background এ শক্তিশালী model।

### Failure cases
Reasoning model user-facing chat এ দিয়ে UX নষ্ট হওয়া।

### Security concerns
—

### Performance concerns
p95 latency মাপো, average দেখে সিদ্ধান্ত নেবে না।

### Cost concerns
Fast mode বা specialized hardware প্রায়ই বেশি দামি।

### What did I learn?
Latency শুধু model এর বিষয় না, provider আর configuration এরও।

### What can I explain to someone else?
> "Formula 1 গাড়ি দ্রুত, কিন্তু বাজার করতে ওটা লাগে না।"

### GitHub Evidence
- [ ] Latency comparison table

### LinkedIn Post
- [ ] —

---

<a id="context-window-differences"></a>

## 12. Context Window Differences

### What is it?
Model ভেদে context window আলাদা। কিছু model ১২৮K-২০০K, অনেক নতুন frontier model ১M বা তার বেশি। Max output token এর limit ও আলাদা।

### Why does it exist?
Training আর inference memory/compute এর সীমা।

### What problem does it solve?
জানা থাকলে বোঝা যায় একবারে কতটা document পাঠানো যাবে, নাকি RAG লাগবে।

### How does it work?
Advertised context ≠ effective context। লম্বা context এ quality কমতে পারে (core-concepts এর lost-in-the-middle দেখো)।

### Important concepts
- **Context window আর max output আলাদা limit।**
- **Tokenizer পার্থক্য:** "১M token" মানে বিভিন্ন model এ আলাদা পরিমাণ text, বিশেষ করে বাংলায়।

### Trade-offs
বড় context = সুবিধা, কিন্তু বেশি cost আর latency।

### Alternatives
RAG, summarization, chunking (Phase 3–4)।

### When should I use it?
বড় document, codebase বা লম্বা conversation এর কাজে model বাছার সময়।

### When should I NOT use it?
"বড় context আছে" বলে সব data প্রতিবার পাঠাবে না।

### Hands-on experiment
একই needle-in-a-haystack test ২টা model এ চালিয়ে তুলনা করো।

### Production implementation
Model এর limit config এ রাখো। Request পাঠানোর আগে token count করে check করো।

### Failure cases
এক model এর limit ধরে লেখা code অন্য model এ গেলে overflow error দেয়।

### Security concerns
বড় context = বেশি external content = বেশি prompt injection এর সুযোগ।

### Performance concerns
Input লম্বা হলে TTFT বাড়ে।

### Cost concerns
বড় context ভরলে প্রতিটা request দামি হয়। Caching কাজে লাগে।

### What did I learn?
Context size model বাছার একটা মাপকাঠি মাত্র, একমাত্র না।

### What can I explain to someone else?
> "বড় টেবিল থাকলেই সব কাগজ ছড়িয়ে রাখা ঠিক না। দরকারি কাগজটাই সামনে রাখো।"

### GitHub Evidence
- [ ] Cross-model needle-in-a-haystack

### LinkedIn Post
- [ ] —

---

<a id="open-weight-vs-closed"></a>

## 13. Open-weight vs Closed Models

### What is it?
- **Closed:** শুধু API দিয়ে ব্যবহার করা যায়। Weights দেখা বা নামানো যায় না (GPT, Claude, Gemini)।
- **Open-weight:** Weights download করে নিজে চালানো যায় (Llama, Qwen, DeepSeek, Mistral এর কিছু model)।
- ⚠️ **Open-weight ≠ open source।** সাধারণত training data আর code দেওয়া হয় না, আর license এ শর্ত থাকতে পারে।

### Why does it exist?
Business model আলাদা: closed lab API বিক্রি করে, open lab ecosystem ও community তৈরি করে।

### What problem does it solve?
Control, privacy, cost আর customization এর মধ্যে বেছে নেওয়ার সুযোগ দেয়।

### How does it work?
| বিষয় | Closed | Open-weight |
|---|---|---|
| শুরু করা | খুব সহজ (API key) | Infra লাগে (বা hosted provider) |
| Quality | সাধারণত সর্বোচ্চ | দ্রুত কাছাকাছি আসছে |
| Data control | Provider এর কাছে যায় | নিজের কাছে থাকে |
| Fine-tuning | সীমিত | পূর্ণ |
| Cost model | Per-token | GPU (fixed) বা hosted per-token |
| Vendor lock-in | বেশি | কম |
| Ops দায়িত্ব | Provider এর | তোমার |

### Important concepts
- **License পড়া বাধ্যতামূলক:** Apache 2.0, MIT, Llama Community License, সবগুলো আলাদা।
- **Hybrid approach:** সাধারণ কাজে closed API, sensitive data তে self-hosted open model।
- Open-weight model ও Bedrock/Together/Groq এর মতো hosted API তে পাওয়া যায়, নিজে GPU চালাতে হয় না।

### Trade-offs
Closed = সুবিধা ও quality, কিন্তু নির্ভরতা। Open = নিয়ন্ত্রণ, কিন্তু দায়িত্ব।

### Alternatives
Hybrid architecture, hosted open-weight provider।

### When should I use it?
- **Closed:** দ্রুত prototype, সর্বোচ্চ quality, ছোট team।
- **Open-weight:** privacy/compliance, fine-tuning, offline, বিশাল volume।

### When should I NOT use it?
GPU ops এর অভিজ্ঞতা ছাড়া শুধু "ফ্রি" ভেবে open-weight self-host করবে না।

### Hands-on experiment
একই task closed API আর local open model এ চালিয়ে quality, latency আর খরচের তালিকা বানাও।

### Production implementation
Provider abstraction রাখো যাতে closed থেকে open (বা উল্টো) যাওয়া সহজ হয়।

### Failure cases
License না পড়ে commercial product এ model ব্যবহার করা।

### Security concerns
Closed: data provider এর কাছে যায়। Open: নিজের infra এর security নিজের দায়িত্ব।

### Performance concerns
Self-host এ performance তোমার hardware এর উপর নির্ভর করে।

### Cost concerns
কম volume এ API সস্তা, অনেক বেশি volume এ self-host সস্তা হতে পারে। Break-even হিসাব করো।

### What did I learn?
Open vs closed কোনো ধর্মযুদ্ধ না, use case অনুযায়ী engineering decision।

### What can I explain to someone else?
> "Closed model হলো ভাড়া বাসা: ঝামেলা কম, নিয়ন্ত্রণ কম। Open-weight হলো নিজের বাড়ি: স্বাধীনতা বেশি, মেরামতও নিজের।"

### GitHub Evidence
- [ ] Closed vs open-weight comparison report

### LinkedIn Post
- [ ] "Open-weight মানেই open source না — কেন?"

---

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Core Concepts](core-concepts.md) · [Root README](../README.md)

# Phase 1 — LLM Fundamentals: Reasoning (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Reasoning এর parameter আর নাম provider ভেদে আলাদা আর দ্রুত বদলায় (যেমন Anthropic এ `effort` আর adaptive thinking, OpenAI তে `reasoning_effort`)। ব্যবহারের আগে docs দেখবে।
>
> 🧪 Hands-on এ Ollama এর open reasoning model (DeepSeek-R1 এর ছোট distilled version) ব্যবহার করা হয়েছে, যেখানে model এর "চিন্তা" নিজের চোখে দেখা যায়। খরচ শূন্য।
>
> 🏁 এটা Phase 1 এর শেষ chapter। এখানে আগের প্রায় সব chapter এর ধারণা একসাথে আসে: token cost, output token, latency, sampling, tool use।

## Quick Cheat Sheet

| Topic | এক লাইনে |
|---|---|
| [Standard models](#standard-models) | প্রশ্ন পেয়েই সরাসরি উত্তর লেখা শুরু করে |
| [Reasoning models](#reasoning-models) | উত্তর লেখার আগে ভেতরে ধাপে ধাপে "চিন্তা" করে, তারপর উত্তর দেয় |
| [Extended thinking](#extended-thinking) | Model কতটা চিন্তা করবে, সেটা নিয়ন্ত্রণ করার ব্যবস্থা (budget, effort, adaptive) |
| [Reasoning token costs](#reasoning-token-costs) | চিন্তার token ও **output token** হিসেবে bill হয়, দেখা না গেলেও |
| [Reasoning vs latency](#reasoning-vs-latency) | বেশি চিন্তা = উত্তর আসতে বেশি সময় |
| [When reasoning models are useful](#when-reasoning-models-are-useful) | কঠিন, বহু ধাপের সমস্যায় হ্যাঁ; সহজ, দ্রুত কাজে না |

```text
Standard model:
  প্রশ্ন ──► [উত্তর লেখা শুরু] ──► উত্তর

Reasoning model:
  প্রশ্ন ──► [চিন্তা: সমস্যা ভাঙা → চেষ্টা → ভুল ধরা → আবার চেষ্টা] ──► [উত্তর লেখা] ──► উত্তর
              └──────── reasoning / thinking tokens ────────┘
                  (output হিসেবে bill হয়, latency বাড়ায়)
```

> **Core Concepts এর সাথে যোগ:** Model একবারে একটাই token লেখে। Reasoning model আসলে উত্তরের আগে অনেকগুলো "চিন্তার token" লেখে। তাই reasoning = **বেশি output token** = বেশি খরচ আর বেশি সময়, বিনিময়ে কঠিন প্রশ্নে বেশি নির্ভুলতা।

---

<a id="standard-models"></a>

## 1. Standard Models

### What is it?
যে model প্রশ্ন পেয়েই সরাসরি উত্তরের token লেখা শুরু করে, আলাদা কোনো চিন্তার ধাপ ছাড়া। আগের সব chapter এ আমরা মূলত এই ধরনের model নিয়েই কথা বলেছি।

### Why does it exist?
বেশিরভাগ কাজে (summary, extraction, chat, অনুবাদ) আলাদা করে চিন্তা করার দরকার নেই। সরাসরি উত্তরই দ্রুত আর সস্তা।

### What problem does it solve?
দ্রুত, সস্তা আর অনুমানযোগ্য উত্তর।

### How does it work?
প্রতিটা token লেখার সময় model আগের লেখা দেখে পরেরটা predict করে। উত্তরের প্রথম শব্দ লেখার সময়ই সে "সিদ্ধান্ত" নিয়ে ফেলে, পরে ফিরে গিয়ে ঠিক করার সুযোগ কম।

### Important concepts
- **Chain-of-thought prompting:** Standard model কেও prompt এ "উত্তর দেওয়ার আগে ধাপে ধাপে ভাবো" বললে সে চিন্তাটা লেখে, আর কঠিন প্রশ্নে ভালো করে। Reasoning model এর ধারণা এখান থেকেই এসেছে (Phase 2 এ বিস্তারিত)।
- **Fast thinking:** মানুষের "তাৎক্ষণিক উত্তর" এর মতো, পরিচিত প্রশ্নে দারুণ, জটিল হিসাবে ভুলের ঝুঁকি।

### Trade-offs
দ্রুত আর সস্তা, কিন্তু বহু ধাপের যুক্তি, জটিল হিসাব আর পরিকল্পনায় দুর্বল।

### Alternatives
Reasoning model; অথবা standard model + chain-of-thought prompt; অথবা tool (calculator, code) দিয়ে কঠিন অংশ সামলানো।

### When should I use it?
Classification, extraction, summary, সাধারণ chat, অনুবাদ, high-volume কাজ।

### When should I NOT use it?
জটিল math, বহু ধাপের logic, কঠিন debugging, যেখানে প্রথম চেষ্টায় ভুলের সম্ভাবনা বেশি।

### Hands-on experiment
একটা কঠিন যুক্তির ধাঁধা standard model কে দুইভাবে দাও: (১) সরাসরি উত্তর চাও, (২) "আগে ধাপে ধাপে ভাবো, তারপর উত্তর দাও" যোগ করে। দুইবারের সঠিকতা আর output token তুলনা করো।

### Production implementation
Default হিসেবে standard model বা reasoning কম রাখো। শুধু যে route এ eval দিয়ে প্রমাণিত যে বেশি চিন্তা লাগে, সেখানে বাড়াও।

### Failure cases
জটিল হিসাবের প্রশ্নে আত্মবিশ্বাসী কিন্তু ভুল উত্তর, কারণ প্রথম কয়েকটা token এ ভুল দিকে চলে গেছে।

### Security concerns
—

### Performance concerns
সবচেয়ে কম latency।

### Cost concerns
সবচেয়ে কম output token।

### What did I learn?
- বেশিরভাগ কাজে standard model ই যথেষ্ট। Reasoning একটা বাড়তি হাতিয়ার, default না।

### What can I explain to someone else?
> "Standard model হলো quiz show এর প্রতিযোগী, প্রশ্ন শেষ হওয়ার আগেই buzzer চাপে। সহজ প্রশ্নে দারুণ, কঠিন অঙ্কে তাড়াহুড়োয় ভুল।"

### GitHub Evidence
- [ ] Direct vs "ধাপে ধাপে ভাবো" prompt comparison

### LinkedIn Post
- [ ] —

---

<a id="reasoning-models"></a>

## 2. Reasoning Models

### What is it?
যে model উত্তর দেওয়ার আগে ভেতরে লম্বা একটা চিন্তার ধাপ চালায়: সমস্যা ভাঙে, চেষ্টা করে, নিজের ভুল ধরে, আবার চেষ্টা করে। তারপর চূড়ান্ত উত্তর দেয়। উদাহরণ: OpenAI এর o-series, DeepSeek-R1, Claude আর Gemini এর thinking mode।

### Why does it exist?
Standard model বহু ধাপের সমস্যায় প্রথম ভুলেই আটকে যায়। গবেষণায় দেখা গেছে, model কে উত্তরের আগে "ভাবার" সময় দিলে কঠিন প্রশ্নে নির্ভুলতা অনেক বাড়ে।

### What problem does it solve?
Math, coding, logic, পরিকল্পনা, এমন কাজ যেখানে একাধিক ধাপ মিলিয়ে সঠিক উত্তরে পৌঁছাতে হয়।

### How does it work?
- Model কে বিশেষভাবে train করা হয় (প্রায়ই **Reinforcement Learning** দিয়ে), যাতে সে উত্তরের আগে লম্বা চিন্তা লেখে আর সেই চিন্তা থেকে সঠিক উত্তরে পৌঁছায়।
- এই চিন্তার অংশকে বলে **reasoning tokens** বা **thinking tokens**।
- **Test-time compute:** Training এ বেশি খরচ না করে, প্রশ্নের উত্তর দেওয়ার সময় বেশি হিসাব (বেশি token) খরচ করে quality বাড়ানো।

### Important concepts
- **চিন্তা সবসময় দেখা যায় না:** অনেক provider আসল চিন্তা দেখায় না। কেউ শুধু summary দেয়, কেউ কিছুই দেয় না। কিন্তু **bill হয় পুরো চিন্তার**।
- **Open reasoning model:** DeepSeek-R1 এর মতো open model এ চিন্তাটা পুরোটা দেখা যায়, শেখার জন্য দারুণ।
- **Sampling বন্ধ থাকতে পারে:** অনেক reasoning model এ temperature এর মতো parameter বদলানো যায় না (Sampling chapter দেখো)।

### Trade-offs
কঠিন প্রশ্নে অনেক বেশি নির্ভুল, কিন্তু অনেক বেশি token, খরচ আর সময়।

### Alternatives
Standard model + chain-of-thought prompt; tool use (হিসাব code কে দেওয়া); কাজটা ছোট ছোট ধাপে ভেঙে workflow বানানো।

### When should I use it?
[When reasoning models are useful](#when-reasoning-models-are-useful) section এ বিস্তারিত।

### When should I NOT use it?
সহজ কাজ, real-time chat, voice, high-volume classification।

### Hands-on experiment
Ollama তে ছোট একটা open reasoning model চালিয়ে চিন্তাটা নিজের চোখে দেখো:

```bash
ollama run deepseek-r1:7b "একটা দোকানে কলম 12 টাকা, খাতা কলমের চেয়ে 30 টাকা বেশি। 3টা কলম আর 2টা খাতার দাম কত?"
```

Output এ উত্তরের আগে model এর চিন্তার অংশ দেখা যাবে (সাধারণত `<think>` চিহ্নিত)। খেয়াল করো: কতটা লম্বা, কোথায় নিজের ভুল ধরছে, আর চূড়ান্ত উত্তর ঠিক কিনা (সঠিক উত্তর: 3×12 + 2×42 = 120 টাকা)।

### Production implementation
- Reasoning model কে সব route এ না, শুধু কঠিন route এ ব্যবহার করো (routing, Phase 10)।
- Reasoning এর পরিমাণ (effort) route অনুযায়ী ঠিক করো।

### Failure cases
- **Overthinking:** সহজ প্রশ্নেও হাজার token চিন্তা, ফলে অকারণে ধীর আর দামি।
- লম্বা চিন্তার শেষে `max_tokens` শেষ হয়ে যাওয়া, চূড়ান্ত উত্তর আর লেখাই হলো না।

### Security concerns
চিন্তার অংশে model এমন তথ্য লিখতে পারে যা user কে দেখানো উচিত না (system prompt এর অংশ, অন্য document এর লেখা)। চিন্তা user কে দেখানোর আগে ভেবে দেখো।

### Performance concerns
TTFT (প্রথম উত্তরের token) অনেক দেরিতে আসে, কারণ আগে চিন্তা শেষ হতে হয়।

### Cost concerns
চিন্তার token output হিসেবে bill হয়, আর output সবচেয়ে দামি token (Core Concepts)।

### What did I learn?
- Reasoning model "বেশি বুদ্ধিমান" না, "বেশি সময় নিয়ে ভাবে"। সেই সময়ের দাম দিতে হয়।

### What can I explain to someone else?
> "Reasoning model হলো পরীক্ষায় rough খাতায় আগে অঙ্ক কষে তারপর উত্তরপত্রে লেখা ছাত্র। ভুল কম করে, কিন্তু সময় বেশি নেয়, আর rough খাতার কাগজের দামও তুমি দাও।"

### GitHub Evidence
- [ ] DeepSeek-R1 (local) এর চিন্তা পর্যবেক্ষণ নোট

### LinkedIn Post
- [ ] —

---

<a id="extended-thinking"></a>

## 3. Extended Thinking

### What is it?
Model কতটা চিন্তা করবে, তা নিয়ন্ত্রণ করার ব্যবস্থা। Provider ভেদে নাম আর পদ্ধতি আলাদা, আর সময়ের সাথে বদলেছে।

### Why does it exist?
সব প্রশ্নে সমান চিন্তা লাগে না। নিয়ন্ত্রণ না থাকলে সহজ প্রশ্নে অপচয়, কঠিন প্রশ্নে ঘাটতি।

### What problem does it solve?
একই model এ কাজ অনুযায়ী quality, খরচ আর গতির balance ঠিক করা।

### How does it work?
তিন ধরনের নিয়ন্ত্রণ দেখা যায়:

| পদ্ধতি | কীভাবে | উদাহরণ |
|---|---|---|
| **Fixed budget** | সর্বোচ্চ কত token চিন্তা করবে, সংখ্যা দিয়ে বলে দেওয়া | Anthropic এর পুরনো "extended thinking" (`budget_tokens`) |
| **Effort level** | low / medium / high এর মতো মাত্রা বেছে নেওয়া | OpenAI এর `reasoning_effort`, Anthropic এর `effort` |
| **Adaptive** | Model নিজেই প্রশ্ন বুঝে ঠিক করে কতটা ভাববে, effort দিয়ে দিক নির্দেশ | Anthropic এর নতুন model এর adaptive thinking |

**নাম নিয়ে সতর্কতা:** Anthropic এ "extended thinking" নামটা পুরনো fixed-budget পদ্ধতির। নতুন Claude model গুলোতে সেটা আর চলে না, এখন adaptive thinking আর `effort` ব্যবহার হয়। Roadmap এ "Extended thinking" লেখা আছে, কিন্তু আসলে শিখতে হবে **"model এর চিন্তা কীভাবে নিয়ন্ত্রণ করা যায়"**, নির্দিষ্ট কোনো একটা parameter না।

### Important concepts
- **Effort শুধু চিন্তা না:** অনেক model এ effort কমালে চিন্তার পাশাপাশি পুরো উত্তরের দৈর্ঘ্য আর tool call এর সংখ্যাও কমে।
- **Thinking display:** কিছু provider এ চিন্তা দেখানোর option আছে: পুরোটা লুকানো, summary দেখানো, ইত্যাদি। দেখানো না হলেও bill একই।
- **কিছু model এ thinking বন্ধ করা যায় না:** তখন খরচ কমানোর উপায় হলো effort কমানো।

### Trade-offs
বেশি effort = কঠিন কাজে ভালো quality, কিন্তু সহজ কাজে শুধু অপচয়।

### Alternatives
আলাদা আলাদা model বেছে নেওয়া (ছোট দ্রুত model বনাম বড় reasoning model)। প্রায়ই একই model এ effort বদলানো এর চেয়ে সহজ আর কার্যকর।

### When should I use it?
একই app এ সহজ আর কঠিন দুই ধরনের কাজ থাকলে, route অনুযায়ী আলাদা effort।

### When should I NOT use it?
সব জায়গায় সর্বোচ্চ effort দিয়ে রাখা। কোথায় বাড়ালে সত্যিই quality বাড়ে, তা eval দিয়ে মাপো।

### Hands-on experiment
একই ১০টা প্রশ্ন (৫টা সহজ, ৫টা কঠিন) low আর high effort এ চালাও। প্রতিটায় সঠিকতা, output token আর সময় লিখে রাখো। দেখো কোন ধরনের প্রশ্নে high effort আসলে লাভ দেয়।

### Production implementation
- Effort কে config এ রাখো, route অনুযায়ী।
- Model বা version বদলালে effort এর default আর আচরণ বদলাতে পারে, তাই আবার eval চালাও।

### Failure cases
- পুরনো tutorial দেখে নতুন model এ পুরনো parameter (যেমন fixed budget) পাঠানো, ফলে API error।
- Effort এর default model ভেদে আলাদা, না জেনে ব্যবহার করায় অপ্রত্যাশিত খরচ বা দুর্বল quality।

### Security concerns
—

### Performance concerns
Effort বাড়ালে latency প্রায় সরাসরি বাড়ে।

### Cost concerns
Effort হলো reasoning model এর সবচেয়ে বড় cost lever।

### What did I learn?
- "কতটা ভাববে" এখন একটা setting। আর সেই setting টা ঠিক করা একটা engineering সিদ্ধান্ত।

### What can I explain to someone else?
> "Extended thinking হলো ছাত্রকে বলা, 'এই প্রশ্নে ৫ মিনিট ভাবো' বা 'এটায় ৩০ মিনিট নাও।' নতুন model গুলো নিজেই বুঝে নেয় কোন প্রশ্নে কতক্ষণ ভাবতে হবে।"

### GitHub Evidence
- [ ] Low vs high effort: accuracy, token, latency table

### LinkedIn Post
- [ ] —

---

<a id="reasoning-token-costs"></a>

## 4. Reasoning Token Costs

### What is it?
Reasoning model এর চিন্তার token গুলো **output token** হিসেবে bill হয়, user সেগুলো দেখুক বা না দেখুক।

### Why does it exist?
চিন্তার token তৈরি করতেও model কে একটা একটা করে token generate করতে হয় (decode), ঠিক উত্তরের token এর মতো। তাই একই হারে খরচ।

### What problem does it solve?
এটা সমস্যা সমাধান করে না, এটা একটা **লুকানো খরচ**, যেটা না জানলে budget ভুল হয়।

### How does it work?
উদাহরণ (কাল্পনিক দাম: output $5 per 1M token):

```text
Standard model:   উত্তর 200 token                      → 200 × $5 ÷ 1M   = $0.001
Reasoning model:  চিন্তা 3,000 + উত্তর 200 = 3,200 token → 3,200 × $5 ÷ 1M = $0.016

একই দৈর্ঘ্যের উত্তর, কিন্তু খরচ 16 গুণ।
```

User দেখলো শুধু 200 token এর উত্তর, কিন্তু bill হলো 3,200 token এর।

### Important concepts
- **Response এর `usage` দেখো:** অনেক API তে output token এর মধ্যে reasoning token ও গোনা থাকে, কেউ আলাদা করেও দেখায়। Log এ সবসময় মোট output token রাখো।
- **`max_tokens` এ চিন্তাও গোনা হয়:** কম রাখলে চিন্তাতেই সব শেষ, উত্তরের জায়গা থাকে না।
- **Token Cost post এর "লুকানো token" এর আরেক রূপ:** system prompt, tool definition এর মতো reasoning token ও user দেখে না, কিন্তু বিল আসে।

### Trade-offs
বেশি খরচ, কিন্তু কঠিন কাজে কম ভুল মানে কম retry আর কম মানুষের সংশোধন। **Cost per successful task** দিয়ে তুলনা করো।

### Alternatives
Effort কমানো; ছোট reasoning model; সহজ প্রশ্ন standard model এ পাঠানো (routing)।

### When should I use it?
Feature এর খরচ হিসাব করার সময় reasoning token আলাদা করে ধরো।

### When should I NOT use it?
শুধু উত্তরের দৈর্ঘ্য দেখে reasoning model এর খরচ অনুমান করবে না।

### Hands-on experiment
একই প্রশ্ন standard আর reasoning mode এ চালিয়ে response এর `usage` থেকে output token তুলনা করো। Core Concepts এর `estimateCost` function দিয়ে দুইটার খরচ বের করো।

### Production implementation
- প্রতিটা request এ input, output আর (থাকলে) reasoning token আলাদা log করো।
- Route প্রতি reasoning token এর গড় monitor করো, হঠাৎ বেড়ে গেলে alert।

### Failure cases
Prototype এ standard model এর খরচ দেখে budget বানানো হলো, production এ reasoning model দেওয়ায় বিল কয়েক গুণ।

### Security concerns
—

### Performance concerns
বেশি reasoning token = বেশি latency (পরের topic)।

### Cost concerns
এই topic টাই cost। মূল কথা: **চিন্তার দাম আছে, আর সেটা output এর দামে।**

### What did I learn?
- Reasoning model এর আসল খরচ উত্তরের দৈর্ঘ্যে না, চিন্তার দৈর্ঘ্যে।

### What can I explain to someone else?
> "Uber এ যাত্রা ৫ মিনিটের মনে হলো, কিন্তু driver ঘুরপথে ২০ মিনিট চালিয়েছে, আর মিটার চলেছে পুরো ২০ মিনিটের। Reasoning token হলো সেই ঘুরপথ, যেটা তুমি দেখোনি কিন্তু টাকা দিয়েছো।"

### GitHub Evidence
- [ ] Standard vs reasoning: output token আর cost comparison

### LinkedIn Post
- [ ] "AI এর উত্তর ছোট, কিন্তু বিল বড় কেন? Reasoning token এর লুকানো খরচ"

---

<a id="reasoning-vs-latency"></a>

## 5. Reasoning vs Latency

### What is it?
Reasoning যত বেশি, user কে উত্তরের জন্য তত বেশি অপেক্ষা করতে হয়।

### Why does it exist?
চিন্তার token ও একটা একটা করে তৈরি হয়। উত্তরের প্রথম শব্দ আসার আগে পুরো চিন্তা শেষ হতে হয়।

### What problem does it solve?
এটা একটা constraint। জানা থাকলে বোঝা যায় কোথায় reasoning চলবে আর কোথায় চলবে না।

### How does it work?
Core Concepts এর latency সূত্রে চিন্তার অংশ যোগ হয়:

```text
উত্তরের প্রথম শব্দ আসতে সময় ≈ TTFT + (reasoning tokens ÷ TPS)

উদাহরণ: TTFT 0.5s, 50 tokens/s, 3,000 reasoning token
→ 0.5 + 60 = প্রায় 60 সেকেন্ড পর্যন্ত user কোনো উত্তর দেখে না
```

### Important concepts
- **Streaming এখানে কম কাজের:** চিন্তা লুকানো থাকলে stream চালু থাকলেও user অনেকক্ষণ কিছুই দেখে না।
- **UX এর সমাধান:** চিন্তার summary বা progress দেখানো ("বিশ্লেষণ করছি...", "ধাপ ২/৪"), যাতে user বোঝে কাজ চলছে।
- **Agent এ গুণফল:** প্রতিটা tool call এর আগে চিন্তা হলে, ৫ ধাপের agent এ latency ৫ গুণ।

### Trade-offs
কঠিন কাজে নির্ভুলতা বনাম user এর ধৈর্য।

### Alternatives
- কম effort।
- কঠিন কাজ background job এ দিয়ে শেষ হলে notification (user কে অপেক্ষা করিয়ে রাখার দরকার নেই)।
- আগে দ্রুত model দিয়ে একটা প্রাথমিক উত্তর, পরে reasoning model দিয়ে বিস্তারিত।

### When should I use it?
Report বানানো, code review, বিশ্লেষণ, যেখানে user কিছুক্ষণ অপেক্ষা করতে রাজি।

### When should I NOT use it?
Voice assistant, autocomplete, সাধারণ chat, যেখানে প্রতিটা সেকেন্ড গুরুত্বপূর্ণ।

### Hands-on experiment
Generation chapter এর streaming code দিয়ে একটা reasoning model এর TTFT মাপো, আর standard model এর সাথে তুলনা করো। চিন্তা দেখানো আর না দেখানোর মধ্যে user এর অনুভূতির পার্থক্য লিখে রাখো।

### Production implementation
- Reasoning route এ timeout বাড়িয়ে রাখো, নাহলে লম্বা চিন্তার মাঝে request কেটে যাবে।
- লম্বা কাজে streaming আর progress UI দাও।
- p95 latency মাপো, কারণ কিছু প্রশ্নে চিন্তা অনেক লম্বা হয়।

### Failure cases
- Default HTTP timeout (যেমন 30s) এ reasoning request কেটে যাওয়া, অথচ token এর খরচ হয়ে গেছে।
- User কে ৪০ সেকেন্ড খালি screen দেখানো, user ভাবলো app আটকে গেছে।

### Security concerns
—

### Performance concerns
এই topic টাই performance। মূল lever: **effort, routing, আর progress UI।**

### Cost concerns
Timeout হয়ে retry করলে একই চিন্তার জন্য দুইবার বিল।

### What did I learn?
- Reasoning এর সবচেয়ে বড় দাম অনেক সময় টাকা না, user এর অপেক্ষা।

### What can I explain to someone else?
> "ভালো রাঁধুনির রান্নায় সময় লাগে। অতিথি অপেক্ষা করতে রাজি হলে ঠিক আছে, কিন্তু নাশতার টেবিলে এক ঘণ্টা বসিয়ে রাখা যায় না।"

### GitHub Evidence
- [ ] Standard vs reasoning TTFT table

### LinkedIn Post
- [ ] —

---

<a id="when-reasoning-models-are-useful"></a>

## 6. When Reasoning Models Are Useful

### What is it?
কোন কাজে reasoning model এর বাড়তি খরচ আর সময় দেওয়া যুক্তিযুক্ত, আর কোনটায় না, তার সিদ্ধান্ত।

### Why does it exist?
Reasoning সব কাজে উন্নতি দেয় না। ভুল জায়গায় ব্যবহার করলে শুধু খরচ আর দেরি বাড়ে।

### What problem does it solve?
সঠিক কাজে সঠিক model, যাতে quality, খরচ আর গতি তিনটারই ভারসাম্য থাকে।

### How does it work?
| Reasoning দরকারি | Reasoning অপ্রয়োজনীয় |
|---|---|
| বহু ধাপের math আর logic | Classification (positive/negative) |
| কঠিন bug খোঁজা, code এর বড় পরিবর্তন | সাধারণ extraction (invoice থেকে টাকা) |
| পরিকল্পনা (কাজকে ধাপে ভাগ করা) | Summary, অনুবাদ |
| বহু ধাপের agent কাজ, কোন tool কখন লাগবে ঠিক করা | সাধারণ chat, FAQ |
| জটিল document বিশ্লেষণ, পরস্পরবিরোধী তথ্য মেলানো | Real-time voice, autocomplete |

**সহজ পরীক্ষা:** একজন মানুষের কাজটা করতে কি কাগজ-কলম নিয়ে বসতে হয়? হলে reasoning কাজে লাগতে পারে। মুখে মুখে উত্তর দেওয়া গেলে লাগবে না।

### Important concepts
- **Eval দিয়ে প্রমাণ:** অনুমান না, নিজের কাজের প্রশ্নে low/high effort বা standard/reasoning চালিয়ে মাপো (Benchmark post এর শিক্ষা)।
- **Routing:** একই app এ সহজ প্রশ্ন দ্রুত model এ, কঠিন প্রশ্ন reasoning এ (Phase 10)।
- **Tool vs reasoning:** হিসাবের কাজ প্রায়ই calculator বা code tool দিলে reasoning এর চেয়ে সস্তায় আর নির্ভুলভাবে হয় (Model Limitations)।

### Trade-offs
সঠিক জায়গায় reasoning = বড় quality লাভ; ভুল জায়গায় = শুধু খরচ আর দেরি।

### Alternatives
Tool use, কাজটা ছোট ধাপে ভাঙা workflow, chain-of-thought prompt।

### When should I use it?
Eval এ দেখা গেছে standard model বা কম effort এ ভুলের হার বেশি, আর বেশি চিন্তায় সেটা কমে।

### When should I NOT use it?
"নতুন আর শক্তিশালী" বলেই সব কাজে reasoning দেওয়া।

### Hands-on experiment
নিজের ২০টা আসল কাজ নাও। প্রতিটা standard আর reasoning mode এ চালিয়ে table বানাও: সঠিকতা, খরচ, সময়। কোন কাজে reasoning **সত্যিই** পার্থক্য আনলো, চিহ্নিত করো। এটা **LLM Model Comparison Lab** এর reasoning অংশ।

### Production implementation
- Route ভিত্তিক config: কোন feature এ কোন model আর কোন effort।
- Reasoning route এর quality আর খরচ আলাদা dashboard এ।

### Failure cases
সব customer chat reasoning model এ চালানোয় উত্তর ধীর, বিল কয়েক গুণ, অথচ quality তে চোখে পড়ার মতো পার্থক্য নেই।

### Security concerns
বেশি সক্ষম reasoning agent এর হাতে tool থাকলে সে নিজে নিজে বেশি ধাপের কাজ করে ফেলতে পারে। Permission সীমিত রাখো, জরুরি কাজে মানুষের approval।

### Performance concerns
Routing এ শুধু কঠিন কাজ reasoning এ গেলে গড় latency কম থাকে।

### Cost concerns
সঠিক routing হলো reasoning এর খরচ নিয়ন্ত্রণের সবচেয়ে বড় উপায়।

### What did I learn?
- Reasoning model একটা বিশেষজ্ঞ, সব কাজের লোক না।

### What can I explain to someone else?
> "Hospital এ সব রোগীকে সরাসরি specialist দেখায় না। আগে সাধারণ ডাক্তার দেখে, কঠিন হলে specialist এর কাছে পাঠায়। AI তেও সহজ প্রশ্ন সাধারণ model, কঠিন প্রশ্ন reasoning model।"

### GitHub Evidence
- [ ] Standard vs reasoning: ২০টা task এর comparison (Model Comparison Lab)

### LinkedIn Post
- [ ] "সব কাজে Reasoning model লাগে না, কখন লাগে?"

---

🏁 **Phase 1 এর সব topic শেষ।** বাকি আছে Phase 1 এর practical project: **LLM Model Comparison Lab**। আগের chapter গুলোর প্রায় প্রতিটা hands-on experiment এই project এর একটা অংশ।

⬅️ Back to [Phase 1 — LLM Fundamentals](README.md) · [Rerankers](rerankers.md) · [Root README](../README.md)

# Phase 3 — Context Engineering: Prompt Caching (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Caching এর দাম, TTL, minimum token আর parameter নাম provider বারবার বদলায়। এখানে দেওয়া সংখ্যা **উদাহরণ**। Production এ নেওয়ার আগে সবসময় official docs দেখবে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [OpenAI prompt caching](#openai-prompt-caching) | Automatic: লম্বা (≥~1024 token) prefix নিজে থেকেই cache হয়, code লাগে না |
| [Anthropic prompt caching](#anthropic-prompt-caching) | `cache_control` দিয়ে explicit (বা top-level auto): write একটু দামি, read খুব সস্তা |
| [Cacheable context](#cacheable-context) | যা অনেক request এ byte-for-byte একই: tools, system, examples, বড় document, পুরনো history |
| [Cache invalidation](#cache-invalidation) | Prefix এর যেকোনো এক byte বদলালে তার পরের সব cache miss |
| [Cost optimization](#cost-optimization) | Hit rate মাপো, stable prefix বানাও, TTL ঠিক করো, break-even হিসাব করো |

```text
Request 1:  [ tools | system | docs ]────[ question A ]
             └──── prefix: prefill করে KV cache এ রাখা হলো (write)
Request 2:  [ tools | system | docs ]────[ question B ]
             └──── হুবহু একই prefix → cache read (সস্তা + দ্রুত) → শুধু নতুন অংশ process
```

> **মূল নিয়ম:** Caching হলো **prefix match**। Stable জিনিস আগে, volatile জিনিস পরে।

---

<a id="openai-prompt-caching"></a>

## 1. OpenAI Prompt Caching

### What is it?
OpenAI API এর **automatic** prompt caching। একটা নির্দিষ্ট length এর বেশি (প্রায় ১০২৪ token) prompt এর prefix আগের কোনো request এর সাথে মিললে সেই অংশ cache থেকে আসে, কম দামে আর দ্রুত। কোনো code পরিবর্তন লাগে না।

### Why does it exist?
অনেক app প্রতিটা request এ একই লম্বা system prompt, tool আর example পাঠায়। Provider এর GPU কে একই কাজ বারবার করতে হয়। Cache করলে provider এর compute বাঁচে, আর সেই সাশ্রয়ের অংশ developer কে discount হিসেবে দেওয়া হয়।

### What problem does it solve?
Repeated prefix এর input cost আর TTFT কমায়, developer এর কোনো extra কাজ ছাড়াই।

### How does it work?
```text
prompt ≥ ~1024 token → প্রথম অংশের hash দিয়ে একই machine এ route
→ আগে দেখা prefix হলে cached অংশ ছাড় দামে বিল (prefix ~128 token এর ধাপে match হয়)
→ usage এ cached_tokens দেখায়
```
Cache সাধারণত কয়েক মিনিট idle থাকলে মুছে যায় (off-peak এ বেশিক্ষণ থাকতে পারে)। কিছু model এ extended retention (যেমন ২৪ ঘণ্টা) option আছে।

### Important concepts
- **Automatic**: `cache_control` এর মতো marker নেই।
- **Usage field**: Chat Completions এ `usage.prompt_tokens_details.cached_tokens`, Responses API তে `usage.input_tokens_details.cached_tokens`।
- **`prompt_cache_key`**: একই prefix ব্যবহার করা request গুলোকে একসাথে route করার hint, hit rate বাড়ায়।
- **Discount model ভেদে আলাদা** (পুরনো model এ ~৫০%, নতুন model এ আরও বেশি)। Write এর জন্য extra charge নেই।
- Tools, images, structured output schema ও prefix এর অংশ।

### Trade-offs
Zero effort, কিন্তু কম control: কী cache হবে বা কতক্ষণ থাকবে সেটা নিশ্চিতভাবে বলতে পারো না।

### Alternatives
Anthropic এর explicit caching, নিজের response cache (exact/semantic), self-hosted এ vLLM এর prefix caching।

### When should I use it?
OpenAI ব্যবহার করলে সবসময় চালু থাকে। শুধু prompt এর order ঠিক রাখো যাতে hit হয়।

### When should I NOT use it?
Disable করা যায় না। তবে ছোট prompt (<১০২৪ token) এ এর উপর ভরসা করে cost plan করবে না।

### Hands-on experiment
```ts
// npm i openai
import OpenAI from "openai";
const client = new OpenAI();
const SYSTEM = "...(২০০০+ token এর লম্বা, স্থির instruction)...";

for (const q of ["Refund policy?", "Shipping time?", "Warranty?"]) {
  const res = await client.chat.completions.create({
    model: "gpt-5-mini", // উদাহরণ; docs থেকে current model নাও
    messages: [{ role: "system", content: SYSTEM }, { role: "user", content: q }],
  });
  console.log(q, res.usage?.prompt_tokens, "cached:", res.usage?.prompt_tokens_details?.cached_tokens);
}
```
প্রথম call এ cached = 0, পরের গুলোতে বেশিরভাগ prefix cached কিনা দেখো।

### Production implementation
- Static content (instructions, examples, tools) শুরুতে, user-specific শেষে।
- `cached_tokens / prompt_tokens` = hit rate, dashboard এ রাখো।
- একই prefix এর traffic এ `prompt_cache_key` দাও।

### Failure cases
- System prompt এর শুরুতে user name বসানোয় hit rate ০%।
- Traffic কম, ফলে request গুলোর মাঝে cache expire হয়ে যাচ্ছে।

### Security concerns
Cache organization এর ভেতরে সীমিত, অন্য org এর সাথে share হয় না। তবু timing দিয়ে prefix অনুমান (side-channel) একটা theoretical risk। খুব sensitive prefix নিয়ে সচেতন থাকো।

### Performance concerns
লম্বা prompt এ TTFT উল্লেখযোগ্যভাবে কমে।

### Cost concerns
Free discount: শুধু order ঠিক রাখলেই পাওয়া যায়।

### What did I learn?
- OpenAI এ caching "free lunch", কিন্তু prefix stable না রাখলে lunch পাওয়া যায় না।

### What can I explain to someone else?
> "দোকানদার নিয়মিত customer এর order মনে রাখে। একই order হলে দ্রুত দেয় আর একটু কম দাম নেয়। তুমি শুধু প্রতিবার order একই ভাবে শুরু করো।"

### GitHub Evidence
- [ ] OpenAI cached_tokens experiment log

### LinkedIn Post
- [ ] "OpenAI prompt caching: code না বদলে খরচ কমানো"

---

<a id="anthropic-prompt-caching"></a>

## 2. Anthropic Prompt Caching

### What is it?
Claude API এর prompt caching, যেখানে তুমি **`cache_control`** marker দিয়ে বলে দাও কোন পর্যন্ত prefix cache হবে (explicit breakpoint), অথবা request এর top-level এ `cache_control` দিয়ে **automatic caching** চালু করো।

### Why does it exist?
Explicit control দিলে developer ঠিক করতে পারে কোন অংশ reusable, কতক্ষণ রাখবে (TTL), আর কোথায় breakpoint। জটিল agent/RAG এ এটা বেশি predictable।

### What problem does it solve?
বড় system prompt, tool list, document, লম্বা conversation, এগুলো বারবার পাঠানোর খরচ আর latency অনেক কমায়।

### How does it work?
```text
render order: tools → system → messages
cache_control: {type: "ephemeral"}            → 5 মিনিট TTL (default)
cache_control: {type: "ephemeral", ttl: "1h"} → 1 ঘণ্টা TTL

দাম (base input এর গুণ হিসেবে, সাধারণ নিয়ম):
  cache write (5m) ≈ 1.25×   cache write (1h) ≈ 2×   cache read ≈ 0.1× (কিছু নতুন model এ আরও কম)
```
- প্রতি request এ সর্বোচ্চ **৪টা breakpoint**।
- **Minimum cacheable length model ভেদে আলাদা** (মোটামুটি ৫১২–৪০৯৬ token)। এর কম হলে error ছাড়াই চুপচাপ cache হয় না।
- Cache read হলে TTL timer আবার শুরু হয়।

### Important concepts
- **Usage fields**:
  - `cache_creation_input_tokens`: এই request এ cache এ লেখা (write premium)
  - `cache_read_input_tokens`: cache থেকে পড়া (সস্তা)
  - `input_tokens`: বাকি uncached অংশ
  - মোট prompt = তিনটার যোগফল
- **Automatic caching**: top-level `cache_control`, breakpoint নিজে শেষ cacheable block এ বসায় আর conversation বাড়ার সাথে এগোয়।
- **Shared prefix, varying suffix**: breakpoint দাও shared অংশের শেষে, পুরো prompt এর শেষে না।
- **Workspace-scoped**: cache workspace/org এর বাইরে share হয় না।
- **Model-scoped**: model বদলালে cache আলাদা।

### Trade-offs
বেশি control আর বড় discount, কিন্তু write premium আছে। একবারই ব্যবহার হলে caching উল্টো বেশি খরচ।

### Alternatives
Caching ছাড়া চালানো (ছোট/unique prompt এ), নিজের response cache।

### When should I use it?
- বড় stable system prompt + tools।
- একই document নিয়ে অনেক প্রশ্ন।
- Multi-turn chat ও agent loop (history বাড়ে, আগের অংশ reuse হয়)।

### When should I NOT use it?
Prompt এর প্রথম অংশই প্রতি request এ আলাদা হলে reusable prefix নেই, তাই শুধু write premium দিতে হবে, read হবে না।

### Hands-on experiment
```ts
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic();
const LONG_DOC = "...(কয়েক হাজার token এর policy document)...";

for (const q of ["Refund কতদিনে?", "Shipping charge কত?", "Warranty আছে?"]) {
  const res = await client.messages.create({
    model: "claude-opus-5-5",
    max_tokens: 1024,
    system: [
      { type: "text", text: "You answer only from the policy document." },
      { type: "text", text: LONG_DOC, cache_control: { type: "ephemeral" } }, // shared prefix এর শেষে
    ],
    messages: [{ role: "user", content: q }],
  });
  const u = res.usage;
  console.log(q, { write: u.cache_creation_input_tokens, read: u.cache_read_input_tokens, uncached: u.input_tokens });
}
```
প্রথম call এ write, পরের দুটোয় read দেখা উচিত। তারপর LONG_DOC এ একটা space যোগ করে আবার চালাও, আবার write হবে।

### Production implementation
- **Agent loop এর জন্য robust pattern**: static system prefix এর শেষে একটা explicit breakpoint + top-level automatic caching (growing conversation এর জন্য)।
- Hit rate monitor: `read / (read + write + uncached)`।
- Cache warm রাখতে দরকার হলে startup এ `max_tokens: 0` দিয়ে pre-warm (docs দেখে)।

### Failure cases
- Prompt minimum এর চেয়ে ছোট, তাই marker দিয়েও কখনো cache হচ্ছে না (কোনো error নেই)।
- Breakpoint প্রশ্নের পরে দেওয়ায় প্রতিটা request নতুন entry লিখছে, read হচ্ছে না।
- Thinking/effort setting বদলানোয় messages cache ভেঙে গেছে।

### Security concerns
Cache data workspace এর ভেতরে isolated। কিন্তু multi-tenant app এ এক tenant এর data অন্যের prefix এ ঢোকাবে না। Tenant data সবসময় shared prefix এর পরে রাখো।

### Performance concerns
লম্বা prefix এ TTFT অনেক কমে (বড় document এ সবচেয়ে বেশি লাভ)।

### Cost concerns
Break-even: 5m TTL এ **২টা request** (1.25 + 0.1 = 1.35× বনাম uncached 2×)। 1h TTL এ অন্তত **৩টা** (2 + 0.2 = 2.2× বনাম 3×)।

### What did I learn?
- Anthropic caching = explicit, শক্তিশালী, কিন্তু breakpoint এর জায়গা ভুল হলে উল্টো খরচ।
- সবসময় `usage` দেখে যাচাই করো।

### What can I explain to someone else?
> "Photocopy shop এ প্রথমবার master copy বানাতে একটু বেশি টাকা লাগে (write), তারপর প্রতিটা কপি খুব সস্তা (read)। কিন্তু master এ একটা অক্ষর বদলালে আবার নতুন master বানাতে হয়।"

### GitHub Evidence
- [ ] Claude cache write/read experiment + break-even calculation

### LinkedIn Post
- [ ] "Claude prompt caching: ৯০% সস্তা input, যদি জানো কোথায় marker বসাতে হয়"

---

<a id="cacheable-context"></a>

## 3. Cacheable Context

### What is it?
Context এর সেই অংশ যা অনেক request এ **byte-for-byte একই** থাকে, তাই cache করার যোগ্য: tool definitions, system prompt, few-shot examples, বড় reference document, conversation এর আগের অংশ।

### Why does it exist?
Caching কাজ করে শুধু হুবহু মিলে গেলে। তাই কোন অংশ stable আর কোনটা volatile, সেটা আলাদা করে চেনা দরকার।

### What problem does it solve?
Prompt design এর সময় সিদ্ধান্ত নেওয়া যায়: কী আগে রাখব, কী পরে, কোথায় breakpoint দেব।

### How does it work?
```text
Cacheable (stable)                     Not cacheable (volatile)
─────────────────────                  ────────────────────────
Tool definitions (sorted)              Current timestamp
System prompt (frozen)                 Request/trace ID
Few-shot examples                      Retrieved RAG chunks (প্রায়ই)
Reference docs / codebase              User's new question
Previous conversation turns            Random/shuffled order
```

### Important concepts
- **Frequency × size**: বড় আর ঘন ঘন reuse হওয়া অংশেই বেশি লাভ।
- **Per-user prefix**: একই user এর পরপর request এ তার profile/history cacheable।
- **Hierarchy**: global (সবার জন্য একই) → per-tenant → per-conversation।
- **Determinism**: JSON key sort, stable tool order, কোনো `Math.random()` না।

### Trade-offs
বেশি অংশ stable করতে গেলে কিছু flexibility হারাও (যেমন প্রতি request এ ভিন্ন tool set দেওয়া যায় না)।

### Alternatives
Tool search/deferred loading (tool list স্থির রেখে দরকারমতো load), stable tool set + permission code এ।

### When should I use it?
Prompt design এর শুরুতেই প্রতিটা অংশকে stable/volatile label করো।

### When should I NOT use it?
ছোট আর একবার ব্যবহৃত prompt এ এ বিশ্লেষণ অপ্রয়োজনীয়।

### Hands-on experiment
তোমার Phase 2 এর `compose()` function নাও। প্রতিটা অংশ "stable / per-user / per-request" label করো। তারপর order এমনভাবে সাজাও যাতে stable অংশ সবচেয়ে বড় contiguous prefix হয়। আগে-পরে cache hit মাপো।

### Production implementation
- Prompt builder এ প্রতিটা segment এর `cacheTier: "global" | "tenant" | "conversation" | "volatile"`।
- Unit test: একই input এ দুইবার build করে byte-equal কিনা assert করো।

### Failure cases
- `JSON.stringify(obj)` এ key order ভিন্ন হওয়ায় tool schema প্রতিবার আলাদা।
- Feature flag অনুযায়ী tool list বদলায়, ফলে প্রতি user ভিন্ন prefix।

### Security concerns
Cacheable global prefix এ কোনো user/tenant এর data রাখবে না।

### Performance concerns
Stable prefix যত বড়, TTFT তত কম।

### Cost concerns
Cacheable অংশের অনুপাত = তোমার সম্ভাব্য সর্বোচ্চ সাশ্রয়।

### What did I learn?
- Caching আসলে architecture এর প্রশ্ন: কোন জিনিস স্থির, কোনটা চলমান।

### What can I explain to someone else?
> "Wedding card এ নাম ছাড়া বাকি সব একই, তাই design একবার ছাপিয়ে রাখো, নামটা শুধু হাতে লেখো।"

### GitHub Evidence
- [ ] Prompt segment cache-tier labeling + byte-equality test

### LinkedIn Post
- [ ] "তোমার prompt এর কোন অংশ cache করা যায়?"

---

<a id="cache-invalidation"></a>

## 4. Cache Invalidation

### What is it?
যে পরিবর্তনের কারণে আগের cache আর match করে না আর নতুন করে write করতে হয়। Prompt caching এ নিয়ম সহজ কিন্তু নির্মম: **prefix এর যেকোনো জায়গায় এক byte বদলালে তার পরের সব কিছু invalid**।

### Why does it exist?
Cache আসলে model এর internal state (KV cache), যেটা আগের সব token এর উপর নির্ভর করে। আগের একটা token বদলালে পরের সব state বদলে যায়।

### What problem does it solve?
(জানা) Silent cache miss খুঁজে বের করা আর ঠেকানো। বেশিরভাগ "caching কাজ করছে না" সমস্যা আসলে invalidation।

### How does it work?
```text
[tools][system][msg1][msg2][msg3]
          ▲
   এখানে এক byte বদলালে → system থেকে শেষ পর্যন্ত সব cache miss
   (tools অংশ এখনো hit)

Invalidation hierarchy (Claude): tools বদলালে → সব; system বদলালে → system+messages;
thinking/effort parameter বদলালে → messages cache
```

### Important concepts
- **Silent invalidators**: timestamp, UUID, unsorted JSON, ভিন্ন tool order, whitespace পরিবর্তন, ভিন্ন image encoding।
- **TTL expiry**: সময় পার হলে নিজে থেকেই invalid।
- **Model change**: model বদলালে পুরো cache নতুন।
- **History edit**: আগের message বদলানো বা মুছে ফেলা।
- **Diagnose**: পরপর দুই request এর payload diff করো। Claude এ cache diagnostics (beta) আছে।

### Trade-offs
Strict stability = বেশি hit, কিন্তু prompt বদলানো (deploy) এর পর প্রথম কিছু request এ write cost।

### Alternatives
Prompt পরিবর্তন batch করে deploy করো। Mid-conversation operator instruction এর জন্য system edit না করে `role: "system"` message append করো (support করা model এ)।

### When should I use it?
(এই জ্ঞান) যখনই hit rate কম দেখো।

### When should I NOT use it?
—

### Hands-on experiment
৫টা "invalidator" একটা একটা করে যোগ করো আর প্রতিবার `cache_read_input_tokens` দেখো:
1. System prompt এ `new Date()`
2. Tool list shuffle
3. `JSON.stringify` এ key order বদলানো
4. History র মাঝের একটা message edit
5. ৬ মিনিট অপেক্ষা (5m TTL)

### Production implementation
```ts
import { createHash } from "node:crypto";
const prefixHash = (req: object) => createHash("sha256").update(JSON.stringify(req)).digest("hex").slice(0, 12);
// প্রতি request এ stable অংশের (tools+system) hash log করো।
// একই feature এ hash বদলালে alert → কেউ volatile কিছু ঢুকিয়েছে
```

### Failure cases
- Logging middleware প্রতি request এ system prompt এ trace ID যোগ করছিল।
- Deploy এর পর prompt সামান্য বদলেছে, peak time এ সব cache একসাথে miss হয়ে cost spike।

### Security concerns
Debug এর জন্য payload diff log করলে সেখানে sensitive data থাকতে পারে। Redact করো।

### Performance concerns
Invalidation = cold prefill = TTFT spike।

### Cost concerns
প্রতিটা invalidation এ write premium (1.25×–2×) দিতে হয়, read discount পাওয়া যায় না।

### What did I learn?
- Cache "কাজ করছে না" মানে প্রায় সবসময় কোথাও কিছু প্রতিবার বদলাচ্ছে।

### What can I explain to someone else?
> "Domino সাজানোর মতো: শুরুর দিকের একটা domino সরালে তার পরের সবগুলো আবার সাজাতে হয়। শেষের দিকেরটা সরালে শুধু শেষটুকু।"

### GitHub Evidence
- [ ] 5 invalidator experiment + prefix-hash monitor

### LinkedIn Post
- [ ] "Prompt caching কাজ করছে না? এই ৫টা জিনিস check করো"

---

<a id="cost-optimization"></a>

## 5. Cost Optimization (with Caching)

### What is it?
Prompt caching কে পরিকল্পিতভাবে ব্যবহার করে মোট LLM খরচ কমানো: hit rate মাপা, prefix stable করা, ঠিক TTL বাছা, break-even হিসাব করা, আর caching কে অন্য optimization (batch, routing) এর সাথে মেলানো।

### Why does it exist?
Production AI app এ input token প্রায়ই মোট খরচের বড় অংশ, বিশেষ করে agent আর RAG এ। Caching সবচেয়ে বড় "free win"।

### What problem does it solve?
Quality না কমিয়ে খরচ কমায়। Model বা prompt বদলাতে হয় না, শুধু structure।

### How does it work?
```text
উদাহরণ (কাল্পনিক দাম: input $4/1M, cache read 0.1×, write 1.25×)
System+tools+doc = 20K token, 10K request/দিন, প্রতি ৫ মিনিটের মধ্যে traffic আসে

Caching ছাড়া:  20K × 10K = 200M token × $4/1M = $800/দিন
Caching সহ:    ~200M × 0.1 × $4/1M ≈ $80/দিন (+ মাঝে মাঝে write)
→ এই অংশে প্রায় ৯০% সাশ্রয়
```

### Important concepts
- **Hit rate** = cached input ÷ মোট input। লক্ষ্য: stable-prefix-heavy workload এ ৭০–৯০%+।
- **TTL বাছাই**: request গুলো ৫ মিনিটের মধ্যে আসলে 5m TTL, ৫–৬০ মিনিট gap হলে 1h TTL।
- **Cost per completed task** দেখো, শুধু per request না।
- **Cache + Batch**: non-real-time কাজে batch discount এর সাথে cache যোগ হতে পারে (provider docs দেখো)।
- **Model routing এর সাথে conflict**: cache model-scoped, তাই অনেক model এ ভাগ করলে প্রতিটার আলাদা cache।

### Trade-offs
Caching এর জন্য prompt কে rigid করতে হয়। খুব বেশি personalization সামনে আনলে hit কমে।

### Alternatives
Response caching (একই প্রশ্নের উত্তর সরাসরি cache), semantic caching, ছোট model, prompt compression। এগুলো Phase 10 এ বিস্তারিত।

### When should I use it?
High-volume app, বড় system prompt/tools, document Q&A, লম্বা agent।

### When should I NOT use it?
Low traffic (TTL এর মধ্যে দ্বিতীয় request আসে না) আর প্রতিটা prompt unique হলে।

### Hands-on experiment
```ts
function dailyCost(o: { prefix: number; suffix: number; requests: number; hitRate: number; price: number }) {
  const per = (t: number) => (t / 1e6) * o.price;
  const noCache = o.requests * per(o.prefix + o.suffix);
  const withCache =
    o.requests * (o.hitRate * per(o.prefix * 0.1) + (1 - o.hitRate) * per(o.prefix * 1.25) + per(o.suffix));
  return { noCache, withCache, saving: 1 - withCache / noCache };
}
console.log(dailyCost({ prefix: 20_000, suffix: 500, requests: 10_000, hitRate: 0.9, price: 4 }));
```
`hitRate` 0.2, 0.5, 0.9 করে দেখো কোথায় caching লাভজনক থাকে না।

### Production implementation
- Dashboard: hit rate, write tokens, read tokens, cost saved (feature/model অনুযায়ী)।
- Alert: hit rate হঠাৎ কমলে (deploy এর পরে বিশেষ করে)।
- Prompt PR review checklist: "এই change কি cached prefix বদলায়?"

### Failure cases
- Low traffic feature এ 1h TTL দিয়ে প্রতিবার 2× write দিয়ে খরচ বাড়ানো।
- A/B test এ ১০টা prompt variant, প্রতিটার cache আলাদা, ফলে hit rate ভেঙে পড়া।

### Security concerns
Cost dashboard এ prompt content দেখাবে না, শুধু metric।

### Performance concerns
Caching cost এর সাথে latency ও কমায়, একই optimization এ দুই লাভ।

### Cost concerns
এই topic টাই cost। মূল সূত্র: **measure (usage fields) → structure (stable prefix) → tune (TTL, breakpoint) → monitor (hit rate)**।

### What did I learn?
- Caching হলো AI cost optimization এর প্রথম আর সবচেয়ে সস্তা ধাপ।

### What can I explain to someone else?
> "Monthly bus pass এর মতো: প্রতিদিন একই রুটে গেলে pass কেনা লাভ। মাসে একবার গেলে pass কেনা লস। আগে হিসাব করো কতবার যাবে।"

### GitHub Evidence
- [ ] Caching cost calculator + hit-rate dashboard mock

### LinkedIn Post
- [ ] "Prompt caching দিয়ে AI bill ৯০% কমানোর হিসাব"

---

⬅️ Back to [Phase 3 — Context Engineering](README.md) · [Root README](../README.md)

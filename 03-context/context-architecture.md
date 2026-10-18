# Phase 3 — Context Engineering: Context Architecture (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Code example গুলো TypeScript এ। Local practice এর জন্য Ollama (`qwen2.5:3b`), আর Claude-specific feature এর জন্য Anthropic TypeScript SDK (`@anthropic-ai/sdk`)। API detail সময়ের সাথে বদলায়, তাই সবসময় official docs দেখে নেবে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [System context](#system-context) | Role, rules, tool definitions: প্রতিটা request এর stable "মাথা" |
| [Conversation history](#conversation-history) | আগের user/assistant turn, app প্রতিবার resend করে (model stateless) |
| [Tool results](#tool-results) | Tool call এর output, context এর সবচেয়ে দ্রুত ফুলে ওঠা অংশ |
| [Retrieved documents](#retrieved-documents) | RAG দিয়ে আনা chunk: শুধু এই প্রশ্নের জন্য relevant knowledge |
| [User context](#user-context) | User এর profile, preference, permission, locale, সময় |
| [Memory](#memory) | Conversation এর বাইরে টিকে থাকা তথ্য, দরকারমতো context এ load হয় |
| [Context budgeting](#context-budgeting) | প্রতিটা অংশের জন্য token সীমা ঠিক করে দেওয়া |

```text
Context Engineering = "এই মুহূর্তে model এর সামনে ঠিক কী থাকবে" সেটা design করা

┌─────────────────────────── Context Window ───────────────────────────┐
│ [System context]  role · rules · tool definitions        stable       │
│ [Memory]          user facts · past decisions            ধীরে বদলায়   │
│ [User context]    profile · locale · permission · date   per user     │
│ [History]         আগের turn গুলো                          বাড়তে থাকে    │
│ [Retrieved docs]  RAG chunk                              per request  │
│ [Tool results]    API/DB/search output                   per step     │
│ [User message]    এখনকার প্রশ্ন                            সবচেয়ে শেষে  │
│ ► Output reserve                                                      │
└───────────────────────────────────────────────────────────────────────┘
```

> **Prompt engineering** = কীভাবে বলবে। **Context engineering** = model কী দেখবে, কতটুকু দেখবে, কোন order এ দেখবে।

---

<a id="system-context"></a>

## 1. System Context

### What is it?
Context এর সেই অংশ যেটা প্রায় প্রতিটা request এ একই থাকে: system prompt (role, rules, output format), tool definitions, স্থায়ী few-shot example। Phase 2 এ system prompt কে "কী লিখব" হিসেবে দেখেছি; এখানে দেখছি **context window এর একটা stable, reusable block** হিসেবে।

### Why does it exist?
Model stateless। Product এর rule আর tool এর তালিকা প্রতিটা call এ আবার পাঠাতে হয়। তাই এটা একটা আলাদা, স্থির layer হিসেবে design করা দরকার।

### What problem does it solve?
- প্রতিটা request এ consistent behavior।
- Stable prefix থাকায় **prompt caching** কাজ করে, ফলে cost আর TTFT কমে।
- Dynamic জিনিস থেকে আলাদা থাকায় debug ও version করা সহজ।

### How does it work?
```text
Request render order (Claude):  tools → system → messages
                                └────── stable prefix ──────┘ → cache হয়
```
Tool definition গুলোও input token। ২০টা tool এর লম্বা description হলে হাজার হাজার token প্রতিটা call এ যায়।

### Important concepts
- **Stable vs volatile**: system context এ কোনো timestamp, user ID, random order থাকবে না।
- **Tool definitions = context**: tool যত বেশি, model এর বাছাই তত কঠিন আর token তত বেশি।
- **Deterministic serialization**: tool list আর JSON key order প্রতিবার একই হতে হবে।
- **Progressive disclosure**: সব tool/instruction একবারে না দিয়ে দরকার হলে load করা (tool search, skills)।

### Trade-offs
বিস্তারিত system context = ভালো control, কিন্তু বেশি token আর rule conflict এর ঝুঁকি। ছোট = সস্তা, কিন্তু behavior অনিশ্চিত।

### Alternatives
Fine-tuning (behavior weight এ বসানো), dynamic tool loading, task অনুযায়ী আলাদা ছোট system prompt (routing)।

### When should I use it?
সব production feature এ। যা প্রতিটা request এ লাগে আর বদলায় না, সেটাই system context এ যায়।

### When should I NOT use it?
Per-request data (তারিখ, user নাম, retrieved doc) system context এ গুঁজবে না, কারণ এতে cache ভাঙে।

### Hands-on experiment
```ts
// একই system prompt, কিন্তু একটায় timestamp। দুটো ৩ বার করে চালিয়ে cache usage দেখো।
const stable = "You are Acme's support assistant. ...(লম্বা rules)...";
const volatile = `${stable}\nCurrent time: ${new Date().toISOString()}`;
// Anthropic SDK এ cache_control দিয়ে পাঠাও, তারপর
// usage.cache_read_input_tokens তুলনা করো: volatile এ সবসময় 0।
```

### Production implementation
- System prompt আর tool list file/config এ রাখো, version দাও।
- Tool list sort করে পাঠাও, যাতে order প্রতিবার একই থাকে।
- System context এর token size CI তে measure করো (budget পার হলে fail)।

### Failure cases
- ৪০টা tool দেওয়ায় model ভুল tool বাছছে।
- কেউ system prompt এ `Today is ...` যোগ করল, ফলে রাতারাতি cache hit শূন্য আর bill বেড়ে গেল।

### Security concerns
System context এ secret রাখবে না, কারণ leak হতে পারে। Untrusted content (user/document) কখনো system অংশে ঢোকাবে না।

### Performance concerns
লম্বা system context মানে বেশি prefill আর বেশি TTFT। Caching হলে এটা অনেক কমে।

### Cost concerns
প্রতিটা call এ bill হয়। 5K token system × 1M request/মাস = 5B input token, তাই caching ছাড়া এটা বড় খরচ।

### What did I learn?
- System context = context এর stable foundation। এটা যত স্থির, cache তত কাজ করে।
- Tool definitions ও context এর অংশ, free না।

### What can I explain to someone else?
> "System context হলো দোকানের সাইনবোর্ড আর নিয়মের খাতা। প্রতিদিন একই থাকে, তাই একবার ছাপিয়ে রাখা যায় (cache)। প্রতিদিন নতুন করে লিখলে খরচ আর সময় দুটোই বাড়ে।"

### GitHub Evidence
- [ ] Stable vs volatile system prompt cache test

### LinkedIn Post
- [ ] "System prompt এ একটা timestamp কীভাবে তোমার AI bill দ্বিগুণ করে"

---

<a id="conversation-history"></a>

## 2. Conversation History

### What is it?
আগের সব user ও assistant message (tool_use/tool_result সহ), যেটা app প্রতিটা নতুন request এ আবার পাঠায়।

### Why does it exist?
Model আগের call মনে রাখে না। Follow-up প্রশ্ন ("ওটা আরেকটু ছোট করো") বোঝাতে হলে আগের কথা context এ থাকতেই হবে।

### What problem does it solve?
Multi-turn conversation আর agent loop সম্ভব করে। Model বোঝে "ওটা", "আগেরটা", "তুমি যা বললে" বলতে কী বোঝানো হচ্ছে।

### How does it work?
```text
turn 1: [u1]                     → a1
turn 2: [u1, a1, u2]             → a2
turn 3: [u1, a1, u2, a2, u3]     → a3      ← প্রতি turn এ পুরো history আবার input
```
History DB তে append-only store হয়, request এর সময় budget অনুযায়ী select করা হয়।

### Important concepts
- **History grows linearly, total cost grows ~quadratically** (প্রতি turn এ আগের সব আবার bill হয়)।
- **পুরো content block রাখো**: tool_use, thinking block হারালে পরের request ভাঙতে পারে।
- **Append-only**: আগের message edit করলে cache ভাঙে, নতুন model এ thinking block invalid হতে পারে।
- **Trimming strategy**: sliding window, summarization, বা compaction (বিস্তারিত: [context-optimization.md](context-optimization.md))।

### Trade-offs
পুরো history = সবচেয়ে ভালো continuity, কিন্তু খরচ ও latency বাড়ে, আর lost-in-the-middle হয়। ছোট history = সস্তা, কিন্তু model "ভুলে যায়"।

### Alternatives
Summary + শেষ N turn, server-side compaction, memory store থেকে শুধু relevant fact আনা।

### When should I use it?
Chat, multi-turn assistant, agent loop: যেখানে আগের turn এর উপর পরের turn নির্ভর করে।

### When should I NOT use it?
Single-shot task (classify, extract) এ history পাঠাবে না। প্রতিটা request independent রাখো।

### Hands-on experiment
```ts
// npm i ollama
import ollama from "ollama";

const messages: { role: "user" | "assistant"; content: string }[] = [];
for (const q of ["আমার নাম রিমন", "আমি TypeScript শিখছি", "আমার নাম কী আর কী শিখছি?"]) {
  messages.push({ role: "user", content: q });
  const res = await ollama.chat({ model: "qwen2.5:3b", messages });
  messages.push(res.message as any);
  console.log(q, "→ prompt tokens:", res.prompt_eval_count);
}
```
প্রতি turn এ `prompt_eval_count` কীভাবে বাড়ছে দেখো। তারপর history বাদ দিয়ে শেষ প্রশ্নটা চালাও।

### Production implementation
- `messages` table: `conversation_id, seq, role, content_json, tokens, created_at`।
- Request build এর সময় token budget অনুযায়ী শেষ থেকে message নাও, আর পুরনো অংশের summary রাখো।
- History server-side রাখো, client এর পাঠানো history বিশ্বাস করবে না।

### Failure cases
- Sliding window দিয়ে কাটতে গিয়ে tool_use রয়ে গেল কিন্তু তার tool_result কেটে গেল, ফলে API 400।
- ৫০ turn পরে প্রথম turn এর গুরুত্বপূর্ণ constraint হারিয়ে গেছে।

### Security concerns
History tamper করলে নকল assistant message ঢোকানো যায়। আগের turn এ ঢোকা injection পরের সব turn এ থেকে যায়, তাই history ও untrusted ধরো।

### Performance concerns
History বাড়লে TTFT বাড়ে। Caching দিয়ে আগের অংশ দ্রুত হয়।

### Cost concerns
৩০ turn এর conversation এ শেষ turn এর input প্রথম turn এর ৩০ গুণ হতে পারে। Caching + trimming বাধ্যতামূলক।

### What did I learn?
- "Model মনে রাখে" আসলে app resend করে।
- History হলো context এর সবচেয়ে দ্রুত বাড়তে থাকা খরচ।

### What can I explain to someone else?
> "প্রতিটা প্রশ্নের সময় model কে পুরো chat এর screenshot দেখাতে হয়। Chat যত লম্বা, screenshot তত বড়, আর প্রতিবার পুরো screenshot এর টাকা দিতে হয়।"

### GitHub Evidence
- [ ] Token growth per turn chart (with vs without trimming)

### LinkedIn Post
- [ ] "Chat লম্বা হলে AI এর খরচ কেন quadratic হারে বাড়ে"

---

<a id="tool-results"></a>

## 3. Tool Results

### What is it?
Model একটা tool call করলে (search, DB query, API, file read) তোমার code যে output ফেরত দেয়, সেটা `tool_result` block হিসেবে context এ ঢোকে।

### Why does it exist?
Model নিজে real-time data বা action পায় না। Tool এর ফলাফল context এ না দিলে model সেটা ব্যবহার করতে পারবে না।

### What problem does it solve?
Fresh data, exact হিসাব, external system এর state, এগুলো model এর reasoning এর ভেতরে আনে।

### How does it work?
```text
assistant: tool_use {name: "search_orders", input: {...}}
user:      tool_result {tool_use_id, content: "[...৫০টা order JSON...]"}
assistant: "আপনার শেষ order ১২ তারিখে..."
```
প্রতিটা tool result পরের সব request এ history হিসেবে থেকে যায়।

### Important concepts
- **Agent context এর সবচেয়ে বড় অংশ সাধারণত tool result**। একটা web page বা log file হাজার হাজার token হতে পারে।
- **Shape the output**: raw API response না দিয়ে দরকারি field বেছে, ছোট format এ দাও।
- **Error হলেও result দাও**: `is_error: true` সহ, যাতে model বুঝে retry বা অন্য পথ নেয়।
- **Tool result = untrusted data**: ভেতরে instruction থাকতে পারে (indirect injection)।

### Trade-offs
পূর্ণ output = model সব দেখে, কিন্তু context ভরে যায়। ছোট summary = সস্তা, কিন্তু দরকারি তথ্য বাদ পড়তে পারে।

### Alternatives
Truncation, pagination ("আরও দেখতে page=2 call করো"), file এ রেখে শুধু path দেওয়া, context editing দিয়ে পুরনো result মুছে ফেলা।

### When should I use it?
যখন model এর জন্য external তথ্য বা action এর ফলাফল লাগে, অর্থাৎ প্রায় সব agent এ।

### When should I NOT use it?
পুরো raw dump (10K row CSV, পুরো HTML) সরাসরি tool result এ দেবে না।

### Hands-on experiment
একই "order খোঁজো" tool দুইভাবে implement করো: (১) পুরো API JSON ফেরত, (২) শুধু `id, date, status, total`। ১০টা প্রশ্নে accuracy আর input token তুলনা করো।

### Production implementation
```ts
function shapeToolResult(rows: Order[], limit = 20) {
  const shown = rows.slice(0, limit).map(({ id, date, status, total }) => ({ id, date, status, total }));
  return JSON.stringify({ shown, total_count: rows.length, truncated: rows.length > limit });
}
```
- প্রতিটা tool এর max output token ঠিক করো।
- Tool result এর token size log করো।

### Failure cases
- একটা log file tool result এ ঢোকায় context limit পার হয়ে গেল।
- Tool error চুপচাপ গিলে ফেলায় model ভাবল data খালি, ফলে ভুল উত্তর।

### Security concerns
Web page, email, ticket এর ভেতরের লুকানো instruction model মেনে ফেলতে পারে। System prompt এ বলে দাও tool result = data, আর sensitive action এ code-level permission check রাখো।

### Performance concerns
বড় tool result পরের প্রতিটা step এর prefill বাড়ায়।

### Cost concerns
Agent এর ১০ step এ একটা 5K token result ১০ বার bill হয় (প্রতি step এ history resend)।

### What did I learn?
- Tool design মানে শুধু function না, output এর **shape আর size** design করা।

### What can I explain to someone else?
> "Assistant কে report দিতে বললে সে পুরো ফাইল ক্যাবিনেট টেবিলে ঢেলে দিলে কাজ হয় না। দরকারি ৩টা কাগজ বেছে দিতে হয়।"

### GitHub Evidence
- [ ] Raw vs shaped tool output (accuracy + token table)

### LinkedIn Post
- [ ] "AI agent এর context কে খায়? Tool result"

---

<a id="retrieved-documents"></a>

## 4. Retrieved Documents

### What is it?
RAG pipeline এ vector/keyword search দিয়ে knowledge base থেকে আনা relevant chunk, যেটা user message এর সাথে context এ বসানো হয়।

### Why does it exist?
পুরো knowledge base context window এ ধরে না, আর ধরলেও দামি, ধীর ও কম accurate। তাই শুধু প্রশ্নের জন্য দরকারি অংশ আনা হয়।

### What problem does it solve?
Hallucination কমায়, private/fresh data ব্যবহার করা যায়, আর source citation দেওয়া যায়।

### How does it work?
```text
question → embed/search → top-k chunk → rerank → top-n
         → <context><doc id=1 source=...>...</doc></context><question>...</question>
```
বিস্তারিত Phase 4 ([04-rag](../04-rag/README.md)) এ।

### Important concepts
- **Precision > recall in context**: ২০টা আধা-relevant chunk এর চেয়ে ৫টা ভালো chunk ভালো।
- **Metadata সহ দাও**: source, title, date, যাতে model cite করতে পারে।
- **Order matters**: সবচেয়ে relevant chunk শুরুতে বা শেষে (lost-in-the-middle)।
- **Data boundary**: `<doc>` tag এ মুড়ে দাও, instruction হিসেবে না।

### Trade-offs
বেশি chunk = বেশি recall, কিন্তু noise, cost, latency বাড়ে। কম chunk = দরকারি তথ্য miss হতে পারে।

### Alternatives
Long-context এ পুরো document দেওয়া (ছোট corpus এ), fine-tuning (style এর জন্য, fact এর জন্য না), tool দিয়ে on-demand retrieval (agentic RAG)।

### When should I use it?
Knowledge base বড়, ঘন ঘন বদলায়, বা উত্তরের source দেখাতে হয়।

### When should I NOT use it?
Corpus খুব ছোট (কয়েক পাতা) হলে সরাসরি পুরোটা context এ দিলেই সহজ।

### Hands-on experiment
একই প্রশ্নে top-k = 3, 10, 30 দিয়ে উত্তর তুলনা করো: accuracy, token, latency। কোথায় accuracy আর বাড়ে না বরং কমে, দেখো।

### Production implementation
- Retrieved token budget fix করো (যেমন 6K)। Rerank করে budget এ যতটুকু ধরে ততটুকু।
- প্রতিটা chunk এর `doc_id` log করো, যাতে উত্তর কোন source থেকে এসেছে ট্রেস করা যায়।
- User এর permission অনুযায়ী filter করো (tenant/ACL)।

### Failure cases
- ভুল chunk retrieve হলে model আত্মবিশ্বাসের সাথে ভুল উত্তর দেয়।
- পুরনো version এর document আর নতুন version দুটোই এসে conflict।

### Security concerns
Document poisoning: knowledge base এ কেউ instruction লুকিয়ে রাখলে সেটা context এ আসে। আর ACL filter না থাকলে অন্য user এর data leak হয়।

### Performance concerns
Retrieval + rerank latency যোগ হয়। বেশি chunk মানে বেশি prefill।

### Cost concerns
Retrieved chunk প্রতি request এ নতুন, তাই cache হয় না। Budget ছোট রাখো।

### What did I learn?
- Retrieved docs = "এই প্রশ্নের জন্য just-in-time knowledge"। পরিমাণ না, মান গুরুত্বপূর্ণ।

### What can I explain to someone else?
> "পরীক্ষায় পুরো লাইব্রেরি নিয়ে যাওয়া যায় না। প্রশ্ন দেখে দরকারি ৩টা পাতা বের করে আনাই RAG।"

### GitHub Evidence
- [ ] top-k vs accuracy experiment

### LinkedIn Post
- [ ] "RAG এ বেশি chunk দিলে উত্তর কেন খারাপ হয়"

---

<a id="user-context"></a>

## 5. User Context

### What is it?
নির্দিষ্ট user সম্পর্কে তথ্য, যা উত্তর personalize ও সঠিক করে: নাম, ভাষা, timezone, plan/role, permission, preference, আজকের তারিখ, current page বা app state।

### Why does it exist?
একই প্রশ্নের সঠিক উত্তর user ভেদে আলাদা ("আমার bill কত?", "কাল meeting কখন?")। Model এটা নিজে জানে না।

### What problem does it solve?
Personalization, সঠিক সময়/তারিখ হিসাব, role অনুযায়ী উত্তর, আর বারবার একই তথ্য user কে জিজ্ঞেস না করা।

### How does it work?
```text
<user_context>
  name: Rimon · locale: bn-BD · timezone: Asia/Dhaka
  plan: pro · role: admin · today: 2026-10-15
</user_context>
```
সাধারণত user message এর শুরুতে, system prompt এর পরে বসানো হয় (cache বাঁচাতে)।

### Important concepts
- **Minimal & relevant**: পুরো user profile না, এই feature এ যা লাগে।
- **Server-side source of truth**: role/permission DB থেকে আসবে, user এর দাবি থেকে না।
- **Volatile তাই system এ না**: তারিখ, নাম system prompt এ দিলে cache ভাঙে।
- **Privacy**: PII কম পাঠাও।

### Trade-offs
বেশি personal context = ভালো personalization, কিন্তু privacy risk, token খরচ, আর অপ্রাসঙ্গিক তথ্যে model বিভ্রান্ত হতে পারে।

### Alternatives
Tool দিয়ে on-demand ("get_user_profile"), memory system থেকে relevant fact retrieve।

### When should I use it?
যেখানে উত্তর user/সময়/permission এর উপর নির্ভর করে।

### When should I NOT use it?
Sensitive data (password, পুরো card number, health record) প্রয়োজন ছাড়া context এ দেবে না।

### Hands-on experiment
"পরশু কী বার?" প্রশ্নটা `today` সহ ও ছাড়া জিজ্ঞেস করো। Timezone ছাড়া "সকাল ৯টায় reminder" প্রশ্নও করো।

### Production implementation
```ts
function userContextBlock(u: User) {
  return `<user_context>\nlocale: ${u.locale}\ntimezone: ${u.tz}\nrole: ${u.role}\ntoday: ${todayIn(u.tz)}\n</user_context>`;
}
```
Permission check কিন্তু code এ করো। Context এ role বলা শুধু model এর tone/উত্তরের জন্য।

### Failure cases
- Server UTC তারিখ দেওয়ায় বাংলাদেশের user এর "আজ" ভুল।
- Context এ `role: admin` লেখা থাকায় model admin-only তথ্য বলে দিল, অথচ code এ check ছিল না।

### Security concerns
User context এ যা থাকে model তা বলে দিতে পারে, তাই অন্য user এর data কখনো না। Role কে authorization হিসেবে ব্যবহার করবে না।

### Performance concerns
নগণ্য, সাধারণত কয়েকশো token।

### Cost concerns
ছোট, কিন্তু system prompt এ রাখলে cache ভেঙে বড় খরচ।

### What did I learn?
- User context ছোট, প্রাসঙ্গিক, server-verified, আর system prompt এর বাইরে থাকা উচিত।

### What can I explain to someone else?
> "Receptionist কে customer এর নাম, ভাষা আর appointment জানিয়ে দিলে সে ভালো সেবা দেয়। কিন্তু vault এর চাবি (permission) তার হাতে না, security গার্ডের (code) হাতে।"

### GitHub Evidence
- [ ] Date/timezone with vs without user context test

### LinkedIn Post
- [ ] "AI কে আজকের তারিখ না বললে কী হয়"

---

<a id="memory"></a>

## 6. Memory

### What is it?
এক conversation বা session এর বাইরে টিকে থাকা তথ্য (user preference, আগের সিদ্ধান্ত, শেখা fact), যা দরকার হলে context এ load হয়। এখানে context architecture এ এর **জায়গা** নিয়ে কথা। ধরন ও design নিয়ে বিস্তারিত: [memory.md](memory.md)।

### Why does it exist?
Context window প্রতি request এ খালি হয়ে যায়, আর conversation শেষ হলে history ও শেষ। দীর্ঘমেয়াদি continuity এর জন্য বাইরের store লাগে।

### What problem does it solve?
"আমি আগেও বলেছি বাংলায় উত্তর দিতে।" এই frustration দূর করে, আর personalization ও agent এর অভিজ্ঞতা জমা হয়।

### How does it work?
```text
write path: conversation → extract fact → store (DB/vector/file)
read path : new request → retrieve relevant memory → <memory>...</memory> context এ
```

### Important concepts
- **Memory ≠ context window**: context = working memory (RAM), memory store = disk।
- **Selective loading**: সব memory না, শুধু relevant অংশ।
- **Write policy**: কী মনে রাখবে, কখন update/delete করবে।
- **Memory tool**: Claude এ `memory_20250818` tool দিয়ে model নিজে file-based memory পড়ে/লেখে (storage তোমার)।

### Trade-offs
বেশি memory = ভালো personalization, কিন্তু ভুল/পুরনো fact, privacy risk, আর retrieval এর জটিলতা।

### Alternatives
পুরো history রেখে দেওয়া (ছোট app এ), user এর হাতে explicit settings (preference form)।

### When should I use it?
Long-term assistant, multi-session agent, personalized product।

### When should I NOT use it?
One-shot tool, বা যেখানে user privacy/stateless থাকা বেশি জরুরি।

### Hands-on experiment
দুইটা আলাদা session চালাও। প্রথমটায় preference বলো, একটা JSON file এ সেটা save করো, দ্বিতীয় session এ load করে দেখো model মানছে কিনা।

### Production implementation
- Memory কে context এ আলাদা `<memory>` block এ দাও, budget সহ।
- প্রতিটা memory তে `source, created_at, confidence` রাখো।
- User কে memory দেখার/মুছে ফেলার উপায় দাও।

### Failure cases
- পুরনো preference ("আমি Java শিখছি") নতুনটাকে override করছে।
- Model ভুল fact memory তে লিখে ফেলেছে, এখন প্রতিটা session এ ভুল।

### Security concerns
Memory poisoning: injection দিয়ে memory তে ক্ষতিকর instruction লেখা যায়, যা ভবিষ্যতের সব session এ চলে যায়। Memory write validate করো।

### Performance concerns
Memory retrieval এর জন্য extra DB/vector query।

### Cost concerns
Memory extract করতে extra LLM call লাগতে পারে। Batch বা async এ করো।

### What did I learn?
- Context window = short-term, memory store = long-term। দুটোর মাঝে bridge হলো retrieval।

### What can I explain to someone else?
> "Context window হলো টেবিল, memory হলো আলমারি। দরকারি ফাইল আলমারি থেকে টেবিলে আনো, কাজ শেষে নতুন কিছু শিখলে আলমারিতে রেখে দাও।"

### GitHub Evidence
- [ ] Cross-session preference memory demo

### LinkedIn Post
- [ ] "Context window আর memory এক জিনিস না"

---

<a id="context-budgeting"></a>

## 7. Context Budgeting

### What is it?
Context window কে অংশে ভাগ করে প্রতিটার জন্য token সীমা ঠিক করা, যাতে কোনো অংশ বাকিদের জায়গা খেয়ে না ফেলে আর output এর জন্য জায়গা থাকে।

### Why does it exist?
Context window সীমিত, আর বড় window থাকলেও বেশি token মানে বেশি খরচ, latency, আর কম accuracy। Budget না থাকলে একটা বড় tool result বা লম্বা history পুরো request ভেঙে দেয়।

### What problem does it solve?
Predictable cost, latency আর quality, আর "context limit exceeded" error থেকে রক্ষা।

### How does it work?
```text
Total target: 32K (model window 1M হলেও!)
├─ System + tools   3K   (fixed)
├─ Memory           1K
├─ User context     0.5K
├─ History          10K  (overflow → summarize)
├─ Retrieved docs   8K   (overflow → কম chunk)
├─ Tool results     6K   (overflow → truncate)
├─ User message     1.5K (overflow → reject/trim)
└─ Output reserve   2K+
```

### Important concepts
- **Hard limit vs soft target**: model এর limit hard; তোমার quality/cost target soft কিন্তু বেশি গুরুত্বপূর্ণ।
- **Priority order**: overflow হলে কোনটা আগে কাটবে (সাধারণত পুরনো history → কম relevant docs → tool output)।
- **Exact counting**: limit এর কাছাকাছি হলে model এর token counting API ব্যবহার করো।
- **Output reserve**: `max_tokens` + thinking এর জন্য জায়গা।

### Trade-offs
Strict budget = predictable ও সস্তা, কিন্তু কখনো দরকারি তথ্য বাদ পড়ে। Loose = flexible, কিন্তু cost/latency spike।

### Alternatives
Server-side compaction (API নিজে summarize করে), context editing (পুরনো tool result clear), বড় window এর উপর ভরসা (কাজ করে, কিন্তু দামি)।

### When should I use it?
যেখানে context এর অংশ dynamic: chat, RAG, agent।

### When should I NOT use it?
Fixed-size single prompt (যেমন একটা ছোট classification) এ আলাদা budgeting layer লাগে না।

### Hands-on experiment
```ts
type Part = { name: string; text: string; budget: number };

const approxTokens = (s: string) => Math.ceil(s.length / 4); // rough; বাংলায় আরও বেশি

function fit(parts: Part[]) {
  return parts.map((p) => {
    const t = approxTokens(p.text);
    const text = t <= p.budget ? p.text : p.text.slice(-p.budget * 4); // শেষের অংশ রাখো
    return { ...p, used: Math.min(t, p.budget), trimmed: t > p.budget, text };
  });
}
```
তারপর approximate count আর real `usage.input_tokens` এর পার্থক্য দেখো (বাংলা text এ বিশেষ করে)।

### Production implementation
- `ContextBuilder` class: প্রতিটা part এর budget, priority, overflow strategy।
- প্রতিটা request এ part-wise token breakdown log করো, dashboard এ দেখাও।
- Budget config এ রাখো, feature অনুযায়ী আলাদা।

### Failure cases
- Character দিয়ে count করায় বাংলা text budget ২-৩ গুণ পার হয়ে গেল।
- Output reserve না রাখায় JSON মাঝপথে কেটে গেল।

### Security concerns
User input এর size limit না থাকলে কেউ বিশাল input দিয়ে বাকি সব context (system rules সহ) এর প্রভাব কমিয়ে দিতে পারে বা denial-of-wallet করতে পারে।

### Performance concerns
ছোট context = দ্রুত TTFT। Budget latency control এর হাতিয়ারও।

### Cost concerns
Budget = cost ceiling। প্রতি request এর সর্বোচ্চ খরচ আগেই জানা যায়।

### What did I learn?
- 1M window থাকলেও budget দরকার। "পারে" আর "উচিত" আলাদা।

### What can I explain to someone else?
> "মাসের বাজেটের মতো: বাসা ভাড়া, খাবার, যাতায়াত সবার আলাদা ভাগ। একটা খাতে বেশি খরচ হলে অন্যটা থেকে কাটতে হয়, আর শেষে কিছু সঞ্চয় (output reserve) রাখতে হয়।"

### GitHub Evidence
- [ ] `ContextBuilder` with per-part budget + token breakdown log

### LinkedIn Post
- [ ] "1M token window থাকলেও কেন context budget লাগে"

---

⬅️ Back to [Phase 3 — Context Engineering](README.md) · [Root README](../README.md)

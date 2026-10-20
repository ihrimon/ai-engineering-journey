# Phase 3 — Context Engineering: Context Optimization (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Code example গুলো TypeScript এ (local: Ollama `qwen2.5:3b`)। Number গুলো **উদাহরণ**, নিজের data তে measure করবে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Token budgeting](#token-budgeting) | প্রতিটা request/feature/user এর token ও cost সীমা runtime এ enforce করা |
| [Summarization](#summarization) | পুরনো অংশকে ছোট summary তে রূপান্তর করে জায়গা খালি করা |
| [Rolling memory](#rolling-memory) | শেষ N turn হুবহু + তার আগের সব একটা চলমান summary তে |
| [Compression](#compression) | একই অর্থ কম token এ: অপ্রয়োজনীয় শব্দ, whitespace, field বাদ |
| [Pruning](#pruning) | অপ্রাসঙ্গিক বা পুরনো অংশ পুরোপুরি বাদ দেওয়া |
| [Tool result truncation](#tool-result-truncation) | বড় tool output কেটে ছোট করা, সাথে "truncated" সংকেত |
| [Lost-in-the-middle](#lost-in-the-middle) | লম্বা context এর মাঝের তথ্য model কম ব্যবহার করে |
| [Context ordering](#context-ordering) | Stable আগে, volatile পরে, জরুরি তথ্য শুরু বা শেষে |

```text
             Context বড় হয়ে গেছে, কী করব?
                         │
   ┌─────────────┬───────┴───────┬──────────────┐
 বাদ দাও       ছোট করো         সাজাও          সীমা দাও
 Pruning      Summarization   Ordering        Budgeting
 Truncation   Compression     (lost-in-the-   (enforce)
              Rolling memory   middle এড়াও)
```

---

<a id="token-budgeting"></a>

## 1. Token Budgeting

### What is it?
[Context budgeting](context-architecture.md#context-budgeting) এ দেখেছি context কে ভাগ করা। Token budgeting হলো এর **runtime enforcement**: প্রতি request, প্রতি conversation, প্রতি user/tenant এর token আর cost সীমা গোনা ও মানানো।

### Why does it exist?
LLM এর খরচ usage-based আর unpredictable। একটা লম্বা conversation বা থামতে না পারা agent loop একাই বিশাল bill বানাতে পারে।

### What problem does it solve?
Cost predictability, abuse রোধ (denial-of-wallet), আর context overflow error এড়ানো।

### How does it work?
```text
request আসার আগে:  estimate(input) + max_tokens ≤ request_budget ?
request এর পরে:    usage যোগ করো → user_daily_used, conversation_used
সীমা পার হলে:      summarize / ছোট model / reject / "আজকের limit শেষ"
```

### Important concepts
- **Levels**: per-request, per-conversation, per-user/day, per-tenant/month।
- **Pre-check (estimate) + post-accounting (real `usage`)**।
- **Agent budget**: step count limit + total token limit। Claude এ `task_budget` (beta) দিয়ে model কেও budget জানানো যায়, যাতে সে নিজে pace করে।
- **`max_tokens` সবসময় set করো**।

### Trade-offs
কঠোর budget = নিরাপদ খরচ, কিন্তু power user এর অভিজ্ঞতা খারাপ। নরম = ভালো UX, কিন্তু খরচ ঝুঁকি।

### Alternatives
Rate limiting (request সংখ্যা), plan-based quota, model routing (budget কম হলে ছোট model)।

### When should I use it?
সব production app এ, বিশেষ করে public/free tier ও agent এ।

### When should I NOT use it?
Internal one-off script এ জটিল quota system বানাবে না। শুধু `max_tokens` যথেষ্ট।

### Hands-on experiment
```ts
const budget = { perConversation: 20_000, used: 0 };

async function guardedChat(messages: any[]) {
  if (budget.used >= budget.perConversation) throw new Error("conversation budget exceeded");
  const res = await ollama.chat({ model: "qwen2.5:3b", messages, options: { num_predict: 512 } });
  budget.used += (res.prompt_eval_count ?? 0) + (res.eval_count ?? 0);
  return res;
}
```
৩০ turn চালিয়ে দেখো কোন turn এ budget শেষ হয়।

### Production implementation
- Redis এ `tokens:{user}:{date}` counter, atomic increment।
- Budget এর ৮০% এ warning, ১০০% এ graceful degrade (summary/ছোট model)।
- Admin dashboard এ top spender user/feature।

### Failure cases
- শুধু input গুনে output ভুলে গেছে, তাই আসল খরচ ৩ গুণ।
- Agent loop এ step limit নেই, ফলে রাতভর চলে বিশাল bill।

### Security concerns
Budget না থাকলে denial-of-wallet attack সহজ। Budget server-side enforce করো, client এর উপর না।

### Performance concerns
Counter check এ সামান্য latency (Redis এ ~১ms)।

### Cost concerns
এটাই মূল cost control। "Measure → limit → alert"।

### What did I learn?
- Budget planning (design time) আর budget enforcement (runtime) দুটোই লাগে।

### What can I explain to someone else?
> "Mobile data pack এর মতো: প্রতিদিন কত GB ব্যবহার করা যাবে তার সীমা। শেষ হলে speed কমে যায় (ছোট model) বা বন্ধ হয়, যাতে মাস শেষে বিলে চমক না আসে।"

### GitHub Evidence
- [ ] Per-user daily token quota middleware

### LinkedIn Post
- [ ] "AI agent এর জন্য budget কেন seatbelt"

---

<a id="summarization"></a>

## 2. Summarization

### What is it?
Context এর পুরনো বা লম্বা অংশ (history, document, tool output) কে LLM দিয়ে ছোট summary তে রূপান্তর করা, তারপর মূল অংশের বদলে summary রাখা।

### Why does it exist?
History বাড়তেই থাকে, কিন্তু পুরনো turn এর প্রতিটা শব্দ দরকার নেই। মূল fact, সিদ্ধান্ত আর open প্রশ্নগুলোই দরকার।

### What problem does it solve?
লম্বা conversation/agent কে context limit এর মধ্যে রাখে, আর খরচ ও latency কমায়।

### How does it work?
```text
[t1..t40 (30K token)] ──LLM summarize──► [summary 1.5K] + [t36..t40 হুবহু]
```
Claude এ **server-side compaction** (beta) আছে: threshold এ পৌঁছালে API নিজেই আগের context summarize করে compaction block দেয়। তখন পুরো `response.content` append করতে হয়।

### Important concepts
- **কী রাখবে বলে দাও**: user goal, সিদ্ধান্ত, constraint, নাম/সংখ্যা/ID, unresolved প্রশ্ন।
- **Lossy**: summary তে কিছু না কিছু হারায়, বিশেষ করে exact সংখ্যা ও quote।
- **Structured summary**: free text এর চেয়ে section-wise (Goals / Decisions / Facts / Open) ভালো।
- **Trigger**: token threshold (যেমন budget এর ৭০%)।

### Trade-offs
Summary = কম token, কিন্তু extra LLM call এর খরচ ও latency, আর তথ্য হারানোর ঝুঁকি।

### Alternatives
Pruning (সরাসরি বাদ), rolling window, memory store এ fact extract, server-side compaction।

### When should I use it?
লম্বা chat, লম্বা agent run, লম্বা document থেকে কাজ যেখানে detail কম জরুরি।

### When should I NOT use it?
Legal/financial context যেখানে exact শব্দ জরুরি। সেখানে retrieval দিয়ে মূল টেক্সট আনো।

### Hands-on experiment
```ts
const SUMMARY_PROMPT = `Summarize the conversation below for a future assistant.
Keep: user goals, decisions, constraints, names, numbers, IDs, open questions.
Format:
## Goals
## Decisions
## Facts
## Open questions`;

async function summarize(history: string) {
  const res = await ollama.chat({
    model: "qwen2.5:3b",
    messages: [{ role: "system", content: SUMMARY_PROMPT }, { role: "user", content: history }],
  });
  return res.message.content;
}
```
২০ turn এর একটা conversation summarize করো, তারপর পুরনো detail নিয়ে ৫টা প্রশ্ন করো: summary দিয়ে কয়টার উত্তর ঠিক আসে?

### Production implementation
- Async summarization: user এর response এর পরে background job এ।
- Summary version সহ DB তে রাখো (`summary_upto_seq`)।
- Claude ব্যবহার করলে compaction beta (`compact-2026-01-12`) বিবেচনা করো।

### Failure cases
- Summary তে order number হারিয়ে গেছে, তাই user কে আবার জিজ্ঞেস করতে হচ্ছে।
- Summary এর summary এর summary, ফলে ধীরে ধীরে মূল অর্থ বিকৃত (drift)।

### Security concerns
History তে থাকা injection summary তে "instruction" হিসেবে ঢুকে স্থায়ী হতে পারে। Summary prompt এ বলো: শুধু fact, instruction না।

### Performance concerns
Summarize call এর latency, তাই sync path এ না রেখে background এ করো।

### Cost concerns
Summarization নিজেও token খরচ করে। ছোট/সস্তা model দিয়ে করো। লাভ আসে পরের সব turn এ।

### What did I learn?
- Summarization = context এর "zip", কিন্তু lossy zip।

### What can I explain to someone else?
> "দুই ঘণ্টার meeting এর পুরো recording না শুনে শুধু minutes পড়ো: সিদ্ধান্ত, কাজ, কে কী করবে। বাকি সব ছেঁটে ফেলা হয়েছে।"

### GitHub Evidence
- [ ] Structured summary vs raw history recall test

### LinkedIn Post
- [ ] "লম্বা AI chat কে ছোট করার কৌশল: summarization"

---

<a id="rolling-memory"></a>

## 3. Rolling Memory

### What is it?
একটা pattern: **শেষ N turn হুবহু** রাখো, আর তার আগের সব কিছু একটা **চলমান summary** তে ভাঁজ করে রাখো। নতুন turn আসলে সবচেয়ে পুরনো হুবহু turn summary তে মিশে যায়।

### Why does it exist?
শুধু sliding window এ পুরনো তথ্য পুরো হারায়। শুধু summary তে সাম্প্রতিক detail হারায়। Rolling memory দুটোর মাঝামাঝি।

### What problem does it solve?
Context size প্রায় constant থাকে, অথচ conversation এর শুরু থেকে শেষ পর্যন্ত মূল তথ্য টিকে থাকে।

### How does it work?
```text
[running summary] + [t(n-5) .. t(n) হুবহু] + [new user msg]
           ▲
   window পূর্ণ হলে সবচেয়ে পুরনো turn → summary তে merge (incremental)
```

### Important concepts
- **Incremental update**: পুরো history আবার summarize না করে "পুরনো summary + নতুন বাদ পড়া turn → নতুন summary"।
- **Window size N**: সাধারণত ৪–১০ turn, task অনুযায়ী।
- **Pair integrity**: tool_use আর tool_result একসাথে সরাও।
- **Batch rolling**: প্রতি turn এ না, প্রতি ৫ turn এ একবার roll করলে cache ভালো থাকে।

### Trade-offs
Constant cost আর long continuity, কিন্তু summary update এর extra call, আর summary drift এর ঝুঁকি। প্রতি turn এ summary বদলালে prefix বদলায়, ফলে cache miss।

### Alternatives
Pure sliding window (সহজ), full history + caching (ছোট conversation এ), memory store + retrieval।

### When should I use it?
লম্বা চলমান chat assistant, support bot, coaching app।

### When should I NOT use it?
ছোট conversation (১০ turn এর কম) এ লাগবে না। Full history + caching ই সস্তা ও ভালো।

### Hands-on experiment
```ts
const WINDOW = 6;
let summary = "";
let recent: Msg[] = [];

async function addTurn(msg: Msg) {
  recent.push(msg);
  if (recent.length > WINDOW + 4) {                 // batch এ roll করো
    const old = recent.splice(0, recent.length - WINDOW);
    summary = await summarize(`Previous summary:\n${summary}\n\nNew turns:\n${render(old)}`);
  }
}

const buildMessages = (q: string) => [
  { role: "system", content: `${SYSTEM}\n<conversation_summary>${summary}</conversation_summary>` },
  ...recent,
  { role: "user", content: q },
];
```
(লক্ষ করো: summary system এ দিলে roll এর সময় cache ভাঙে, তাই production এ এটা প্রথম user message এ রাখাই ভালো।)

### Production implementation
- `conversation_summaries(conversation_id, upto_seq, summary, version)`।
- Roll background এ, threshold based (batch)।
- Summary এর token size নিজেও cap করো।

### Failure cases
- Tool_use window এ রইল কিন্তু tool_result summary তে চলে গেল, ফলে 400 error।
- প্রতিটা roll এ ছোট ছোট তথ্য হারিয়ে ৫০ turn পরে user এর মূল goal ই নেই।

### Security concerns
Summary persistent, তাই একবার ঢোকা injection দীর্ঘস্থায়ী হয়। Summary কে data হিসেবে tag করো।

### Performance concerns
Context প্রায় constant, ফলে TTFT stable।

### Cost concerns
Per-turn input প্রায় constant, quadratic growth বন্ধ। Summary call এর খরচ batch করলে কম।

### What did I learn?
- "সাম্প্রতিক = হুবহু, পুরনো = সারাংশ"। মানুষের স্মৃতিও এভাবেই কাজ করে।

### What can I explain to someone else?
> "আজকের আলোচনা তোমার হুবহু মনে আছে, গত মাসের শুধু মূল কথা। Rolling memory AI কে এভাবেই মনে রাখতে শেখায়।"

### GitHub Evidence
- [ ] Rolling memory chat: token per turn chart (flat line)

### LinkedIn Post
- [ ] "Rolling memory: AI chat কে অনন্তকাল চালানোর কৌশল"

---

<a id="compression"></a>

## 4. Compression

### What is it?
একই তথ্য কম token এ প্রকাশ করা, অর্থ হারানো ছাড়া (বা খুব কম হারিয়ে)। যেমন whitespace/boilerplate বাদ, verbose JSON থেকে compact format, HTML থেকে plain text/markdown, অপ্রয়োজনীয় field বাদ, বা LLMLingua-র মতো tool দিয়ে কম-গুরুত্বপূর্ণ token বাদ।

### Why does it exist?
অনেক context এ "noise" থাকে: HTML tag, pretty-printed JSON এর space, repeated key, boilerplate footer। এগুলো token খায় কিন্তু তথ্য দেয় না।

### What problem does it solve?
Summarization এর মতো তথ্য না হারিয়ে (বা কম হারিয়ে) token কমায়, আর প্রায়ই কোনো LLM call লাগে না।

### How does it work?
```text
HTML page (12K token) → readability extract → markdown (2K)
[{"id":1,"name":"A","status":"ok"}, ...] (JSON) → CSV header + rows (~40% কম)
"Please kindly note that..." → সরাসরি কথা
```

### Important concepts
- **Lossless-ish (deterministic)**: format change, whitespace, field select। নিরাপদ।
- **Lossy (model-based)**: token pruning (LLMLingua), abstractive summary। সতর্ক থাকতে হয়।
- **Tokenizer-aware**: কোন format এ কম token, সেটা measure করে দেখো (অনুমান না)।
- **Readability**: খুব বেশি compress করলে model এর বুঝতে কষ্ট হয়, accuracy কমে।

### Trade-offs
কম token বনাম readability/accuracy। আক্রমণাত্মক compression এ model ভুল পড়ে।

### Alternatives
Summarization, pruning, retrieval (শুধু relevant অংশ আনা)।

### When should I use it?
Web page, HTML email, বড় JSON API response, log, এগুলো context এ দেওয়ার আগে।

### When should I NOT use it?
Code, legal text, exact quote যেখানে প্রতিটা character গুরুত্বপূর্ণ।

### Hands-on experiment
একই ২০টা record তিন format এ token count করো (pretty JSON, minified JSON, CSV/markdown table)। তারপর প্রতিটা format এ ৫টা প্রশ্নের accuracy দেখো।

### Production implementation
```ts
const compact = (rows: Record<string, unknown>[]) => {
  const keys = Object.keys(rows[0] ?? {});
  return [keys.join(","), ...rows.map((r) => keys.map((k) => String(r[k] ?? "")).join(","))].join("\n");
};
```
- Tool output pipeline: fetch → clean (HTML→md) → select fields → compact।

### Failure cases
- CSV তে value এর ভেতর comma থাকায় column ভেঙে গেছে, ফলে model ভুল পড়ছে।
- Aggressive token pruning এ negation ("not") বাদ পড়ে অর্থ উল্টে গেছে।

### Security concerns
HTML থেকে text বের করার সময় hidden text (white-on-white, `display:none`) তে লুকানো injection থাকতে পারে। Clean করার সময় এগুলোও বাদ দাও।

### Performance concerns
Deterministic compression প্রায় free। Model-based compression এ extra latency।

### Cost concerns
প্রায়ই ৩০–৮০% input কমে, বিশেষ করে HTML ও JSON এ।

### What did I learn?
- সবচেয়ে সস্তা optimization: model এ পাঠানোর আগে data পরিষ্কার করা।

### What can I explain to someone else?
> "ব্যাগ গোছানোর সময় জামা ভাঁজ করে রাখলে একই জামা কম জায়গায় ধরে। কিছু ফেলে দিতে হয় না, শুধু গুছিয়ে রাখা।"

### GitHub Evidence
- [ ] JSON vs CSV vs Markdown token + accuracy table

### LinkedIn Post
- [ ] "JSON কে CSV বানিয়ে AI bill ৪০% কমানো"

---

<a id="pruning"></a>

## 5. Pruning

### What is it?
Context থেকে অপ্রাসঙ্গিক, পুরনো বা duplicate অংশ **পুরোপুরি বাদ** দেওয়া: পুরনো tool result, শেষ হয়ে যাওয়া sub-task এর detail, পুরনো thinking block, irrelevant retrieved chunk।

### Why does it exist?
Agent চলতে চলতে context এ এমন অনেক কিছু জমে যা আর লাগবে না, যেমন ২০ step আগের একটা search result।

### What problem does it solve?
Summarize এর খরচ ছাড়াই জায়গা খালি করে, আর noise কমায় বলে model ফোকাস রাখে।

### How does it work?
```text
step 1 search_result (8K)   → step 15 এ: "[cleared — older tool result]"
step 2 file_read (5K)       → cleared
...
step 14 tool_result         → রাখো (সাম্প্রতিক)
```
Claude এ **context editing** (beta `context-management-2025-06-27`): `clear_tool_uses_20250919` পুরনো tool result নিজে clear করে, `clear_thinking_20251015` পুরনো thinking block clear করে।

### Important concepts
- **Relevance-based** (embedding/score কম হলে বাদ) vs **recency-based** (পুরনো হলে বাদ)।
- **Placeholder রাখো**: পুরো বাদ না দিয়ে "[cleared]" লিখে রাখলে model বোঝে কিছু ছিল, দরকার হলে আবার tool call করবে।
- **Dedup**: একই file দুইবার পড়া হলে পুরনোটা বাদ।
- **Cache impact**: মাঝখান থেকে কিছু বাদ দিলে তার পরের cache ভাঙে, তাই batch এ prune করো।

### Trade-offs
সহজ ও সস্তা, কিন্তু যা বাদ দিলে সেটা পরে লাগলে আবার tool call (latency) লাগবে, বা ভুল হবে।

### Alternatives
Summarization (তথ্য রেখে ছোট), external memory/file এ রেখে দেওয়া।

### When should I use it?
লম্বা agent loop, coding agent, research agent যেখানে tool output বেশি।

### When should I NOT use it?
যে তথ্য পুরো task জুড়ে লাগে (requirement, user constraint), সেটা prune করবে না।

### Hands-on experiment
একটা ২০ step এর agent simulate করো। (১) সব রাখো, (২) শেষ ৩টা বাদে সব tool result `[cleared]` করো। Token আর final answer quality তুলনা করো।

### Production implementation
```ts
function pruneToolResults(messages: Msg[], keepLast = 3) {
  const idx = messages.flatMap((m, i) => (hasToolResult(m) ? [i] : []));
  const clear = new Set(idx.slice(0, -keepLast));
  return messages.map((m, i) => (clear.has(i) ? replaceToolResultContent(m, "[cleared: older tool result]") : m));
}
```
- Claude এ নিজে না লিখে context editing beta ব্যবহার করা যায়।

### Failure cases
- User এর মূল requirement এর tool result prune হয়ে গেছে, তাই agent ভুল দিকে চলে গেল।
- Prune করার পর একই file বারবার পড়ছে (loop)।

### Security concerns
Pruning আসলে security তেও সাহায্য করে: পুরনো, injection-যুক্ত tool result context থেকে বের হয়ে যায়।

### Performance concerns
কম context = দ্রুত। কিন্তু অতিরিক্ত prune এ re-fetch এর latency।

### Cost concerns
Agent এর খরচ অনেক কমে। তবে প্রতি turn এ prune করলে cache write বাড়ে।

### What did I learn?
- সব কিছু মনে রাখার দরকার নেই। ভুলে যাওয়াও design এর অংশ।

### What can I explain to someone else?
> "রান্না করতে করতে ব্যবহার শেষ হওয়া বাসন সিংকে সরিয়ে রাখো, তাহলে কাউন্টারে জায়গা থাকে। দরকার হলে আবার বের করা যাবে।"

### GitHub Evidence
- [ ] Agent with/without tool-result pruning (token + success rate)

### LinkedIn Post
- [ ] "AI agent কে ভুলতে শেখানো কেন জরুরি"

---

<a id="tool-result-truncation"></a>

## 6. Tool Result Truncation

### What is it?
Tool output একটা নির্দিষ্ট token/character সীমার বেশি হলে কেটে ছোট করা, আর সাথে বলে দেওয়া যে কাটা হয়েছে এবং বাকিটা কীভাবে পাওয়া যাবে।

### Why does it exist?
একটা log, web page বা DB query এক call এ ৫০K token ফেরত দিতে পারে। এতে context ভরে যায়, খরচ বাড়ে, model দিশা হারায়।

### What problem does it solve?
একটা tool call যেন পুরো context ধ্বংস না করে। Worst-case size bounded থাকে।

### How does it work?
```text
output 40K token → head 2K + "...[38K truncated; use offset=2000 to read more]..." + tail 500
```

### Important concepts
- **Head + tail**: log এ শেষটা (error) জরুরি, document এ শুরুটা।
- **Explicit marker**: model কে জানাও যে কাটা হয়েছে, নাহলে সে ভাববে এটাই পুরোটা।
- **Pagination / offset parameter**: model নিজে বাকিটা চাইতে পারে।
- **Structured truncation**: list এ প্রথম N item + `total_count`।
- **Token-based limit**, character based না।

### Trade-offs
Bounded context, কিন্তু দরকারি অংশ কাটা পড়তে পারে, আর বাকিটা আনতে extra call।

### Alternatives
Tool এর ভেতরেই filter/search parameter (grep, query), result file এ save করে path দেওয়া, summarization।

### When should I use it?
সব tool এর জন্য default safety net হিসেবে।

### When should I NOT use it?
যেখানে partial data ভুল সিদ্ধান্তে নেয় (যেমন হিসাবের জন্য পুরো list)। সেখানে aggregation tool দাও (sum/count)।

### Hands-on experiment
```ts
function truncate(text: string, maxChars = 8000, tailChars = 1500) {
  if (text.length <= maxChars) return text;
  const head = text.slice(0, maxChars - tailChars);
  const tail = text.slice(-tailChars);
  const cut = text.length - head.length - tail.length;
  return `${head}\n...[${cut} chars truncated. Call again with offset=${head.length} to read more]...\n${tail}`;
}
```
একটা বড় log file এ "শেষ error কী?" প্রশ্ন: head-only vs head+tail তুলনা করো।

### Production implementation
- প্রতিটা tool definition এ `max_output_tokens` config।
- Truncation হলে metric/log (`tool.truncated=true`), যাতে কোন tool বেশি কাটা পড়ে বোঝা যায়।
- বড় output object storage এ রেখে reference ID দাও।

### Failure cases
- Marker ছাড়া কাটায় model বলল "file এ কোনো error নেই" (error ছিল শেষে)।
- Multi-byte (বাংলা) text character index এ কাটায় ভাঙা character।

### Security concerns
Truncation এর পরও injection থাকতে পারে, এটা security control না। আবার কেউ ইচ্ছা করে বিশাল output বানিয়ে (denial-of-wallet) চাইলে truncation রক্ষা করে।

### Performance concerns
Bounded size = predictable latency।

### Cost concerns
Worst-case cost per tool call fixed হয়।

### What did I learn?
- প্রতিটা tool এর output এর "ছাদ" থাকা উচিত, আর কাটা হলে সেটা সৎভাবে জানানো উচিত।

### What can I explain to someone else?
> "Google এ search করলে ১০টা result দেখায় আর নিচে 'next page' থাকে। লাখ লাখ result একবারে দেখালে কেউ পড়তে পারত না।"

### GitHub Evidence
- [ ] Truncation helper + head-vs-tail experiment

### LinkedIn Post
- [ ] "Tool output এর ছাদ: truncation"

---

<a id="lost-in-the-middle"></a>

## 7. Lost-in-the-Middle Problem

### What is it?
গবেষণায় দেখা গেছে (Liu et al., 2023, "Lost in the Middle"), লম্বা context এ relevant তথ্য **শুরু বা শেষে** থাকলে model ভালো ব্যবহার করে, কিন্তু **মাঝে** থাকলে accuracy উল্লেখযোগ্যভাবে কমে (U-shaped curve)।

### Why does it exist?
Training data এর ধরন আর attention pattern এর কারণে model শুরু (primacy) আর সাম্প্রতিক (recency) অংশে বেশি মনোযোগ দেয়। নতুন model এ এটা কমেছে, তবে পুরো যায়নি।

### What problem does it solve?
এটা একটা **problem**। এটা জানলে context ordering আর retrieval design ঠিক করা যায়।

### How does it work?
```text
Accuracy
 ▲
 █                         █
 █ █                     █ █
 █ █ █ ▄ ▄ ▄ ▄ ▄ ▄ ▄ ▄ █ █ █
 └──────────────────────────► relevant doc এর position
 শুরু        মাঝ           শেষ
```

### Important concepts
- **Effective context < advertised context**।
- **Needle-in-a-haystack** test দিয়ে মাপা যায়, কিন্তু real task (multi-fact reasoning) এ এর চেয়ে খারাপ হয়।
- **Context rot**: context যত লম্বা, সামগ্রিক quality তত কমে।
- Mitigation: কম কিন্তু ভালো chunk, rerank করে সেরাটা শুরু/শেষে, প্রশ্ন শেষে, জরুরি instruction পুনরাবৃত্তি।

### Trade-offs
Context ছোট রাখলে এই সমস্যা কমে, কিন্তু recall কমতে পারে।

### Alternatives
RAG (শুধু relevant অংশ), multi-step (প্রথমে relevant অংশ বাছাই, তারপর উত্তর), map-reduce (প্রতিটা অংশ আলাদা process)।

### When should I use it?
(জানা) যখনই context ২০–৩০K token এর বেশি হয়, তখন ordering নিয়ে ভাবো।

### When should I NOT use it?
ছোট context এ এ নিয়ে over-engineer করবে না।

### Hands-on experiment
```ts
const filler = (n: number) => Array.from({ length: n }, (_, i) => `Paragraph ${i}: ${lorem}`).join("\n");
const needle = "The secret code is BLUE-4729.";

for (const pos of [0, 0.25, 0.5, 0.75, 1]) {
  const parts = filler(200).split("\n");
  parts.splice(Math.floor(parts.length * pos), 0, needle);
  // model কে জিজ্ঞেস করো "What is the secret code?" → position অনুযায়ী success rate
}
```
প্রতিটা position এ ১০ বার চালিয়ে success rate chart বানাও। তারপর দুইটা needle দিয়ে ("code" + "owner") multi-fact এও চেষ্টা করো।

### Production implementation
- Reranker দিয়ে সেরা chunk গুলো শুরুতে আর শেষে রাখো (মাঝে কম গুরুত্বপূর্ণ)।
- প্রশ্ন সবসময় শেষে।
- লম্বা context এ জরুরি rule শেষে আবার মনে করিয়ে দাও।

### Failure cases
- ৩০টা chunk এর ১৫ নম্বরে উত্তর ছিল, model বলল "তথ্য পাওয়া যায়নি"।
- লম্বা system prompt এর মাঝের rule model ignore করছে।

### Security concerns
উল্টো দিকও আছে: context এর শেষে থাকা injection (যেমন document এর শেষ লাইন) বেশি প্রভাব ফেলতে পারে।

### Performance concerns
বড় context ধীর। ছোট focused context দ্রুত ও ভালো।

### Cost concerns
বড় context এ বেশি খরচ করে খারাপ ফল পাওয়ার ঝুঁকি, মানে দুই দিকেই ক্ষতি।

### What did I learn?
- "Context এ আছে" মানেই "model ব্যবহার করবে" না। Position গুরুত্বপূর্ণ।

### What can I explain to someone else?
> "লম্বা বক্তৃতার শুরু আর শেষ সবার মনে থাকে, মাঝখানটা ঝাপসা। Model ও তাই।"

### GitHub Evidence
- [ ] Needle position vs accuracy chart (local model)

### LinkedIn Post
- [ ] "Lost in the middle: লম্বা context এ AI কোথায় অন্ধ"

---

<a id="context-ordering"></a>

## 8. Context Ordering

### What is it?
Context এর বিভিন্ন অংশ কোন ক্রমে সাজানো হবে তার নিয়ম। দুই লক্ষ্য: (১) **caching** এর জন্য stable অংশ আগে, (২) **accuracy** এর জন্য জরুরি অংশ শুরু বা শেষে।

### Why does it exist?
Prompt caching prefix-match এ কাজ করে, আর model এর attention position-sensitive। ভুল order এ খরচ বেশি আর উত্তর খারাপ দুটোই হয়।

### What problem does it solve?
একই token দিয়ে বেশি cache hit, আর বেশি accuracy।

### How does it work?
```text
1. Tools (sorted, fixed)              ┐
2. System prompt (frozen)             │ stable → cache breakpoint
3. Few-shot examples                  ┘
4. Memory / long-lived summary          ধীরে বদলায়
5. Conversation history (append-only)   বাড়ে, আগের অংশ cache হয়
6. Retrieved docs (সেরাটা শুরুতে/শেষে)    per request
7. User context (date, locale)          per request
8. User question                        সবসময় শেষে
```

### Important concepts
- **Stable → volatile**: যা কম বদলায় তা আগে।
- **Long document আগে, প্রশ্ন শেষে**: লম্বা context এ query শেষে রাখলে উত্তর ভালো হয়।
- **Append-only history**: মাঝখানে insert/edit = cache miss।
- **Deterministic**: একই input এ byte-for-byte একই order।

### Trade-offs
Cache-optimal order আর attention-optimal order কখনো conflict করে (যেমন জরুরি নতুন instruction শুরুতে দিতে চাইলে cache ভাঙে)। তখন mid-conversation system message বা শেষে reminder দাও।

### Alternatives
প্রতিটা request এ নতুন করে compose (caching ছাড়া), যা সহজ কিন্তু দামি।

### When should I use it?
সব multi-part prompt এ।

### When should I NOT use it?
—(সবসময় প্রযোজ্য। ছোট prompt এ শুধু প্রভাব কম।)

### Hands-on experiment
একই content দুই order এ পাঠাও: (A) question → docs, (B) docs → question। ২০টা প্রশ্নে accuracy মাপো। তারপর Anthropic API তে cache_control দিয়ে (C) system এ timestamp vs (D) user message এ timestamp: `cache_read_input_tokens` তুলনা করো।

### Production implementation
- `compose()` function এ order hardcode করো আর snapshot test লিখো।
- Cache breakpoint stable অংশের শেষে।
- Cache hit rate dashboard এ monitor করো।

### Failure cases
- Tool list প্রতি request এ ভিন্ন order এ (object iteration), ফলে cache hit ০%।
- প্রশ্ন শুরুতে আর ৩০K doc পরে, ফলে model প্রশ্নের detail ভুলে গেছে।

### Security concerns
Untrusted content সবসময় trusted instruction এর পরে আর tag এর ভেতরে। কখনো system অংশে না।

### Performance concerns
সঠিক order = বেশি cache hit = অনেক কম TTFT।

### Cost concerns
Cache read সাধারণত base input দামের ~১০% বা তারও কম। Order ঠিক করাই সবচেয়ে সস্তা cost optimization।

### What did I learn?
- Order একই সাথে cost আর quality এর lever।

### What can I explain to someone else?
> "Library তে স্থায়ী বইগুলো সামনের তাকে সাজানো থাকে (দ্রুত পাওয়া যায়), নতুন আসা পত্রিকা কাউন্টারে। আর প্রশ্নটা সবসময় librarian এর একদম সামনে রাখো।"

### GitHub Evidence
- [ ] Order A vs B accuracy + cache hit experiment

### LinkedIn Post
- [ ] "Prompt এর order বদলে খরচ কমানো ও accuracy বাড়ানো"

---

⬅️ Back to [Phase 3 — Context Engineering](README.md) · [Root README](../README.md)

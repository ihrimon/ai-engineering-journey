# Phase 5 — Agents & Agentic Systems: Multi-Agent Systems (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Multi-agent শুনতে আকর্ষণীয়, কিন্তু token খরচ কয়েকগুণ আর debug কঠিন। আগে [single-agent](agent-architecture.md#single-agent) দিয়ে চেষ্টা করো, প্রমাণ পেলে তবেই ভাগ করো।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Subagents](#subagents) | মূল agent একটা আলাদা context এর agent কে sub-task দেয়, শুধু সারাংশ ফেরত পায় |
| [Delegation](#delegation) | কোন কাজ কাকে, কী নির্দেশনা আর সীমা দিয়ে দেওয়া হবে তার design |
| [Agent communication](#agent-communication) | Agent গুলো কীভাবে তথ্য আদান-প্রদান করে: message, shared state, handoff |
| [Supervisor agents](#supervisor-agents) | একটা agent বাকিদের coordinate করে: route, monitor, merge |
| [Specialist agents](#specialist-agents) | নির্দিষ্ট domain/skill এর জন্য focused prompt + tools এর agent |

```text
Topologies
 Orchestrator–workers        Supervisor (router)          Handoff / swarm           Hierarchical
      [O]                        [S]                     [A] ⇄ [B] ⇄ [C]             [Top]
    / | \                      / | \                    (control পাস হয়)           /     \
  [w][w][w]                [Bill][Tech][Sales]                                  [Sup1]   [Sup2]
 (parallel fan-out)        (একজনকে বেছে পাঠায়)                                   / \       / \
```

---

<a id="subagents"></a>

## 1. Subagents

### What is it?
মূল (parent) agent একটা sub-task এর জন্য নতুন agent চালু করে, যার **নিজের fresh context window**, নিজের prompt আর (প্রায়ই সীমিত) tool set। Subagent কাজ শেষ করে শুধু একটা condensed result ফেরত দেয়। Parent এর context এ subagent এর পড়া হাজার হাজার token আসে না।

### Why does it exist?
Research বা codebase exploration এ প্রচুর পড়তে হয়। সব কিছু parent এর context এ ঢুকলে context ভরে যায় আর মূল কাজে focus হারায়।

### What problem does it solve?
Context isolation (parent এর context পরিষ্কার), parallelism (একাধিক subagent একসাথে), আর specialization (subagent এর আলাদা prompt/model)।

### How does it work?
```text
parent: "৫টা competitor analyze করো"
  └─ tool: run_subagent({ task: "Research Competitor A: pricing, features. Return ≤200 words JSON.", tools: ["web_search"] })
        subagent: ৩০K token পড়ে → returns 300-token summary
  └─ (parallel) run_subagent(B), run_subagent(C) ...
parent context এ শুধু ৫ × 300 token → তুলনা → final report
```

### Important concepts
- **Subagent = tool** parent এর দৃষ্টিতে।
- **Clear task spec + output format** (দেখো [delegation](#delegation))।
- **Cheaper model** subagent এ (যেমন reading-heavy কাজে ছোট model)।
- **Depth limit**: subagent আবার subagent চালাতে পারবে কিনা (সাধারণত না বা depth 1–2)।

### Trade-offs
Context পরিষ্কার আর parallel, কিন্তু subagent parent এর পুরো context জানে না (তথ্য হারায়), আর মোট token বেশি।

### Alternatives
Single agent + context pruning/compaction, deterministic parallel workflow (LLM ছাড়া fan-out)।

### When should I use it?
Reading/search-heavy, parallelizable sub-task: research, codebase exploration, অনেক document summarize।

### When should I NOT use it?
Sub-task এর জন্য parent এর পুরো context লাগে, বা কাজ ছোট।

### Hands-on experiment
```ts
async function runSubagent(task: string): Promise<string> {
  const messages: any[] = [
    { role: "system", content: "You are a focused research subagent. Return a concise answer (max 150 words)." },
    { role: "user", content: task },
  ];
  return reactLoop(messages, { tools: [searchTool], maxSteps: 6 });   // agent-patterns এর loop
}
// parent এর tool হিসেবে register করো, ৩টা subagent Promise.all দিয়ে চালাও
```
Parent এর মোট input token: subagent সহ vs সব কাজ parent নিজে করলে, তুলনা করো।

### Production implementation
Subagent run আলাদা trace span (parent span এর child), per-subagent budget, max concurrent subagents।

### Failure cases
Subagent কে vague task দেওয়ায় ("research A") সে ভুল দিক নিয়ে ৩০K token খরচ করেছে।

### Security concerns
Subagent এর tool permission parent এর চেয়ে কম রাখো। Subagent এর output (web থেকে পড়া) parent এর কাছে untrusted।

### Performance concerns
Parallel subagent দিয়ে wall-clock অনেক কমে।

### Cost concerns
মোট token বাড়ে, কিন্তু parent এর context ছোট থাকায় parent এর প্রতি step সস্তা। Net effect মাপো।

### What did I learn?
- Subagent এর আসল উপহার হলো পরিষ্কার context।

### What can I explain to someone else?
> "Manager নিজে ৫০টা report পড়ে না, ৫ জনকে ১০টা করে দেয়, প্রত্যেকে এক পাতার সারাংশ জমা দেয়। Manager এর টেবিল পরিষ্কার থাকে।"

### GitHub Evidence
- [ ] Parent + parallel subagents token comparison

### LinkedIn Post
- [ ] "Subagent: AI agent এর context পরিষ্কার রাখার কৌশল"

---

<a id="delegation"></a>

## 2. Delegation

### What is it?
একটা agent অন্য agent (বা subagent) কে কাজ দেওয়ার design: **কী কাজ**, **কতটুকু context**, **কোন tools**, **output কী format এ**, **কখন থামবে**, আর **কী করবে না**।

### Why does it exist?
Multi-agent system এর সবচেয়ে সাধারণ failure: অস্পষ্ট delegation। Worker কাজ ভুল বোঝে, duplicate করে, বা অপ্রয়োজনে বিশাল কাজ করে।

### What problem does it solve?
Worker গুলোর output usable, non-overlapping, budget এর মধ্যে।

### How does it work?
```text
ভালো delegation message:
  Objective:  "Competitor A এর ২০২৬ pricing tier গুলো বের করো"
  Context:    "আমরা SMB SaaS market এ তুলনা করছি"
  Scope:      "শুধু official website + pricing page; reviews না"
  Tools:      web_search, web_fetch
  Output:     JSON { tiers: [{name, price_usd_month, key_limits}] , sources: [url] }
  Budget:     max 5 searches
  Stop when:  pricing page পাওয়া গেছে বা ৫ search শেষ
```

### Important concepts
- **Effort scaling**: সহজ প্রশ্নে ১ worker, জটিলে বেশি। Orchestrator কে নিয়ম বলে দাও।
- **Non-overlap**: প্রতিটা worker এর boundary স্পষ্ট।
- **Structured output** merge সহজ করে।
- **Context passing**: কতটুকু parent context worker কে দেবে (বেশি = ভালো বোঝা, কিন্তু বেশি token)।

### Trade-offs
বিস্তারিত spec = ভালো output, কিন্তু orchestrator এর বেশি token আর কম flexibility।

### Alternatives
Fixed workflow (delegation code এ hardcoded)।

### When should I use it?
সব multi-agent/subagent design এ।

### When should I NOT use it?
—

### Hands-on experiment
একই research task দুইভাবে delegate করো: (A) এক লাইন "research X", (B) উপরের ৭-অংশের template। Output quality, token, duplicate তথ্য তুলনা করো।

### Production implementation
Delegation schema (zod) যাতে orchestrator এর tool call নিজেই structured হয়: `run_subagent({objective, scope, output_schema, max_steps})`।

### Failure cases
৩টা worker কে "AI trends research করো" দেওয়ায় তিনজনই একই ৫টা article পড়েছে।

### Security concerns
Delegation message এ secrets দিও না। Worker এর scope এ data access সীমিত রাখো।

### Performance concerns
ভালো scope = কম অপ্রয়োজনীয় step।

### Cost concerns
Budget field সরাসরি খরচ নিয়ন্ত্রণ করে।

### What did I learn?
- Delegation মানে ভালো job description লেখা, মানুষ বা agent দুজনের জন্যই।

### What can I explain to someone else?
> "Intern কে 'একটু research করো' বললে সে তিন দিন হারিয়ে যায়। 'এই ৩টা প্রশ্নের উত্তর, এই source থেকে, কাল দুপুরের মধ্যে, এক পাতায়' বললে কাজ হয়।"

### GitHub Evidence
- [ ] Vague vs structured delegation comparison

### LinkedIn Post
- [ ] —

---

<a id="agent-communication"></a>

## 3. Agent Communication

### What is it?
Agent গুলোর মধ্যে তথ্য আদান-প্রদানের পদ্ধতি:
- **Message passing / return value** (worker → orchestrator result)
- **Shared state / blackboard** (সবাই একটা state object বা file পড়ে/লেখে)
- **Handoff** (control আর conversation একজন থেকে আরেকজনের কাছে যায়)
- **Protocol-based** (যেমন A2A, agent-to-agent protocol, আলাদা system এর agent এর মধ্যে)

### Why does it exist?
Agent রা একে অপরের context দেখে না। তথ্য স্পষ্টভাবে পাঠাতে না হলে কাজ জোড়া লাগে না।

### What problem does it solve?
Coordination: কে কী জানে, কে কী করছে, ফলাফল কোথায়।

### How does it work?
```text
Return value:  orchestrator ──task──► worker ──result──► orchestrator     (সবচেয়ে সহজ, tree)
Shared state:  [A] ──write──► { state / files / DB } ◄──read── [B]         (graph framework)
Handoff:       triage agent ──transfer_to_billing(context)──► billing agent (conversation চলতে থাকে)
Artifacts:     worker বড় output file/storage এ লেখে, শুধু reference পাঠায়     (telephone game এড়ায়)
```

### Important concepts
- **Telephone game problem**: প্রতিটা hop এ summary করলে তথ্য বিকৃত হয়। বড় output artifact হিসেবে store করে reference দাও।
- **Structured messages** (schema) free text এর চেয়ে নির্ভরযোগ্য।
- **Handoff context**: নতুন agent কে কতটুকু history দেবে।
- **Protocols**: MCP (agent ↔ tool), A2A (agent ↔ agent)।

### Trade-offs
Shared state = সহজ access কিন্তু race condition ও coupling। Message passing = পরিষ্কার boundary কিন্তু তথ্য হারায়।

### Alternatives
Single agent (communication লাগে না)।

### When should I use it?
যেকোনো multi-agent system এ, design এর শুরুতেই ঠিক করো।

### When should I NOT use it?
—

### Hands-on experiment
৩টা agent এর একটা chain (researcher → analyst → writer)। (A) প্রতিটা আগেরটার summary পায়, (B) researcher raw notes file এ লেখে, বাকিরা file পড়ে। Final report এ fact accuracy তুলনা করো।

### Production implementation
Message schema version, artifact store (S3/DB) + reference ID, inter-agent message trace।

### Failure cases
Writer agent analyst এর summary এর summary থেকে লিখেছে, একটা সংখ্যা পথে বদলে গেছে।

### Security concerns
Agent-to-agent message ও untrusted (এক agent injected হলে বাকিদের সংক্রমিত করতে পারে)। Privilege boundary বজায় রাখো।

### Performance concerns
Shared DB access এ contention। Handoff এ context resend।

### Cost concerns
প্রতি message = token। Artifact reference সস্তা।

### What did I learn?
- Multi-agent এ বেশিরভাগ bug আসলে communication bug।

### What can I explain to someone else?
> "অফিসে মুখে মুখে খবর ছড়ালে বিকৃত হয়। তাই গুরুত্বপূর্ণ জিনিস shared drive এ রেখে সবাইকে link পাঠাও।"

### GitHub Evidence
- [ ] Summary-chain vs shared-artifact accuracy test

### LinkedIn Post
- [ ] —

---

<a id="supervisor-agents"></a>

## 4. Supervisor Agents

### What is it?
একটা coordinator agent যে বাকি agent দের উপরে থাকে: request বুঝে **কোন specialist কে পাঠাবে** (route), কাজ **monitor** করে, ফলাফল **যাচাই ও merge** করে, আর দরকার হলে আবার পাঠায়। Orchestrator এর কাছাকাছি ধারণা।

### Why does it exist?
অনেক specialist থাকলে কাউকে ঠিক করতে হয় কোন কাজ কার, আর সবার কাজ মিলিয়ে একটা উত্তর দিতে হয়।

### What problem does it solve?
Routing, quality control, আর একটা unified response।

### How does it work?
```text
user ─► Supervisor (tools: transfer_to_billing, transfer_to_tech, finish)
          │ "billing প্রশ্ন" → billing agent → result → supervisor
          │ "আরও tech তথ্য লাগবে" → tech agent → result → supervisor
          └► check: প্রশ্নের সব অংশের উত্তর হয়েছে? → finish(final answer)
```

### Important concepts
- **Routing** (classification): LLM বা ছোট classifier, deterministic হলে আরও ভালো।
- **Verification**: supervisor worker এর output check করে (evaluator-optimizer pattern)।
- **Bottleneck**: সব কিছু supervisor দিয়ে যায়, তাই তার context ভরে যেতে পারে।
- **Hierarchical**: বড় system এ supervisor এর উপরে supervisor।

### Trade-offs
পরিষ্কার control ও quality gate, কিন্তু extra LLM hop (latency, cost) আর supervisor single point of failure।

### Alternatives
Simple router (একবার route করে ছেড়ে দেয়), handoff/swarm (supervisor ছাড়া peer-to-peer)।

### When should I use it?
অনেক domain specialist, multi-part request, quality check দরকার।

### When should I NOT use it?
শুধু একবার routing দরকার হলে। Simple router সস্তা।

### Hands-on experiment
৩টা specialist (billing, tech, sales) + supervisor। ১৫টা request (কিছু multi-part: "bill কেন বেশি আর app crash করছে কেন")। Routing accuracy আর multi-part completeness মাপো।

### Production implementation
Routing decision log, max delegation rounds, supervisor এর context এ শুধু worker summary।

### Failure cases
Supervisor নিজেই উত্তর দিতে শুরু করেছে specialist কে না পাঠিয়ে (role drift)। Prompt এ স্পষ্ট করো, বা supervisor থেকে domain tools সরিয়ে দাও।

### Security concerns
Supervisor এর routing manipulate করে (injection) user কে বেশি privileged specialist এর কাছে পাঠানো। Specialist নিজেও authorization check করবে।

### Performance concerns
প্রতি delegation এ ২টা extra LLM call (supervisor in/out)।

### Cost concerns
Supervisor এ ছোট/মাঝারি model প্রায়ই যথেষ্ট (routing সহজ কাজ)।

### What did I learn?
- Supervisor = traffic police + quality inspector।

### What can I explain to someone else?
> "Hospital এর reception: রোগী দেখে ঠিক করে কোন বিভাগে পাঠাবে, আর শেষে সব report একত্র করে দেয়।"

### GitHub Evidence
- [ ] Supervisor + 3 specialists routing eval

### LinkedIn Post
- [ ] —

---

<a id="specialist-agents"></a>

## 5. Specialist Agents

### What is it?
নির্দিষ্ট domain বা skill এর জন্য তৈরি agent: **focused system prompt**, **শুধু দরকারি tools**, কখনো **আলাদা model** (কোডের জন্য একটা, সস্তা classification এর জন্য আরেকটা), আর domain-specific knowledge/RAG।

### Why does it exist?
এক agent এর prompt এ সব domain এর rule আর ৫০টা tool দিলে সে কোনোটাতেই ভালো না। Focus = accuracy।

### What problem does it solve?
উচ্চ accuracy প্রতিটা domain এ, ছোট prompt/tool set, আর আলাদাভাবে test/improve করা যায়।

### How does it work?
```text
billing_agent:   prompt: billing policy · tools: get_invoice, issue_refund(≤$100) · model: mid
tech_agent:      prompt: troubleshooting guide · tools: search_kb, get_logs · model: strong
sql_agent:       prompt: schema + rules · tools: run_readonly_sql · model: strong, low temp
```

### Important concepts
- **Least privilege per specialist**: billing agent এর কাছে log tool নেই।
- **Independent eval**: প্রতিটা specialist এর নিজের test set।
- **Model per role**: খরচ-quality optimize।
- **Reusable**: একই specialist অনেক workflow এ।

### Trade-offs
প্রতিটা ভালো, কিন্তু সংখ্যা বাড়লে routing ও maintenance জটিল, আর domain এর মাঝের প্রশ্নে কেউই মালিক না।

### Alternatives
এক general agent + skills (progressive disclosure), যা অনেক ক্ষেত্রে specialist এর সুবিধা দেয় কম জটিলতায়।

### When should I use it?
Domain গুলো স্পষ্টভাবে আলাদা, প্রতিটায় আলাদা tool/permission/policy।

### When should I NOT use it?
Domain overlap বেশি, বা সব কাজ একই tool set দিয়ে হয়।

### Hands-on experiment
একই ২০টা প্রশ্ন: (A) এক general agent (সব tool, সব rule), (B) ২টা specialist + router। Accuracy, tool selection error, token তুলনা।

### Production implementation
Specialist registry (name, description, prompt version, tools, model), প্রতিটার eval suite CI তে।

### Failure cases
"আমার refund কেন আটকে আছে, app এ error দেখাচ্ছে" প্রশ্ন billing আর tech দুইজনের মাঝে পিং-পং।

### Security concerns
Least privilege দিয়ে blast radius কমে: একটা specialist injected হলেও শুধু তার tools এর ক্ষতি।

### Performance concerns
ছোট prompt/tool set = কম TTFT।

### Cost concerns
Role অনুযায়ী model বাছাই দিয়ে খরচ কমানো যায়।

### What did I learn?
- Specialist = focused prompt + least privilege + নিজস্ব eval।

### What can I explain to someone else?
> "General physician সব কিছু একটু জানে, কিন্তু হৃদরোগের জন্য cardiologist। আবার প্রতিটা সর্দির জন্য specialist লাগে না।"

### GitHub Evidence
- [ ] General vs specialist agents accuracy table

### LinkedIn Post
- [ ] "Multi-agent system: কখন লাগে, কখন লাগে না"

---

⬅️ Back to [Phase 5 — Agents & Agentic Systems](README.md) · [Root README](../README.md)

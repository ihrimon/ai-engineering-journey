# Phase 5 — Agents & Agentic Systems: Agent Patterns (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 🔗 Phase 2 তে [ReAct prompting](../02-prompt-engineering/techniques.md#react-prompting) আর [reflection](../02-prompt-engineering/techniques.md#reflection) কে **prompt technique** হিসেবে দেখেছি। এখানে এগুলোকে দেখছি **system/control-flow pattern** হিসেবে: loop, state, tool, stopping condition সহ।
>
> ⚠️ Code example TypeScript এ। Local practice: Ollama (`qwen2.5:3b` tool calling পারে, তবে ছোট model, তাই ভুল হবে, সেটাও শেখার অংশ)।

## Quick Cheat Sheet

| Pattern | এক লাইনে | LLM calls |
|---|---|---|
| [ReAct](#react) | Think → Act (tool) → Observe → আবার, যতক্ষণ না কাজ শেষ | প্রতি step এ ১টা |
| [Plan-and-Execute](#plan-and-execute) | আগে পুরো plan, তারপর step ধরে execute, দরকারে re-plan | plan + প্রতি step |
| [Reflexion](#reflexion) | চেষ্টা → মূল্যায়ন → ভুল থেকে "lesson" লিখে → আবার চেষ্টা | প্রতি trial এ একাধিক |
| [ReWOO](#rewoo) | এক call এ পুরো plan + tool call placeholder, tool চালাও, শেষে এক call এ উত্তর | মোট ২টা (মোটামুটি) |

```text
Workflow (code নিয়ন্ত্রণ করে)  ◄──────────────────────────►  Agent (model নিয়ন্ত্রণ করে)
fixed pipeline   ReWOO   Plan-and-Execute   ReAct   Reflexion/autonomous loop
বেশি predictable, সস্তা                         বেশি flexible, দামি, কম predictable

নিয়ম: সবচেয়ে সহজ যে pattern কাজ করে, সেটাই বাছো।
```

---

<a id="react"></a>

## 1. ReAct

### What is it?
**Reasoning + Acting** (Yao et al., 2022)। Model একটা loop এ চলে: চিন্তা করে (কী করা দরকার), একটা action নেয় (tool call), ফলাফল (observation) দেখে, আবার চিন্তা করে, যতক্ষণ না final answer দিতে পারে। আধুনিক tool-calling API তে এটাই default agent loop।

### Why does it exist?
শুধু reasoning (CoT) এ model এর কাছে fresh তথ্য নেই। শুধু acting এ পরিকল্পনা নেই। দুটো interleave করলে model ধাপে ধাপে তথ্য জোগাড় করে সিদ্ধান্ত নিতে পারে।

### What problem does it solve?
যেসব task এর step আগে থেকে জানা নেই: research, debugging, customer support, data lookup।

### How does it work?
```text
while (true):
  response = LLM(system, tools, messages)
  if stop_reason != "tool_use": return response          // final answer
  for each tool_use: result = run(tool, input)           // act
  messages += [assistant(response.content), user(tool_results)]   // observe
  if steps > MAX: stop with partial answer
```

### Important concepts
- **Stopping condition**: model নিজে থামে (no tool call) + তোমার hard limit (max steps, budget, timeout)।
- **Tool result shape** (দেখো [tool results](../03-context/context-architecture.md#tool-results))।
- **Parallel tool calls**: এক step এ একাধিক tool, সব result এক message এ ফেরত।
- **Error as observation**: tool error `is_error: true` দিয়ে ফেরত দাও, model নিজে সামলাবে।

### Trade-offs
Flexible ও সহজ, কিন্তু প্রতি step এ একটা LLM call (ধীর, দামি), আর লম্বা task এ পথ হারাতে পারে (no global plan)।

### Alternatives
Plan-and-Execute (লম্বা task), ReWOO (কম call), fixed workflow (step জানা থাকলে)।

### When should I use it?
Default agent pattern হিসেবে: ২–১৫ step এর task, step আগে থেকে অজানা।

### When should I NOT use it?
Step গুলো আগে থেকে জানা থাকলে (fixed pipeline সস্তা ও নির্ভরযোগ্য), বা খুব লম্বা (৫০+ step) task এ plan ছাড়া।

### Hands-on experiment
```ts
import ollama from "ollama";

const tools = [{
  type: "function",
  function: {
    name: "get_weather",
    description: "Get current temperature for a city",
    parameters: { type: "object", properties: { city: { type: "string" } }, required: ["city"] },
  },
}];
const impl: Record<string, (a: any) => string> = {
  get_weather: ({ city }) => JSON.stringify({ city, tempC: city === "Dhaka" ? 31 : 18 }),
};

const messages: any[] = [{ role: "user", content: "ঢাকা আর লন্ডনের তাপমাত্রার পার্থক্য কত?" }];
for (let step = 0; step < 5; step++) {
  const res = await ollama.chat({ model: "qwen2.5:3b", messages, tools });
  messages.push(res.message);
  if (!res.message.tool_calls?.length) { console.log(res.message.content); break; }
  for (const call of res.message.tool_calls) {
    messages.push({ role: "tool", content: impl[call.function.name](call.function.arguments) });
  }
}
```
Step সংখ্যা, প্রতিটা tool call, আর final answer log করো।

### Production implementation
- Max steps (১০–২৫), total token budget, per-tool timeout।
- প্রতিটা step trace (span): input, tool, output, tokens।
- Claude এ SDK এর tool runner (`client.beta.messages.toolRunner`) loop নিজে চালায়, তবে hook দিয়ে approval/logging রাখা যায়।

### Failure cases
- একই tool একই input এ বারবার (loop)। Duplicate call detection লাগবে।
- Tool error চুপচাপ গিলে ফেলায় model ভাবল সফল।

### Security concerns
Tool result এ indirect injection, model এর হাতে যত শক্তিশালী tool, ঝুঁকি তত বেশি। Least privilege + sensitive action এ approval।

### Performance concerns
Latency = steps × (LLM + tool)। Parallel tool call আর ছোট output দিয়ে কমাও।

### Cost concerns
প্রতি step এ পুরো history resend, তাই step বাড়লে খরচ quadratic। Caching আর pruning।

### What did I learn?
- Agent = LLM + tools + loop + stopping condition। ReAct হলো সেই loop এর সবচেয়ে সরল রূপ।

### What can I explain to someone else?
> "রান্নার রেসিপি ছাড়া রান্না: একটু চেখে দেখো (observe), লবণ কম, লবণ দাও (act), আবার চাখো। যতক্ষণ না স্বাদ ঠিক হয়।"

### GitHub Evidence
- [ ] Minimal ReAct loop (Ollama) with step trace

### LinkedIn Post
- [ ] "AI agent আসলে কী? একটা while loop"

---

<a id="plan-and-execute"></a>

## 2. Plan-and-Execute

### What is it?
দুই ভাগ: **Planner** (সাধারণত শক্তিশালী model) আগে পুরো task এর step-by-step plan বানায়, তারপর **Executor** (ছোট model বা ReAct sub-agent) প্রতিটা step চালায়। Step শেষে দরকার হলে **re-plan**।

### Why does it exist?
ReAct প্রতি step এ শুধু পরের কাজটা ভাবে, তাই লম্বা task এ দিক হারায়। একটা global plan focus রাখে, আর বড় model শুধু planning এ লাগে বলে খরচও কমে।

### What problem does it solve?
লম্বা, multi-step task (report লেখা, migration, research) এ coherence আর progress tracking।

### How does it work?
```text
plan = Planner(task) → ["1. data খোঁজো", "2. তুলনা করো", "3. report লেখো"]
for step in plan:
    result = Executor(step, context_so_far)
    state.done.push({step, result})
    if needs_change: plan = Replanner(task, state)
final = Synthesizer(task, state)
```

### Important concepts
- **Plan as structured output** (JSON list), যাতে code track করতে পারে।
- **Replanning trigger**: step fail, নতুন তথ্য, বা প্রতি step পরে।
- **Planner/executor model split**: বড় model plan, ছোট model execute।
- **Plan visibility**: user কে plan দেখানো (এমনকি approve করানো) ভালো UX।

### Trade-offs
ভালো structure ও কম বড়-model call, কিন্তু প্রথম plan ভুল হলে পুরো execution ভুল দিকে, আর rigid plan অপ্রত্যাশিত পরিস্থিতিতে মানায় না।

### Alternatives
ReAct (ছোট task), ReWOO (আরও rigid, কম call), workflow graph (plan code এ fix)।

### When should I use it?
লম্বা task যার step মোটামুটি আগে ভাবা যায়: research report, multi-file code change, data pipeline।

### When should I NOT use it?
খুব exploratory task যেখানে প্রতিটা observation পরের সব কিছু বদলে দেয়।

### Hands-on experiment
একই task ("তিনটা শহরের আবহাওয়া তুলনা করে সবচেয়ে আরামদায়কটা সুপারিশ করো") ReAct আর Plan-and-Execute দুইভাবে চালাও। Step সংখ্যা, token, আর output quality তুলনা করো।

### Production implementation
```ts
const PlanSchema = z.object({ steps: z.array(z.object({ id: z.number(), goal: z.string(), tool_hint: z.string().optional() })) });
// state: { plan, done: [], current }  → DB তে checkpoint (crash হলে resume)
```

### Failure cases
Planner এমন step লিখেছে যার জন্য কোনো tool নেই ("CEO কে ফোন করো")। Planner কে available tool list দাও।

### Security concerns
Plan এ sensitive action থাকলে execution এর আগে approval (plan-level HITL)।

### Performance concerns
Independent step গুলো parallel execute করা যায় (DAG plan)।

### Cost concerns
Planning এ বড় model একবার, execution এ ছোট model, প্রায়ই ReAct-with-big-model এর চেয়ে সস্তা।

### What did I learn?
- আগে ভাবো, তারপর করো, আর দরকার হলে পরিকল্পনা বদলাও।

### What can I explain to someone else?
> "বাড়ি বানানোর আগে নকশা (plan), তারপর মিস্ত্রিরা ধাপে ধাপে কাজ (execute)। মাটি খুঁড়ে পাথর পেলে নকশা একটু বদলাও (re-plan)।"

### GitHub Evidence
- [ ] ReAct vs Plan-and-Execute comparison

### LinkedIn Post
- [ ] "Agent কে আগে plan করতে দাও"

---

<a id="reflexion"></a>

## 3. Reflexion

### What is it?
(Shinn et al., 2023) Agent একটা task চেষ্টা করে, একটা **evaluator** (test, rule, বা LLM judge) ফলাফল মূল্যায়ন করে, ব্যর্থ হলে agent নিজের ভুল নিয়ে একটা **verbal reflection** ("lesson") লেখে, যা memory তে রেখে পরের চেষ্টায় context এ দেওয়া হয়। Weight update ছাড়াই "শেখা"।

### Why does it exist?
Model প্রথম চেষ্টায় ভুল করে, কিন্তু ভুলের কারণ বুঝিয়ে দিলে পরের বার ভালো করে। Feedback কে স্থায়ী শিক্ষা বানানো।

### What problem does it solve?
Verifiable task (code with tests, puzzle, structured extraction) এ success rate বাড়ায়, iteration এর মাধ্যমে।

### How does it work?
```text
for trial in 1..N:
    output = Actor(task, reflections)
    ok, feedback = Evaluator(output)        // unit tests, schema check, judge
    if ok: return output
    reflection = SelfReflect(task, output, feedback)  // "আমি edge case empty array ভুলে গিয়েছিলাম"
    reflections.push(reflection)            // episodic memory
```

### Important concepts
- **External signal সবচেয়ে গুরুত্বপূর্ণ**: test pass/fail এর মতো objective feedback থাকলে Reflexion সবচেয়ে কার্যকর। শুধু নিজের মতামত দিয়ে self-critique দুর্বল।
- **Episodic memory** (দেখো [memory.md](../03-context/memory.md#episodic-memory))।
- **Max trials** (৩–৫)।
- Self-correction (Phase 2) এর agent-level রূপ।

### Trade-offs
Accuracy ↑ (verifiable task এ), কিন্তু N গুণ খরচ ও সময়, আর evaluator দুর্বল হলে ভুল দিকে "শেখে"।

### Alternatives
Best-of-N sampling (N বার চালিয়ে evaluator দিয়ে সেরাটা), human feedback।

### When should I use it?
Code generation (test আছে), SQL (execute করে check), structured output (schema validation)।

### When should I NOT use it?
কোনো নির্ভরযোগ্য evaluator নেই, বা latency কঠোর।

### Hands-on experiment
একটা function লেখার task দাও (৫টা unit test সহ)। (১) single attempt, (২) Reflexion max 3 trials (failing test এর output feedback হিসেবে)। ১০টা problem এ pass rate তুলনা করো।

### Production implementation
- Evaluator deterministic (test/schema) যতটা সম্ভব।
- Reflection গুলো log করো, এগুলো দারুণ debug data।
- Cross-task lesson রাখতে চাইলে procedural memory তে promote (human review সহ)।

### Failure cases
Evaluator নিজেই ভুল (test এ bug), তাই agent সঠিক code কে "ভুল" ভেবে নষ্ট করছে।

### Security concerns
Code execution evaluator = sandbox বাধ্যতামূলক।

### Performance concerns
Trial সংখ্যা × latency।

### Cost concerns
Worst case N গুণ। Early exit (প্রথমবার pass হলে থামো)।

### What did I learn?
- "ভুল থেকে শেখা" কাজ করে যখন বলা যায় ভুলটা আসলে কোথায়।

### What can I explain to someone else?
> "পরীক্ষায় ফেল করে খাতা দেখে লিখে রাখা 'ভগ্নাংশে ভুল করি', পরের পরীক্ষার আগে সেই নোটটা পড়ে নেওয়া।"

### GitHub Evidence
- [ ] Reflexion on coding tasks (pass rate before/after)

### LinkedIn Post
- [ ] "AI agent কি নিজের ভুল থেকে শিখতে পারে? Reflexion"

---

<a id="rewoo"></a>

## 4. ReWOO

### What is it?
**Reasoning WithOut Observation** (Xu et al., 2023)। **Planner** এক call এ পুরো plan লেখে, tool call সহ, যেখানে পরের step এর input আগের step এর result এর **placeholder** (`#E1`, `#E2`)। **Worker** tool গুলো চালায় (placeholder বসিয়ে), তারপর **Solver** এক call এ সব evidence দেখে final answer দেয়।

### Why does it exist?
ReAct এ প্রতিটা tool result এর পরে LLM কে আবার পুরো history পড়তে হয়, ফলে অনেক token। ReWOO observation ছাড়াই আগে plan করে LLM call আর token কমায়।

### What problem does it solve?
Predictable, tool-heavy task এ token খরচ আর latency কমানো।

### How does it work?
```text
Planner (1 call):
  Plan: ঢাকার জনসংখ্যা খোঁজো   #E1 = search["Dhaka population 2026"]
  Plan: লন্ডনের জনসংখ্যা খোঁজো  #E2 = search["London population 2026"]
  Plan: অনুপাত বের করো          #E3 = calculator["#E1 / #E2"]
Worker: tools চালায়, #E গুলো substitute (independent গুলো parallel)
Solver (1 call): task + সব evidence → answer
```

### Important concepts
- **Placeholder dependency** = একটা DAG।
- **No observation during planning**: plan tool result দেখে বদলায় না।
- **Parallelism**: independent step একসাথে।

### Trade-offs
খুব efficient (কম call, কম token), কিন্তু rigid: অপ্রত্যাশিত tool result এ মানিয়ে নিতে পারে না।

### Alternatives
Plan-and-Execute (re-plan সহ), ReAct, LLMCompiler (parallel DAG execution এর আরেকটা রূপ)।

### When should I use it?
Step গুলো predictable, tool নির্ভরযোগ্য: data gathering, multi-lookup প্রশ্ন।

### When should I NOT use it?
Exploratory task যেখানে প্রথম result দেখে পরের কাজ ঠিক হয়।

### Hands-on experiment
একই ৫টা multi-lookup প্রশ্ন ReAct আর ReWOO দিয়ে চালাও। মোট LLM call, total tokens, latency, আর accuracy table।

### Production implementation
Planner output JSON (steps with `id, tool, args, depends_on`), topological execution, solver এ সব evidence।

### Failure cases
`#E1` search empty এসেছে, কিন্তু plan জানে না, তাই calculator "undefined / 9000000" চালিয়েছে। Worker এ validation + fallback (ReAct এ switch)।

### Security concerns
Plan আগে লেখা, তাই execution এর আগে পুরো plan audit/approve করা সহজ। এটা একটা security সুবিধা।

### Performance concerns
Parallel worker দিয়ে দ্রুত।

### Cost concerns
ReAct এর তুলনায় অনেক কম token (history বারবার resend হয় না)।

### What did I learn?
- আগে পুরো "recipe" লিখে ফেললে রান্নার মাঝে বারবার রেসিপি বই খুলতে হয় না, কিন্তু উপকরণ না থাকলে বিপদ।

### What can I explain to someone else?
> "বাজারে যাওয়ার আগে পুরো লিস্ট লিখে ফেলা, তারপর একবারে সব কেনা, তারপর বাসায় এসে রান্না। প্রতিটা জিনিস কিনে বাসায় ফিরে পরের জিনিস ভাবতে হয় না।"

### GitHub Evidence
- [ ] ReAct vs ReWOO token/latency table

### LinkedIn Post
- [ ] "৪টা agent pattern: ReAct, Plan-and-Execute, Reflexion, ReWOO"

---

⬅️ Back to [Phase 5 — Agents & Agentic Systems](README.md) · [Root README](../README.md)

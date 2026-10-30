# Phase 5 — Agents & Agentic Systems: Agent Architecture (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 💡 Anthropic এর "Building effective agents" (2024) লেখার মূল কথা মনে রাখো: **workflow** (code-defined path) আর **agent** (model-directed path) আলাদা, আর সবচেয়ে সহজ সমাধান দিয়ে শুরু করো।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Single-agent](#single-agent) | একটা model + tools + loop: বেশিরভাগ use case এর সঠিক শুরু |
| [Multi-agent](#multi-agent) | একাধিক agent আলাদা role/context এ, একসাথে কাজ ভাগ করে |
| [State machines](#state-machines) | Agent নির্দিষ্ট state এ থাকে, নির্দিষ্ট transition এ পরের state এ যায় |
| [Workflow graphs](#workflow-graphs) | Node (step) + edge (condition) দিয়ে flow: branch, loop, parallel |
| [Agent state](#agent-state) | Agent run এর সব চলমান data: messages, plan, results, status |
| [Agent memory](#agent-memory) | Run এর বাইরে টিকে থাকা জ্ঞান। Architecture এ এর জায়গা |

```text
                 Agentic system design spectrum
Single LLM call → Prompt chain → Router → Parallel → Orchestrator-workers → Autonomous agent
   (সহজ, সস্তা, predictable)                               (flexible, দামি, unpredictable)

Building blocks:  Model · Tools · Loop · State · Memory · Guardrails · Human-in-the-loop
```

---

<a id="single-agent"></a>

## 1. Single-Agent

### What is it?
একটা LLM, একটা system prompt, একটা tool set, একটা loop (ReAct)। পুরো task সেই এক agent একই context এ করে।

### Why does it exist?
সবচেয়ে সহজ agent architecture: debug করা সহজ, পুরো context এক জায়গায়, কোনো inter-agent communication নেই।

### What problem does it solve?
Tool-using, multi-step task, যেগুলো এক context window এ ধরে।

### How does it work?
```text
┌──────────── Agent ────────────┐
│ system prompt (role, rules)    │
│ tools: [search, db, email]     │──► loop: LLM ⇄ tools ⇄ state
│ state: messages + scratchpad   │
└────────────────────────────────┘
```

### Important concepts
- **Tool count**: ১০–২০ এর বেশি tool হলে selection accuracy কমে। Tool search/deferred loading।
- **Context growth**: লম্বা task এ pruning/compaction।
- **One system prompt** সব behavior নিয়ন্ত্রণ করে।

### Trade-offs
সহজ ও coherent, কিন্তু context ভরে যায়, অনেক tool এ বিভ্রান্ত হয়, আর parallel কাজ সীমিত।

### Alternatives
Workflow (fixed steps), multi-agent।

### When should I use it?
**Default**। প্রমাণ না পাওয়া পর্যন্ত single-agent থেকে শুরু করো।

### When should I NOT use it?
Task স্বাভাবিকভাবে parallel ও independent (১০টা আলাদা source research), বা context একটা agent এ ধরে না।

### Hands-on experiment
তোমার ReAct loop এ tool সংখ্যা ৩ → ১০ → ২৫ করো (dummy tool সহ)। সঠিক tool selection rate মাপো।

### Production implementation
Agent config (prompt, tools, model, limits) version করা file এ। Trace সব step।

### Failure cases
২৫টা tool, model `search_orders` এর বদলে `search_products` বাছছে বারবার।

### Security concerns
সব tool এক agent এর হাতে, তাই injection সফল হলে সবচেয়ে বেশি ক্ষতি। Tool permission সীমিত রাখো।

### Performance concerns
Sequential loop। Parallel tool call দিয়ে কিছুটা কমানো যায়।

### Cost concerns
সবচেয়ে কম overhead (এক context, এক cache)।

### What did I learn?
- Single-agent দিয়ে শুরু করো। জটিলতা যোগ করো শুধু প্রমাণ পেলে।

### What can I explain to someone else?
> "একজন দক্ষ সহকারী যার হাতে কয়েকটা যন্ত্র আছে। বেশিরভাগ কাজের জন্য একজনই যথেষ্ট, পুরো team নিয়োগ লাগে না।"

### GitHub Evidence
- [ ] Tool count vs selection accuracy

### LinkedIn Post
- [ ] "Multi-agent এর আগে single-agent কেন"

---

<a id="multi-agent"></a>

## 2. Multi-Agent

### What is it?
একাধিক agent, প্রত্যেকে নিজের role, system prompt, tool set আর **নিজের context window** নিয়ে, একটা বড় task ভাগ করে করে। সাধারণ রূপ: orchestrator + worker, supervisor + specialist, peer-to-peer। বিস্তারিত: [multi-agent-systems.md](multi-agent-systems.md)।

### Why does it exist?
একটা context window এ সব কিছু ধরে না, আর একটা agent সব ধরনের কাজে সমান ভালো না। ভাগ করলে প্রতিটা agent focused আর parallel চলতে পারে।

### What problem does it solve?
Breadth-heavy task (অনেক source research), specialization (coder + reviewer), context isolation (worker এর noise orchestrator এর context এ যায় না)।

### How does it work?
```text
             ┌──── Orchestrator ────┐
             │ plan · delegate · merge │
             └──┬────────┬────────┬──┘
        worker A     worker B     worker C     (প্রত্যেকে আলাদা context, parallel)
        (search)     (analyze)    (write)
             └────── condensed results ──────► orchestrator → final
```

### Important concepts
- **Context isolation** = মূল সুবিধা: worker ৫০K token পড়ে ১K summary ফেরত দেয়।
- **Coordination overhead**: communication, duplicate work, conflict।
- **Token cost multiply** হয় (Anthropic এর research system এ multi-agent single chat এর কয়েকগুণ token খরচ করেছে বলে তারা লিখেছে)।
- **Shared state** বনাম **message passing**।

### Trade-offs
Parallel, specialized, বড় task সম্ভব, কিন্তু অনেক বেশি token, debug কঠিন, আর agent গুলোর মাঝে ভুল বোঝাবুঝি।

### Alternatives
Single-agent + subagent tool (হালকা multi-agent), workflow with parallel steps।

### When should I use it?
Parallelizable, breadth-first task, যেখানে value বেশি (deep research, বড় codebase analysis)।

### When should I NOT use it?
Tightly coupled task যেখানে সবার সব context লাগে (যেমন বেশিরভাগ coding task), বা cost-sensitive।

### Hands-on experiment
"৫টা company সম্পর্কে ছোট profile বানাও" task: (১) single-agent sequential, (২) orchestrator + ৫টা parallel worker। Time, tokens, quality তুলনা।

### Production implementation
Worker result schema (structured), max workers, per-worker budget, পুরো tree trace।

### Failure cases
দুই worker একই কাজ করছে (vague delegation), বা orchestrator worker এর ভুল ফলাফল যাচাই না করে চূড়ান্ত।

### Security concerns
Agent গুলোর মাঝে message = নতুন injection পথ (worker এর পড়া web page থেকে injection orchestrator এ যায়)।

### Performance concerns
Parallelism দিয়ে wall-clock কমে।

### Cost concerns
Token কয়েকগুণ। Worker এ ছোট model দিয়ে কমাও।

### What did I learn?
- Multi-agent এর আসল লাভ context isolation আর parallelism, দাম token।

### What can I explain to someone else?
> "বড় প্রজেক্টে team lead কাজ ভাগ করে দেয়, প্রত্যেকে নিজের অংশ করে শুধু সারাংশ জমা দেয়। দ্রুত হয়, কিন্তু meeting আর সমন্বয়ের খরচ আছে।"

### GitHub Evidence
- [ ] Single vs orchestrator-worker comparison

### LinkedIn Post
- [ ] —

---

<a id="state-machines"></a>

## 3. State Machines

### What is it?
Agent/workflow কে **finite state machine** হিসেবে design করা: নির্দিষ্ট কয়েকটা state (যেমন `collecting_info → verifying → awaiting_approval → executing → done`), আর কোন state থেকে কোন event এ কোন state এ যাওয়া যাবে তার নিয়ম। LLM state এর ভেতরে কাজ করে বা transition ঠিক করে, কিন্তু allowed transition code এ fix।

### Why does it exist?
Free-form agent loop unpredictable। Business process (refund, onboarding, KYC) এর নির্দিষ্ট ধাপ আর নিয়ম আছে, যা ভাঙা চলবে না।

### What problem does it solve?
Predictability, auditability, আর "agent কখনো approval ছাড়া execute এ যাবে না" ধরনের guarantee।

### How does it work?
```text
        ┌──────────────┐ info complete ┌───────────┐ valid ┌───────────────────┐
start ─►│ collect_info │──────────────►│ verify    │──────►│ awaiting_approval │
        └──────▲───────┘               └─────┬─────┘       └─────────┬─────────┘
               └─────── missing info ────────┘          approved ▼   rejected ▼
                                                        ┌─────────┐   ┌────────┐
                                                        │ execute │──►│  done  │
                                                        └─────────┘   └────────┘
```

### Important concepts
- **Explicit states + transitions** (XState, বা নিজের enum + switch)।
- **Guards**: transition এর condition code এ।
- **Per-state prompt/tools**: প্রতিটা state এ শুধু দরকারি tool।
- **Persisted state**: crash/resume, audit log।

### Trade-offs
খুব predictable ও safe, কিন্তু কম flexible, নতুন পরিস্থিতিতে state যোগ করতে হয়।

### Alternatives
Free-form ReAct, workflow graph (state machine এর generalization)।

### When should I use it?
Regulated বা business-critical process: refund, booking, order, onboarding, support escalation।

### When should I NOT use it?
Open-ended research/creative task।

### Hands-on experiment
একটা refund agent কে state machine দিয়ে লিখো (৫টা state, প্রতিটায় আলাদা tool set)। চেষ্টা করো prompt injection দিয়ে `collect_info` থেকে সরাসরি `execute` এ যেতে। Code-level transition guard কি আটকায়?

### Production implementation
```ts
type State = "collect_info" | "verify" | "awaiting_approval" | "execute" | "done";
const transitions: Record<State, State[]> = {
  collect_info: ["verify"], verify: ["collect_info", "awaiting_approval"],
  awaiting_approval: ["execute", "done"], execute: ["done"], done: [],
};
function go(from: State, to: State) {
  if (!transitions[from].includes(to)) throw new Error(`illegal transition ${from} → ${to}`);
}
```

### Failure cases
LLM এর output থেকে state parse করা হচ্ছিল (free text), তাই model "approved" লিখে দিলেই execute। Transition code এ, প্রমাণ (DB flag) দিয়ে।

### Security concerns
State machine নিজেই একটা guardrail: sensitive transition কখনো LLM এর কথায় না, verified event এ।

### Performance concerns
নগণ্য overhead।

### Cost concerns
Per-state ছোট prompt/tools, তাই কম token।

### What did I learn?
- LLM কে "কী বলবে" এর স্বাধীনতা দাও, "কোথায় যাবে" এর না।

### What can I explain to someone else?
> "Traffic signal: লাল থেকে সরাসরি সবুজে না, মাঝে হলুদ। নিয়ম আগে থেকে ঠিক, চালক যতই তাড়াহুড়ো করুক।"

### GitHub Evidence
- [ ] Refund agent as state machine + injection test

### LinkedIn Post
- [ ] "Production AI agent এ state machine কেন জরুরি"

---

<a id="workflow-graphs"></a>

## 4. Workflow Graphs

### What is it?
Agent/workflow কে **directed graph** হিসেবে প্রকাশ: **node** = একটা step (LLM call, tool, function, human input), **edge** = পরের step (conditional edge সহ), cycle (loop) আর parallel branch সম্ভব। LangGraph, Mastra workflows, Inngest এর মতো framework এই model এ চলে।

### Why does it exist?
State machine শুধু state দেখে। Real system এ লাগে branch, loop, parallel fan-out/fan-in, subgraph, checkpoint, সব একসাথে।

### What problem does it solve?
জটিল agentic flow কে দৃশ্যমান, testable, resumable করা।

### How does it work?
```text
        ┌──────────┐
START ─►│ classify │──"billing"──► [billing_agent] ─┐
        └────┬─────┘                                 ├─► [respond] ─► END
             └──"tech"──► [retrieve] ─► [answer] ────┘
                              ▲            │ low confidence
                              └────────────┘ (loop, max 2)
```
Common patterns (Anthropic): prompt chaining, routing, parallelization (sectioning/voting), orchestrator-workers, evaluator-optimizer।

### Important concepts
- **Shared state object** যা node গুলো পড়ে/আপডেট করে।
- **Conditional edges** (router function)।
- **Checkpointing** প্রতি node এর পরে, যাতে crash/HITL এর পরে resume।
- **Subgraphs** (multi-agent কে graph এর ভেতরে graph)।

### Trade-offs
শক্তিশালী ও দৃশ্যমান, কিন্তু framework শেখা আর abstraction এর overhead। ছোট task এ over-engineering।

### Alternatives
সাধারণ TypeScript function (if/else + loop), যা ছোট flow এর জন্য প্রায়ই সবচেয়ে ভালো।

### When should I use it?
Branch/loop/parallel/HITL সহ মাঝারি-বড় flow, যেখানে resume আর visualization দরকার।

### When should I NOT use it?
Linear ৩-step chain। Plain code লিখো।

### Hands-on experiment
একটা support flow (classify → route → retrieve → answer → confidence check loop) আগে plain TS function এ লিখো, তারপর LangGraph.js বা Mastra workflow এ। Code size, readability, resume সুবিধা তুলনা করো।

### Production implementation
Graph definition version করা, প্রতি node trace span, checkpointer (Postgres), graph visualization docs এ।

### Failure cases
Conditional edge এ অপ্রত্যাশিত value ("Billing" vs "billing"), তাই কোনো edge match করেনি, ফলে graph আটকে গেছে। Enum + default edge।

### Security concerns
Sensitive node এর আগে interrupt (HITL) node।

### Performance concerns
Parallel branch দিয়ে দ্রুত। Checkpoint write এ সামান্য overhead।

### Cost concerns
Deterministic node (code) এ LLM খরচ নেই। যত বেশি কাজ code এ, তত সস্তা।

### What did I learn?
- Agent architecture আসলে graph design: কোথায় LLM সিদ্ধান্ত নেবে, কোথায় code।

### What can I explain to someone else?
> "Metro map এর মতো: station (কাজ), লাইন (পরের কাজ), junction এ কোন দিকে যাবে সেটা নির্ভর করে টিকিটের গন্তব্যের উপর।"

### GitHub Evidence
- [ ] Same flow: plain TS vs graph framework

### LinkedIn Post
- [ ] "Anthropic এর ৫টা agentic workflow pattern সহজ ভাষায়"

---

<a id="agent-state"></a>

## 5. Agent State

### What is it?
একটা agent run চলাকালীন সব data: messages/history, current plan ও step, tool results, intermediate output, status (`running | waiting_approval | failed | done`), counters (steps, tokens, cost), user/tenant info, error।

### Why does it exist?
Agent loop এর প্রতিটা iteration আগের সব কিছুর উপর নির্ভর করে। আর long-running বা HITL agent এ process থামলে state disk এ না থাকলে সব হারায়।

### What problem does it solve?
Resumability, debugging (কী অবস্থায় কী সিদ্ধান্ত), observability, আর multi-step coordination।

### How does it work?
```ts
type AgentState = {
  runId: string; userId: string; status: "running" | "waiting_approval" | "failed" | "done";
  messages: Anthropic.MessageParam[];
  plan?: { steps: { id: number; goal: string; status: "todo" | "done" | "failed" }[] };
  artifacts: Record<string, unknown>;          // intermediate results
  usage: { steps: number; inputTokens: number; outputTokens: number; costUsd: number };
  pendingApproval?: { tool: string; input: unknown };
};
// প্রতিটা step এর পরে: save(state)  → crash হলে load(runId) থেকে resume
```

### Important concepts
- **Single source of truth**: state object, LLM এর "মনে রাখা" না।
- **Serializable** (JSON), যাতে DB তে রাখা যায়।
- **Messages ≠ state**: messages model এর জন্য, structured state code এর জন্য। দুটোই রাখো।
- **Immutable updates / event log**: audit ও replay এর জন্য।

### Trade-offs
Rich state = শক্তিশালী resume/debug, কিন্তু schema maintain আর storage।

### Alternatives
শুধু messages array (সহজ, কিন্তু structured query কঠিন)।

### When should I use it?
সব production agent এ।

### When should I NOT use it?
Single-call script এ।

### Hands-on experiment
তোমার ReAct loop এ প্রতি step এর পরে state JSON file এ save করো। Step 3 এ process kill করো (`Ctrl+C`), তারপর load করে resume। কাজ করে?

### Production implementation
`agent_runs` table (state JSONB, status, timestamps) + `agent_events` (append-only)। Framework checkpointer (LangGraph Postgres saver ইত্যাদি)।

### Failure cases
State এ শুধু text রাখা হয়েছিল, tool_use block বাদ, তাই resume এর পর API error।

### Security concerns
State এ secrets (API token) রাখবে না। PII encrypt। User শুধু নিজের run access করতে পারবে।

### Performance concerns
বড় state প্রতি step এ write, তাই diff/event-based storage।

### Cost concerns
Storage সস্তা। Resume এর ফলে fail হলে পুরো run আবার না চালিয়ে LLM খরচ বাঁচে।

### What did I learn?
- Agent এর "মন" না, state ই তার আসল স্মৃতি।

### What can I explain to someone else?
> "Video game এর save file: কোন level, কত HP, inventory তে কী। Game বন্ধ হলেও save থেকে আবার শুরু।"

### GitHub Evidence
- [ ] Kill-and-resume agent demo

### LinkedIn Post
- [ ] —

---

<a id="agent-memory"></a>

## 6. Agent Memory

### What is it?
Agent architecture এ memory এর **জায়গা আর flow**: short-term (state/messages, run এর ভেতরে) আর long-term (store, run এর বাইরে), আর এদের মাঝে read/write কখন হয়। Memory এর ধরন: [agent-memory.md](agent-memory.md) ও [Phase 3 memory](../03-context/memory.md)।

### Why does it exist?
Agent architecture ঠিক করতে হয়: memory কোথায় থাকবে, কে লিখবে (agent নিজে tool দিয়ে, নাকি background job), কে পড়বে (প্রতি run শুরুতে, নাকি দরকারমতো tool দিয়ে)।

### What problem does it solve?
Run-to-run continuity আর learning, architecture-level সিদ্ধান্তের মাধ্যমে।

### How does it work?
```text
run start ──► load: user profile + relevant memories ──► context
   loop  ──► (optional) memory tools: memory_search / memory_write
run end  ──► background: extract facts/episodes/lessons ──► store
```

### Important concepts
- **Hot path vs background** memory write।
- **Memory as tool** (agent নিজে সিদ্ধান্ত নেয়) বনাম **memory as context** (code সবসময় load করে)।
- **Scope**: per-user, per-agent, shared/team।
- Claude এ memory tool (`memory_20250818`), Managed Agents এ memory store।

### Trade-offs
Agent-controlled memory = flexible কিন্তু অনিশ্চিত। Code-controlled = predictable কিন্তু কম adaptive।

### Alternatives
Memory ছাড়া stateless agent (প্রতি run নতুন)।

### When should I use it?
Repeat user, long-term project, শেখা agent।

### When should I NOT use it?
One-shot, anonymous, privacy-sensitive task।

### Hands-on experiment
দুই version: (A) run শুরুতে সবসময় memory load, (B) agent কে `memory_search` tool দাও। ১০টা run এ কোনটা বেশি relevant memory ব্যবহার করে আর কোনটায় token কম দেখো।

### Production implementation
Memory service আলাদা (API), agent শুধু interface জানে। Write এ validation + rate limit।

### Failure cases
Agent প্রতিটা ছোট কথা memory তে লিখে store ভরিয়ে ফেলেছে।

### Security concerns
Memory poisoning (দেখো [long-term memory](../03-context/memory.md#long-term-memory))।

### Performance concerns
Load-at-start এ এক query। Tool-based এ প্রতি search এক round-trip।

### Cost concerns
Memory extraction background এ ছোট model দিয়ে।

### What did I learn?
- Memory architecture = "কখন পড়বে, কখন লিখবে, কে সিদ্ধান্ত নেবে"।

### What can I explain to someone else?
> "অফিসে প্রতিদিন সকালে আগের দিনের নোট পড়ে কাজ শুরু, আর দিনশেষে নতুন নোট লিখে রাখা। প্রশ্ন হলো নোট কে লিখবে: কর্মী নিজে, নাকি একজন সেক্রেটারি।"

### GitHub Evidence
- [ ] Memory-as-context vs memory-as-tool comparison

### LinkedIn Post
- [ ] —

---

⬅️ Back to [Phase 5 — Agents & Agentic Systems](README.md) · [Root README](../README.md)

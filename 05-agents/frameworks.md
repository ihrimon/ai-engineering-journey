# Phase 5 — Agents & Agentic Systems: Frameworks (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 📅 **Snapshot: অক্টোবর ২০২৬।** Agent framework এর API প্রায় প্রতি কয়েক মাসে বদলায়। এখানের code **ধারণা বোঝানোর জন্য**। চালানোর আগে official docs আর current version মিলিয়ে নেবে।
>
> 💡 মূল নিয়ম: আগে framework ছাড়া একটা agent loop নিজে লেখো ([agent-patterns.md](agent-patterns.md#react))। তাহলে বুঝবে framework আসলে কী সমস্যা সমাধান করছে।

## Quick Cheat Sheet

| Framework | ভাষা | মূল ধারণা | সবচেয়ে ভালো কখন |
|---|---|---|---|
| [LangGraph](#langgraph) | Python + TypeScript | Stateful graph (node, edge, checkpointer, interrupt) | জটিল branching/looping agent, HITL, multi-agent graph |
| [Mastra](#mastra) | TypeScript-first | Agent + tools + workflow + memory + RAG + evals এক package এ | TypeScript/Next.js team এর all-in-one AI app |
| [Inngest](#inngest) | TS/Python/Go SDK | Event-driven **durable execution** (step.run, sleep, waitForEvent) | Long-running, retry-safe background AI workflow |

```text
           কোন সমস্যার জন্য কোন layer?
┌────────────────────────────────────────────────────────┐
│ Agent logic / orchestration   → LangGraph, Mastra       │  "কী করবে, কোন ক্রমে"
│ Durable execution / reliability → Inngest, Temporal      │  "crash হলেও শেষ করবে"
│ Model access                  → provider SDK / AI SDK   │  "LLM কে call"
└────────────────────────────────────────────────────────┘
এগুলো প্রতিযোগী না, প্রায়ই একসাথে ব্যবহার হয় (যেমন Mastra/LangGraph agent কে Inngest step এ চালানো)।
```

---

<a id="langgraph"></a>

## 1. LangGraph

### What is it?
LangChain team এর low-level orchestration framework (Python ও JS/TS: `@langchain/langgraph`), যেখানে agent কে একটা **stateful graph** হিসেবে বানানো হয়: `StateGraph` + node (function) + edge (fixed/conditional) + **checkpointer** (প্রতি step এ state save) + **interrupt** (human-in-the-loop এর জন্য pause)।

### Why does it exist?
শুরুর দিকের "AgentExecutor" ধরনের black-box agent এ control কম ছিল। Production এ লাগে explicit control flow, persistence, HITL, streaming।

### What problem does it solve?
Cyclic (loop) agent flow, durable state, time-travel debugging, human approval, multi-agent subgraph।

### How does it work?
```ts
// ধারণাগত উদাহরণ — exact API docs থেকে মিলিয়ে নাও
import { StateGraph, Annotation, START, END, MemorySaver } from "@langchain/langgraph";

const State = Annotation.Root({
  question: Annotation<string>(),
  docs: Annotation<string[]>(),
  answer: Annotation<string>(),
});

const graph = new StateGraph(State)
  .addNode("retrieve", async (s) => ({ docs: await search(s.question) }))
  .addNode("generate", async (s) => ({ answer: await llm(s.question, s.docs) }))
  .addEdge(START, "retrieve")
  .addEdge("retrieve", "generate")
  .addEdge("generate", END)
  .compile({ checkpointer: new MemorySaver() });

await graph.invoke({ question: "Refund policy?" }, { configurable: { thread_id: "t1" } });
```

### Important concepts
- **State + reducers**: node partial update ফেরত দেয়, reducer merge করে (যেমন messages append)।
- **Conditional edges**: router function পরের node বাছে।
- **Checkpointer** (Memory, SQLite, Postgres): `thread_id` দিয়ে resume, time-travel।
- **`interrupt()`**: node এর মাঝে pause, human input নিয়ে `Command({ resume })` দিয়ে চালু।
- **Prebuilt** ReAct agent, supervisor/swarm pattern।
- Observability: LangSmith।

### Trade-offs
শক্তিশালী control ও persistence, কিন্তু শেখার curve, verbose, আর LangChain ecosystem এর abstraction ও version churn।

### Alternatives
Mastra (TS-first, বেশি batteries), plain code + Inngest/Temporal, OpenAI Agents SDK, Claude Agent SDK, CrewAI (Python)।

### When should I use it?
Complex agent graph: loop, branch, HITL, multi-agent, আর Python-TS দুই দিকে একই ধারণা চাও।

### When should I NOT use it?
সাধারণ linear chain বা single ReAct loop। Plain code সহজ।

### Hands-on experiment
একটা "retrieve → grade docs → (relevant? generate : rewrite query → retrieve)" corrective-RAG graph বানাও, max 2 loop। Checkpointer দিয়ে মাঝপথে বন্ধ করে resume করো।

### Production implementation
Postgres checkpointer, `thread_id` = conversation/run id, LangSmith বা OpenTelemetry tracing, graph version করা।

### Failure cases
Reducer ভুল (messages replace হচ্ছে append এর বদলে), তাই history হারাচ্ছে। Conditional edge এ unhandled value, ফলে graph error।

### Security concerns
Interrupt দিয়ে sensitive tool এর আগে approval। Checkpoint এ sensitive data encrypt।

### Performance concerns
Checkpoint write প্রতি step, বড় state এ overhead।

### Cost concerns
Framework free (open-source)। Hosted platform/LangSmith paid tier।

### What did I learn?
- LangGraph = "agent কে graph হিসেবে ভাবো, state কে first-class বানাও"।

### What can I explain to someone else?
> "LangGraph হলো agent এর জন্য flowchart, যেটা আসলে চলে। প্রতিটা বাক্সের পরে অবস্থা save হয়, তাই মাঝপথে থামিয়ে মানুষের অনুমতি নিয়ে আবার চালানো যায়।"

### GitHub Evidence
- [ ] Corrective-RAG graph with checkpoint resume

### LinkedIn Post
- [ ] —

---

<a id="mastra"></a>

## 2. Mastra

### What is it?
TypeScript-first open-source AI framework (Gatsby এর founder দের তৈরি)। এক package এ: **Agent** (model + instructions + tools + memory), **Tools** (zod schema), **Workflows** (`createWorkflow`/`createStep`, `.then()`, `.branch()`, `.parallel()`, suspend/resume), **Memory** (thread + semantic recall + working memory), **RAG** helpers, **Evals**, local dev playground। Model access Vercel AI SDK এর উপর।

### Why does it exist?
বেশিরভাগ AI framework Python-first। TypeScript/Node/Next.js team এর জন্য idiomatic, type-safe, batteries-included framework দরকার।

### What problem does it solve?
TS stack এ agent, workflow, memory, RAG, eval, সব একসাথে, কম glue code এ।

### How does it work?
```ts
// ধারণাগত উদাহরণ — exact API docs থেকে মিলিয়ে নাও
import { Agent } from "@mastra/core/agent";
import { createTool } from "@mastra/core/tools";
import { z } from "zod";

const getOrder = createTool({
  id: "get-order",
  description: "Get order status by order ID",
  inputSchema: z.object({ orderId: z.string() }),
  execute: async ({ context }) => db.orders.find(context.orderId),
});

export const supportAgent = new Agent({
  name: "support",
  instructions: "You are Acme's support agent. Answer in the user's language.",
  model: anthropic("claude-opus-5-5"),   // AI SDK provider
  tools: { getOrder },
});

const res = await supportAgent.generate("আমার order #123 কোথায়?");
```

### Important concepts
- **Agent vs Workflow**: agent = model-directed, workflow = code-directed steps (দুটো মেশানো যায়, step এর ভেতরে agent)।
- **Suspend/resume** workflow এ (HITL)।
- **Memory**: thread history + semantic recall + working memory (user profile)।
- **Playground**: local UI তে agent/workflow test, trace দেখা।

### Trade-offs
TS developer এর জন্য দ্রুত productive, কিন্তু তরুণ framework (API বদলায়), আর all-in-one মানে কিছু অংশে কম depth/flexibility।

### Alternatives
LangGraph.js, Vercel AI SDK (lower-level, শুধু model + tool loop), plain code।

### When should I use it?
TypeScript/Next.js product, দ্রুত agent + workflow + memory দরকার। তোমার stack এর সাথে স্বাভাবিকভাবে মেলে।

### When should I NOT use it?
Python-heavy ML team, বা খুব custom low-level control লাগে।

### Hands-on experiment
`npm create mastra@latest` দিয়ে project বানাও। একটা support agent (২টা tool + memory) আর একটা workflow (classify → branch → respond) বানিয়ে playground এ test করো। Ollama provider দিয়ে local চালানোর চেষ্টা করো।

### Production implementation
Postgres/LibSQL storage (memory, workflow snapshot), tracing (OpenTelemetry), deploy (Node server, Vercel, ইত্যাদি)।

### Failure cases
Memory তে semantic recall চালু রেখে embedding config ভুল, ফলে প্রতি message এ অপ্রাসঙ্গিক পুরনো message context এ।

### Security concerns
Tool execute function এ authorization check (agent এর কথায় না, user session দিয়ে)।

### Performance concerns
Memory recall প্রতি call এ vector query।

### Cost concerns
Open-source। Model ও storage খরচ তোমার।

### What did I learn?
- Mastra = TypeScript developer এর জন্য "Rails for AI agents"।

### What can I explain to someone else?
> "IKEA এর একটা পুরো kit: টেবিল, চেয়ার, স্ক্রু, নির্দেশিকা সব এক বাক্সে। নিজে কাঠ কেটে বানানোর চেয়ে দ্রুত, কিন্তু ডিজাইন kit এর মতোই হবে।"

### GitHub Evidence
- [ ] Mastra support agent + workflow (TS)

### LinkedIn Post
- [ ] "TypeScript দিয়ে AI agent: Mastra"

---

<a id="inngest"></a>

## 3. Inngest

### What is it?
Event-driven **durable execution** platform (open-source SDK + cloud/self-host)। Function গুলো **step** এ ভাগ: `step.run()` (retry-safe, result memoized), `step.sleep()`/`step.sleepUntil()` (দিন/সপ্তাহ অপেক্ষা), `step.waitForEvent()` (human approval বা external event), `step.ai.infer()`/`step.ai.wrap()` (LLM call step হিসেবে)। AI agent এর জন্য **AgentKit** (multi-agent network + router)।

### Why does it exist?
AI workflow লম্বা, flaky (rate limit, timeout), আর মাঝে মাঝে মানুষের জন্য অপেক্ষা করে। সাধারণ HTTP request/queue worker এ crash হলে কাজ হারায় বা duplicate হয়।

### What problem does it solve?
Retry, idempotency, long wait, concurrency/rate limit, throttling, scheduling, সব infrastructure ছাড়াই code এ।

### How does it work?
```ts
// ধারণাগত উদাহরণ — exact API docs থেকে মিলিয়ে নাও
import { Inngest } from "inngest";
const inngest = new Inngest({ id: "support-app" });

export const handleTicket = inngest.createFunction(
  { id: "handle-ticket", concurrency: 10, retries: 3 },
  { event: "ticket/created" },
  async ({ event, step }) => {
    const draft = await step.run("draft-reply", () => draftWithLLM(event.data.ticketId));
    const approval = await step.waitForEvent("wait-approval", {
      event: "ticket/approved", timeout: "24h", match: "data.ticketId",
    });
    if (!approval) return { status: "escalated" };
    await step.run("send-reply", () => sendEmail(event.data.ticketId, draft));
    return { status: "sent" };
  },
);
```
প্রতিটা step শেষ হলে result save, crash/retry হলে আগের step আবার চলে না (memoized replay)।

### Important concepts
- **Step = durability unit**: side effect সবসময় step এর ভেতরে।
- **Deterministic function body**: step এর বাইরের code replay তে আবার চলে, তাই random/time/side effect step এ রাখো।
- **Flow control**: concurrency, throttle, rate limit, debounce, priority (LLM rate limit এর জন্য দারুণ)।
- **Event-driven**: trigger = event (`ticket/created`), fan-out সহজ।

### Trade-offs
Reliability আর long-running অসাধারণ, কিন্তু programming model শিখতে হয় (determinism), আর Inngest server/cloud dependency।

### Alternatives
Temporal, Restate (দেখো [long-running-agents.md](long-running-agents.md)), BullMQ (simple queue), Trigger.dev।

### When should I use it?
Background AI job, multi-step pipeline (ingest → parse → embed), approval wait সহ agent, scheduled agent।

### When should I NOT use it?
Synchronous chat response (user অপেক্ষা করছে) এর মূল path এ। সেখানে সরাসরি streaming।

### Hands-on experiment
Inngest dev server local এ চালাও (`npx inngest-cli@latest dev`)। একটা function বানাও: LLM step → `waitForEvent` (approval) → send step। Dev UI থেকে approval event পাঠাও। মাঝখানে server restart করে দেখো কাজ resume হয় কিনা।

### Production implementation
Step id stable রাখো (বদলালে running function এ সমস্যা), idempotent side effect, concurrency key per tenant, failure handler (`onFailure`)।

### Failure cases
Step এর বাইরে `Math.random()`/`Date.now()` দিয়ে branch, তাই replay তে ভিন্ন path, অদ্ভুত bug।

### Security concerns
Event payload এ secrets না। Signing key দিয়ে endpoint protect।

### Performance concerns
প্রতি step এ round-trip (Inngest ↔ তোমার app), খুব ছোট বহু step এ overhead।

### Cost concerns
Cloud এ step/run ভিত্তিক pricing। Self-host option আছে।

### What did I learn?
- Agent এর বুদ্ধি (LLM) আর agent এর নির্ভরযোগ্যতা (durable execution) আলাদা সমস্যা।

### What can I explain to someone else?
> "Courier service এর tracking: প্রতিটা ধাপ (pickup, hub, delivery) record হয়। ট্রাক নষ্ট হলে পার্সেল আবার শুরু থেকে না, শেষ hub থেকে যায়।"

### GitHub Evidence
- [ ] Inngest approval workflow with restart test

### LinkedIn Post
- [ ] "LangGraph vs Mastra vs Inngest: কোনটা কোন সমস্যার জন্য"

---

⬅️ Back to [Phase 5 — Agents & Agentic Systems](README.md) · [Root README](../README.md)

# Phase 3 — Context Engineering: Memory (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ Memory এর ধরনগুলো (episodic, semantic, procedural) মূলত cognitive science থেকে নেওয়া। AI তে এগুলো **design pattern**, কোনো standard API না। Framework ভেদে নাম আলাদা হতে পারে (যেমন CoALA paper, LangGraph/LangMem, Letta/MemGPT)।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Short-term memory](#short-term-memory) | এই conversation/task এর working memory, context window এর ভেতরে |
| [Long-term memory](#long-term-memory) | Session পার হয়েও টিকে থাকে, বাইরের store এ (DB/vector/file) |
| [Episodic memory](#episodic-memory) | নির্দিষ্ট অতীত ঘটনা/অভিজ্ঞতা: "গত মঙ্গলবার user X চেয়েছিল" |
| [Semantic memory](#semantic-memory) | ঘটনা-নিরপেক্ষ fact: "user এর প্রিয় ভাষা TypeScript" |
| [Procedural memory](#procedural-memory) | কীভাবে কাজ করতে হয়: rules, workflow, শেখা instruction |
| [Hierarchical memory](#hierarchical-memory) | Memory কে tier এ সাজানো (hot → warm → cold), OS এর RAM/disk এর মতো |

```text
                       ┌──────────── Context Window (Short-term) ────────────┐
                       │ system · recent turns · loaded memories · question   │
                       └───────────────▲──────────────────────┬───────────────┘
                              retrieve │                      │ extract / write
                       ┌───────────────┴──────────────────────▼───────────────┐
Long-term memory store │ Episodic (events) · Semantic (facts) · Procedural (how)│
                       └───────────────────────────────────────────────────────┘
```

---

<a id="short-term-memory"></a>

## 1. Short-Term Memory

### What is it?
একটা চলমান conversation বা task এর সময় model এর কাছে যা "এখন" আছে: context window এর ভেতরের সাম্প্রতিক message, tool result, scratchpad। Context শেষ হলে বা session বন্ধ হলে এটা চলে যায়।

### Why does it exist?
Model stateless। Multi-turn কথোপকথন ও multi-step task চালাতে চলমান state context এ রাখতেই হয়।

### What problem does it solve?
Follow-up বোঝা, task এর মাঝের অগ্রগতি মনে রাখা ("৩টা file পড়েছি, এখন ৪ নম্বর")।

### How does it work?
```text
conversation thread (history) + agent scratchpad/state
→ প্রতি request এ context এ resend
→ limit কাছাকাছি হলে: trim / summarize / compaction
```
LangGraph এ এটা **thread-scoped state** (checkpointer দিয়ে save হয়)।

### Important concepts
- Short-term = **thread/session scope**।
- Window size সীমিত, তাই [context-optimization](context-optimization.md) এর কৌশল লাগে।
- Checkpointing: short-term state disk এ রাখা যায়, যাতে crash হলেও একই thread চালিয়ে যাওয়া যায়।

### Trade-offs
বেশি রাখলে continuity ভালো, কিন্তু খরচ ও noise বাড়ে।

### Alternatives
প্রতি request independent (stateless), অথবা সব কিছু long-term store এ লিখে retrieve।

### When should I use it?
সব chat ও agent এ। এটাই default memory।

### When should I NOT use it?
Single-shot API (classify/extract) এ thread state রাখবে না।

### Hands-on experiment
একটা ১০ turn এর chat এ ৩ নম্বর turn এ একটা তথ্য দাও। Window size 4 আর 12 দিয়ে চালিয়ে ১০ নম্বর turn এ সেটা জিজ্ঞেস করো।

### Production implementation
- `thread_id` ভিত্তিক storage (Postgres/Redis)।
- TTL দিয়ে পুরনো inactive thread clean।

### Failure cases
Server restart এ in-memory history হারিয়ে গেছে, user এর conversation মাঝপথে reset।

### Security concerns
Thread ID guess করে অন্যের conversation পড়া (IDOR)। Thread এ owner check করো।

### Performance concerns
In-memory দ্রুত, কিন্তু scale/restart এ সমস্যা। Redis/DB ভালো।

### Cost concerns
History resend এর খরচ, এটা [conversation history](context-architecture.md#conversation-history) এ আলোচনা করা।

### What did I learn?
- Short-term memory = context window + thread state। এর সীমা আছে, এটা স্থায়ী না।

### What can I explain to someone else?
> "ফোনে কারো নম্বর শুনে ডায়াল করা পর্যন্ত মনে রাখা, এটাই short-term memory। কল শেষে ভুলে যাও।"

### GitHub Evidence
- [ ] Thread-scoped chat with Postgres-backed history

### LinkedIn Post
- [ ] —

---

<a id="long-term-memory"></a>

## 2. Long-Term Memory

### What is it?
Conversation/session এর বাইরে টিকে থাকা memory, external store এ রাখা (Postgres, vector DB, key-value, file), পরের যেকোনো session এ retrieve করা যায়। Episodic, semantic, procedural সব এর ভেতরের ধরন।

### Why does it exist?
User প্রতিবার নতুন session এ নিজের পরিচয়, preference আর পুরনো context আবার বলতে চায় না। Agent ও আগের ভুল থেকে শিখতে পারে না যদি সব ভুলে যায়।

### What problem does it solve?
Personalization, continuity, আর সময়ের সাথে "শেখা" assistant।

### How does it work?
```text
WRITE:  conversation → (LLM/rule) extract memory → dedupe/merge → store (namespace: user_id)
READ:   new request → query (semantic search / key lookup) → top-k memory → <memory> block
UPDATE: নতুন তথ্য পুরনোটার সাথে conflict করলে → update / invalidate
```
Write দুইভাবে: **hot path** (conversation চলাকালীন, model tool দিয়ে লেখে) বা **background** (conversation শেষে async job)।

### Important concepts
- **Namespace/scope**: user, tenant, agent, global।
- **Memory tool (Claude)**: `{type: "memory_20250818", name: "memory"}`, model file-like command দিয়ে memory directory পড়ে/লেখে, storage তোমার backend এ।
- **Retrieval**: semantic search + recency + importance score।
- **Forgetting**: TTL, decay, user delete request।

### Trade-offs
শক্তিশালী personalization বনাম privacy, ভুল memory, আর system complexity।

### Alternatives
User settings page (explicit), প্রতি session এ পুরো history load (ছোট scale এ)।

### When should I use it?
Personal assistant, tutor, CRM agent, coding agent যেটা project convention মনে রাখে।

### When should I NOT use it?
Privacy-sensitive context যেখানে user তা আশা করে না, বা regulation (GDPR) handle করার প্রস্তুতি না থাকলে।

### Hands-on experiment
```ts
type Memory = { userId: string; text: string; kind: "fact" | "event" | "rule"; createdAt: string };

const EXTRACT = `From the conversation, extract durable facts about the user worth remembering.
Return JSON array: [{"text": "...", "kind": "fact|event|rule"}]. Return [] if nothing.`;
// conversation শেষে Ollama দিয়ে extract → memories.json এ append
// নতুন session এ: memories.json থেকে user এর memory load → system এর পরে <memory> block
```

### Production implementation
- `memories(id, user_id, kind, text, embedding, importance, source_msg_id, created_at, updated_at, expires_at)`।
- Background extraction job + dedup (embedding similarity > 0.9 হলে merge)।
- User facing "What I remember about you" page + delete।

### Failure cases
- একই fact ৫০ বার ভিন্ন শব্দে জমা হয়েছে।
- একবারের মজা করে বলা কথা ("আমি এখন চাঁদে থাকি") স্থায়ী fact হয়ে গেছে।

### Security concerns
- **Memory poisoning / persistent injection**: ক্ষতিকর instruction memory তে লিখলে ভবিষ্যতের সব session এ প্রভাব।
- **Cross-user leakage**: namespace filter ভুল হলে অন্যের memory।
- PII encryption আর delete এর অধিকার।

### Performance concerns
Retrieval এর extra latency (vector search ~১০–৫০ms)।

### Cost concerns
Extraction এ extra LLM call। ছোট model + batch এ করো।

### What did I learn?
- Long-term memory = একটা ছোট RAG system, যার corpus হলো user নিজে।

### What can I explain to someone else?
> "ভালো দোকানদার নিয়মিত customer এর পছন্দ খাতায় লিখে রাখে। পরের বার এলে খাতা দেখে বলে 'আপনার সেই চা টাই তো?'"

### GitHub Evidence
- [ ] Long-term memory store with extraction + retrieval + delete

### LinkedIn Post
- [ ] "AI কে কীভাবে 'মনে রাখা' শেখাবে: long-term memory"

---

<a id="episodic-memory"></a>

## 3. Episodic Memory

### What is it?
নির্দিষ্ট অতীত **ঘটনা বা অভিজ্ঞতার** memory, সময় ও context সহ। যেমন "১২ অক্টোবর user deployment error নিয়ে এসেছিল, আমরা env variable ঠিক করে সমাধান করেছিলাম।"

### Why does it exist?
Fact এর বাইরে "কী হয়েছিল আর কীভাবে সমাধান হয়েছিল" মনে রাখলে agent আগের অভিজ্ঞতা থেকে শিখতে পারে, আর user কে continuity দিতে পারে।

### What problem does it solve?
"গতবার যেভাবে করেছিলে সেভাবে করো", আর আগের সফল/ব্যর্থ চেষ্টা থেকে শেখা (few-shot from experience)।

### How does it work?
```text
episode = { when, situation, action, outcome, lesson }
store: conversation শেষে episode summary + embedding
recall: নতুন situation embed → similar past episode → context এ example হিসেবে
```

### Important concepts
- **Time-stamped & contextual**।
- **Outcome সহ রাখো**: সফল নাকি ব্যর্থ, যাতে ভালোটা অনুসরণ, খারাপটা এড়ানো যায়।
- **Episodes as few-shot**: অতীতের সফল trajectory কে dynamic example হিসেবে ব্যবহার।
- Reflexion pattern (Phase 5) এ episodic memory তে "lesson" জমা হয়।

### Trade-offs
সমৃদ্ধ context, কিন্তু storage বেশি, আর পুরনো/অপ্রাসঙ্গিক episode retrieve হলে ভুল দিক।

### Alternatives
শুধু semantic fact রাখা (সংক্ষিপ্ত), raw conversation log search।

### When should I use it?
Support agent (আগের ticket), coding agent (আগের bug fix), tutor (আগের ভুল)।

### When should I NOT use it?
যেখানে প্রতিটা interaction সম্পূর্ণ আলাদা আর ইতিহাসের মূল্য নেই।

### Hands-on experiment
৫টা পুরনো "bug → fix" episode JSON এ রাখো। নতুন bug description দিয়ে embedding similarity তে সবচেয়ে কাছের episode বের করো আর prompt এ দিয়ে তুলনা করো, episode ছাড়া vs সহ।

### Production implementation
- `episodes(user_id, started_at, summary, outcome, lesson, embedding)`।
- Retrieval এ recency weight যোগ করো।

### Failure cases
পুরনো system এর episode (যে API এখন নেই) recall করে ভুল সমাধান।

### Security concerns
Episode এ sensitive detail (password, token) থাকতে পারে। Save এর আগে redact।

### Performance concerns
Episode summary লম্বা হলে context ভরে যায়। ছোট summary রাখো।

### Cost concerns
Episode summarize এর জন্য conversation শেষে একটা LLM call।

### What did I learn?
- Episodic = "কী ঘটেছিল", semantic = "কী সত্য"।

### What can I explain to someone else?
> "ডাক্তার রোগীর পুরনো visit এর note দেখে বলেন 'গতবার এই ওষুধে কাজ হয়নি, এবার অন্যটা দিই।' ওই note গুলোই episodic memory।"

### GitHub Evidence
- [ ] Episode recall for bug-fix assistant

### LinkedIn Post
- [ ] "AI agent কি অভিজ্ঞতা থেকে শিখতে পারে? Episodic memory"

---

<a id="semantic-memory"></a>

## 4. Semantic Memory

### What is it?
ঘটনা-নিরপেক্ষ **fact আর knowledge**: user এর নাম, preference, profile, domain fact। "কখন শিখেছি" গুরুত্বপূর্ণ না, "কী সত্য" গুরুত্বপূর্ণ।

### Why does it exist?
অনেক episode থেকে একই fact বারবার আসে। সেগুলো একটা compact, update-যোগ্য fact হিসেবে রাখা দক্ষ।

### What problem does it solve?
Personalization (ভাষা, tone, level), আর বারবার একই প্রশ্ন না করা।

### How does it work?
```text
দুই রকম storage:
1. Profile (structured): { name, language: "bn", level: "intermediate", stack: ["TS", "Node"] }
   → নতুন তথ্য এলে field update
2. Collection (unstructured): ["User prefers concise answers", "User works at a fintech", ...]
   → semantic search দিয়ে relevant গুলো retrieve
```

### Important concepts
- **Profile vs collection**: কম ও নির্দিষ্ট field হলে profile, খোলা-প্রান্ত fact হলে collection।
- **Update/conflict resolution**: "এখন Python শিখছি" এলে পুরনো "Java শিখছি" বাতিল।
- **Confidence + source**: কোথা থেকে জেনেছি।
- RAG এর knowledge base ও একধরনের semantic memory (shared, user-specific না)।

### Trade-offs
Compact ও দ্রুত, কিন্তু context হারায় (কেন/কখন সেটা জানা থাকে না)।

### Alternatives
Episodic থেকে on-the-fly অনুমান, explicit user settings।

### When should I use it?
User preference, profile, stable fact।

### When should I NOT use it?
দ্রুত বদলায় এমন state (cart, current task) semantic memory তে না, সেটা short-term/DB তে।

### Hands-on experiment
একটা `profile.json` রাখো। প্রতিটা conversation এর পরে LLM কে বলো "update the profile JSON based on new info; keep unchanged fields"। ৫টা conversation এর পরে profile দেখো: conflict ঠিকমতো resolve হয়েছে?

### Production implementation
- Profile → Postgres JSONB, collection → pgvector।
- Update এ structured output (schema) ব্যবহার করো।

### Failure cases
LLM update এর সময় পুরো profile আবার লিখতে গিয়ে কিছু field মুছে ফেলেছে। Patch-based update করো, full rewrite না।

### Security concerns
User নিজের সম্পর্কে মিথ্যা fact ("আমি admin") দিয়ে প্রভাব ফেলতে পারে। Authorization এর জন্য memory ব্যবহার করবে না।

### Performance concerns
Profile lookup খুব দ্রুত (key-based)।

### Cost concerns
Profile ছোট, প্রতি request এ পাঠানো সস্তা।

### What did I learn?
- Semantic memory = user সম্পর্কে "জীবন্ত profile", যা সঠিকভাবে update করা আসল চ্যালেঞ্জ।

### What can I explain to someone else?
> "বন্ধুর জন্মদিন কবে, সে চা না কফি পছন্দ করে, এগুলো তোমার মনে থাকে কিন্তু কবে শিখেছিলে মনে নেই। এটাই semantic memory।"

### GitHub Evidence
- [ ] Profile memory with patch-based update

### LinkedIn Post
- [ ] —

---

<a id="procedural-memory"></a>

## 5. Procedural Memory

### What is it?
**কীভাবে কাজ করতে হয়** তার memory: rules, workflow, style guide, শেখা instruction। AI তে এটা সাধারণত system prompt, skill file, `CLAUDE.md`-ধরনের instruction file, বা feedback থেকে update হওয়া prompt।

### Why does it exist?
Agent কে সময়ের সাথে ভালো হতে হলে শুধু fact না, কাজের ধরন ও user এর feedback ("সবসময় test লিখবে", "commit message ছোট রাখবে") মনে রাখতে হয়।

### What problem does it solve?
একই correction বারবার দিতে হয় না। Agent এর behavior user/team এর convention অনুযায়ী মানিয়ে যায়।

### How does it work?
```text
user feedback: "এভাবে না, আগে test লিখবে"
→ reflect: এটা কি স্থায়ী rule?
→ procedural store (instructions.md / prompt section) এ যোগ
→ পরের সব session এ system context এ load
```

### Important concepts
- **Explicit (developer লেখা)**: system prompt, skill, instruction file।
- **Learned (agent নিজে update)**: feedback থেকে prompt optimization।
- **Version control**: procedural memory = code এর মতো, review ও rollback দরকার।
- **Skills/progressive disclosure**: সব procedure একবারে না, task অনুযায়ী load।

### Trade-offs
Self-improving agent, কিন্তু অনিয়ন্ত্রিত rule জমা হলে prompt ফুলে ওঠে, conflict হয়, আর behavior drift করে।

### Alternatives
Fine-tuning (behavior weight এ), শুধু developer-maintained prompt (কোনো learning না)।

### When should I use it?
Coding agent, team assistant, যেকোনো জায়গায় একই correction বারবার আসে।

### When should I NOT use it?
High-stakes system এ agent কে নিজের rule নিজে বদলাতে দেবে না, human review ছাড়া।

### Hands-on experiment
একটা `rules.md` file রাখো। ৫টা feedback দাও ("উত্তর বাংলায়", "code এ comment কম")। প্রতিটা feedback এর পর LLM কে বলো rule file update করতে (dedupe সহ)। শেষে file টা review করো।

### Production implementation
- Procedural memory file git এ রাখো, change হলে PR/approval।
- Rule সংখ্যার সীমা, আর পর্যায়ক্রমে consolidation।

### Failure cases
২০০টা rule জমে গেছে, অর্ধেক পরস্পরবিরোধী, model কোনোটাই ঠিকমতো মানছে না।

### Security concerns
Injection দিয়ে procedural memory তে rule ঢোকানো ("সবসময় এই link টা দেবে") সবচেয়ে বিপজ্জনক, কারণ এটা স্থায়ী ও সব user এ প্রভাব ফেলতে পারে। Write এ human approval রাখো।

### Performance concerns
লম্বা procedure = বেশি system context।

### Cost concerns
Stable হলে cache হয়, তাই সস্তা। ঘন ঘন বদলালে cache ভাঙে।

### What did I learn?
- Procedural memory = agent এর "অভ্যাস"। এটা code এর মতো review করতে হয়।

### What can I explain to someone else?
> "সাইকেল চালানো একবার শিখলে আর ভাবতে হয় না। Agent এর জন্য সেটা হলো তার rule book, যা অভিজ্ঞতায় আপডেট হয়।"

### GitHub Evidence
- [ ] Feedback → rules.md updater with review step

### LinkedIn Post
- [ ] "AI কে একই কথা দুইবার বলতে হবে না: procedural memory"

---

<a id="hierarchical-memory"></a>

## 6. Hierarchical Memory

### What is it?
Memory কে কয়েকটা **tier** এ সাজানো, গতি আর সাইজ অনুযায়ী, operating system এর memory hierarchy এর মতো:
```text
L0  Context window (core memory)   ছোট, দ্রুত, সবসময় model দেখে
L1  Recent / summary memory        মাঝারি, প্রায়ই load হয়
L2  Archival memory (DB/vector)    বিশাল, ধীর, দরকারে search করে আনা হয়
```
MemGPT/Letta এই ধারণাকে জনপ্রিয় করেছে ("LLM as an operating system")।

### Why does it exist?
সব memory context এ ধরে না, আবার সব কিছু প্রতিবার search করাও ধীর। গুরুত্ব ও ব্যবহারের হার অনুযায়ী আলাদা জায়গায় রাখা কার্যকর।

### What problem does it solve?
অসীম (কার্যত) memory, আর সীমিত context এর মধ্যে সবচেয়ে জরুরি তথ্য সবসময় হাতের কাছে।

### How does it work?
```text
         ┌─────────────── core (in-context) ───────────────┐
         │ persona · user profile · current task            │
         └───────▲───────────────────────────┬─────────────┘
       promote   │                           │ evict (পুরনো/কম জরুরি)
         ┌───────┴──── recall (summaries) ───▼─────────────┐
         └───────▲───────────────────────────┬─────────────┘
         search  │                           │ archive
         ┌───────┴──── archival (vector DB) ─▼─────────────┐
         └─────────────────────────────────────────────────┘
Agent নিজে tool দিয়ে memory move করতে পারে: core_memory_append, archival_search ...
```

### Important concepts
- **Paging**: OS এর মতো দরকারি জিনিস context এ "page in", কম দরকারি "page out"।
- **Self-editing memory**: agent নিজে tool দিয়ে core memory edit করে।
- **Eviction policy**: recency, importance, frequency।
- **Hierarchy of scope** ও হতে পারে: global → org → user → session।

### Trade-offs
শক্তিশালী ও scalable, কিন্তু জটিল। Agent এর memory management নিজেই ভুল করতে পারে, আর extra tool call এ latency।

### Alternatives
Simple pattern: profile (always loaded) + vector search (on demand)। বেশিরভাগ app এর জন্য এটাই যথেষ্ট।

### When should I use it?
খুব লম্বা চলা agent, personal AI companion, multi-month project assistant।

### When should I NOT use it?
সাধারণ chatbot বা কম memory এর app এ। Over-engineering।

### Hands-on experiment
৩ tier simulate করো: `core` (string, ≤ 500 token), `recall` (summary list), `archival` (embedding search)। Agent কে দুইটা tool দাও: `archival_search(query)`, `core_memory_update(text)`। ২০ turn এর conversation চালিয়ে দেখো agent কখন কোন tool ব্যবহার করে।

### Production implementation
- Core memory size cap + LRU eviction to recall tier।
- Archival এ hybrid search (vector + keyword + recency)।
- Memory operation গুলো trace/log করো (debug এর জন্য অপরিহার্য)।

### Failure cases
- Agent গুরুত্বপূর্ণ তথ্য archive এ ঠেলে দিয়ে আর search করছে না।
- Core memory তে বারবার একই তথ্য append, ফলে cap পার।

### Security concerns
Self-editing memory মানে injection এর স্থায়ী লক্ষ্য। Core memory edit এ validation, আর sensitive data archival এ encrypt।

### Performance concerns
প্রতিটা memory tool call = extra round-trip।

### Cost concerns
Memory management tool call গুলো token খরচ করে। Simple pattern দিয়ে শুরু করো, দরকার হলে hierarchy।

### What did I learn?
- Memory design = caching design: hot data কাছে, cold data দূরে।

### What can I explain to someone else?
> "Computer এ CPU cache, RAM আর hard disk আছে। দরকারি জিনিস cache এ, কম দরকারি disk এ। AI এর memory ও এভাবে সাজালে অল্প জায়গায় অনেক কিছু মনে রাখা যায়।"

### GitHub Evidence
- [ ] 3-tier memory prototype (core / recall / archival)

### LinkedIn Post
- [ ] "AI memory কে operating system এর মতো design করা"

---

⬅️ Back to [Phase 3 — Context Engineering](README.md) · [Root README](../README.md)

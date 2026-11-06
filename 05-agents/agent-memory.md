# Phase 5 — Agents & Agentic Systems: Agent Memory (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> 🔗 Memory এর তত্ত্ব (short/long-term, episodic, semantic, procedural, hierarchical) Phase 3 এ: [memory.md](../03-context/memory.md)। এখানে **agent এর কাজের ভেতরে** memory কীভাবে ব্যবহার হয়: task চলাকালীন scratchpad, agent এর জ্ঞান, agent এর দক্ষতা, আর conversation।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Scratchpad](#scratchpad) | Task চলাকালীন agent এর খসড়া খাতা: plan, notes, intermediate result, todo |
| [Semantic memory](#semantic-memory) | Agent যে fact জানে/শিখেছে: user, domain, environment সম্পর্কে |
| [Procedural memory](#procedural-memory) | Agent এর "কীভাবে করতে হয়": instructions, skills, শেখা workflow |
| [Conversation memory](#conversation-memory) | User এর সাথে কথোপকথনের history, session ও session পার করে |

```text
┌─────────────────────────── Agent ───────────────────────────┐
│  Procedural (how)   → system prompt · skills · rules files   │  প্রায় স্থির
│  Semantic (what)    → user profile · domain facts            │  ধীরে বদলায়
│  Conversation       → thread history (+ summary)             │  প্রতি turn
│  Scratchpad         → plan · todo · notes · partial results  │  প্রতি step, task শেষে মুছে যায়
└──────────────────────────────────────────────────────────────┘
```

---

<a id="scratchpad"></a>

## 1. Scratchpad

### What is it?
একটা task চলার সময় agent এর **working notes**: বর্তমান plan, todo list, এ পর্যন্ত কী পাওয়া গেছে, কোন hypothesis বাতিল, পরের step। হতে পারে context এর ভেতরে (message/thinking), structured state এ, বা একটা file এ (`NOTES.md`, `todo.json`) যা agent tool দিয়ে পড়ে/লেখে।

### Why does it exist?
লম্বা task এ মূল তথ্য অনেক tool result এর নিচে চাপা পড়ে, বা context compaction এ হারায়। একটা আলাদা জায়গায় নোট রাখলে agent focus আর progress ধরে রাখে।

### What problem does it solve?
Long-horizon task এ দিক হারানো, একই কাজ দুইবার করা, compaction এর পরে "কোথায় ছিলাম" ভুলে যাওয়া।

### How does it work?
```text
tools: write_notes(content), read_notes()
agent:  step 1 → write_notes("Plan: 1) logs পড়ো 2) error খোঁজো 3) fix. Done: -")
        ...  step 12 → read_notes() → "এখন step 3"
compaction এর পরেও notes file অক্ষত → agent সেখান থেকে চালিয়ে যায়
```

### Important concepts
- **Externalized** (file/state) scratchpad context reset/compaction টিকে থাকে।
- **Structured todo** (status সহ), যা UI তে progress দেখানো যায়।
- **Task-scoped**: task শেষে archive/মুছে ফেলো।
- Reasoning model এর thinking ও একধরনের internal scratchpad, কিন্তু সেটা persistent না।

### Trade-offs
ভালো focus, কিন্তু agent নোট লিখতে-পড়তে extra tool call/token খরচ করে, আর নোট পুরনো হয়ে ভুল দিকে নিতে পারে।

### Alternatives
শুধু conversation history (ছোট task এ যথেষ্ট), code-managed plan state (Plan-and-Execute)।

### When should I use it?
১৫+ step এর task, coding agent, research agent, compaction ব্যবহার করা agent।

### When should I NOT use it?
ছোট ৩–৫ step task।

### Hands-on experiment
একটা ২০-step simulated task (২০টা file থেকে তথ্য জোগাড়) চালাও, max context ছোট রেখে (শেষ ৬টা message ছাড়া সব prune)। (A) scratchpad ছাড়া, (B) `write_notes`/`read_notes` tool সহ। কোনটা সব ২০টা শেষ করে?

### Production implementation
Scratchpad কে run state এর অংশ হিসেবে DB তে রাখো, UI তে todo progress দেখাও।

### Failure cases
Agent নোটে "step 3 done" লিখেছে কিন্তু আসলে fail হয়েছিল, তাই পরে ভুল ধারণায় এগিয়েছে। গুরুত্বপূর্ণ status code দিয়ে verify।

### Security concerns
Scratchpad এ tool result থেকে injection কপি হয়ে যেতে পারে, যা পরে বারবার পড়া হয়। Untrusted হিসেবে ধরো।

### Performance concerns
প্রতি note read/write = extra round-trip।

### Cost concerns
ছোট, কিন্তু লম্বা নোট বারবার পড়লে token বাড়ে, তাই নোট সংক্ষিপ্ত রাখো।

### What did I learn?
- লম্বা কাজে agent এরও একটা নোটবুক লাগে, মাথায় সব রাখা যায় না।

### What can I explain to someone else?
> "অঙ্ক কষার সময় rough খাতা: মাঝের হিসাব লিখে রাখো যাতে ভুলে না যাও, শেষে শুধু উত্তরটা খাতায় তোলো।"

### GitHub Evidence
- [ ] Scratchpad vs no-scratchpad on 20-step task

### LinkedIn Post
- [ ] "লম্বা কাজে AI agent কেন নোট লেখে"

---

<a id="semantic-memory"></a>

## 2. Semantic Memory

### What is it?
Agent এর **fact-ভিত্তিক জ্ঞান**: user সম্পর্কে (preference, role, project), domain সম্পর্কে (company policy, product fact), environment সম্পর্কে (codebase convention, API endpoint)। Agent এটা retrieve করে (RAG বা profile lookup), আর নতুন fact শিখলে update করে।

### Why does it exist?
প্রতিটা run এ agent কে একই তথ্য আবার আবিষ্কার করতে হলে সময় ও token নষ্ট, আর user বিরক্ত।

### What problem does it solve?
Personalized, informed agent: "এই user সবসময় TypeScript চায়", "এই repo তে pnpm ব্যবহার হয়"।

### How does it work?
```text
read:   run start → profile + top-k relevant facts (query = task) → context
write:  agent tool `remember(fact)` (hot path) বা run শেষে extraction job (background)
update: conflict → নতুন fact পুরনোটা supersede করে (timestamp সহ)
```

### Important concepts
- **Profile (structured) + collection (searchable)** (দেখো [Phase 3](../03-context/memory.md#semantic-memory))।
- **Agent-scoped vs user-scoped vs shared** knowledge।
- **Provenance**: fact কোথা থেকে, কবে।
- **Knowledge base (RAG)** = shared semantic memory।

### Trade-offs
Agent smarter হয়, কিন্তু ভুল/পুরনো fact স্থায়ী হতে পারে।

### Alternatives
প্রতি run এ explicit context (user নিজে দেয়), static config file।

### When should I use it?
Repeat user, long-lived project, personal assistant।

### When should I NOT use it?
Anonymous one-time agent।

### Hands-on experiment
Agent কে `remember(fact)` আর `recall(query)` tool দাও। ৩টা আলাদা run এ user তথ্য দাও ("আমি pnpm ব্যবহার করি", "test এ vitest")। চতুর্থ run এ একটা task দাও যেখানে এই তথ্য লাগে, agent recall করে কিনা দেখো।

### Production implementation
Memory service (Postgres + pgvector), namespace = user/agent, fact dedup, user-visible memory page।

### Failure cases
Agent একটা ভুল অনুমান ("user Java developer") fact হিসেবে লিখে ফেলেছে। "অনুমান" আর "user বলেছে" আলাদা label করো।

### Security concerns
Cross-user leakage, memory poisoning। Namespace filter code এ।

### Performance concerns
Recall query প্রতি run শুরুতে।

### Cost concerns
Extraction এ ছোট model।

### What did I learn?
- Agent এর semantic memory = user ও কাজের জগৎ সম্পর্কে তার "জ্ঞানকোষ"।

### What can I explain to someone else?
> "পুরনো সহকর্মী জানে তুমি কোন tool পছন্দ করো, কোন নিয়ম মানো। নতুন সহকর্মীকে সব আবার শেখাতে হয়।"

### GitHub Evidence
- [ ] remember/recall tool agent across runs

### LinkedIn Post
- [ ] —

---

<a id="procedural-memory"></a>

## 3. Procedural Memory

### What is it?
Agent এর **কাজ করার পদ্ধতি**: system prompt এর instruction, skill file (task-specific নির্দেশিকা যা দরকারমতো load হয়), rules/convention file (`CLAUDE.md`, `AGENTS.md` এর মতো), আর অভিজ্ঞতা থেকে শেখা workflow/lesson।

### Why does it exist?
একই ধরনের কাজ বারবার আসে, প্রতিবার model কে নতুন করে পদ্ধতি বের করতে দিলে inconsistent ও ধীর। আর user/team এর correction স্থায়ী করা দরকার।

### What problem does it solve?
Consistent behavior, team convention মানা, আর একই ভুল বারবার না করা।

### How does it work?
```text
static:    system prompt + rules file (repo তে, version controlled)
on-demand: skills/ ─ "pdf-report/SKILL.md" ─ task মিললে load (progressive disclosure)
learned:   feedback → reflection → "lessons.md" এ rule প্রস্তাব → human review → merge
```

### Important concepts
- **Progressive disclosure**: সব skill একবারে context এ না, শুধু নাম/description, দরকার হলে পুরোটা।
- **Version control + review**: procedural memory বদলানো = agent এর behavior বদলানো।
- **Rule bloat**: সময়ের সাথে rule জমে conflict, তাই নিয়মিত consolidation।
- Fine-tuning = procedural memory কে weight এ বসানো (চরম রূপ)।

### Trade-offs
Self-improving ও consistent, কিন্তু খারাপ rule স্থায়ী ক্ষতি করে, আর prompt ফুলে ওঠে।

### Alternatives
Fine-tuning, hard-coded workflow (code এ পদ্ধতি)।

### When should I use it?
Repeated task type, team/organization convention, coding agent।

### When should I NOT use it?
Agent কে human review ছাড়া নিজের rule বদলাতে দেবে না।

### Hands-on experiment
তোমার agent এর জন্য ২টা skill file লিখো (যেমন `write-learning-log` আর `review-typescript`)। System prompt এ শুধু skill এর নাম আর এক লাইন বর্ণনা দাও, আর `load_skill(name)` tool দাও। Task অনুযায়ী agent সঠিক skill load করে কিনা দেখো।

### Production implementation
Skills/rules git এ, PR review, changelog, behavior eval (rule বদলালে regression test)।

### Failure cases
Lessons file এ ১৫০ rule, অর্ধেক পরস্পরবিরোধী, agent কোনটাই ঠিকমতো মানছে না।

### Security concerns
Injection দিয়ে procedural memory তে rule লেখা সবচেয়ে বিপজ্জনক (স্থায়ী, সব run এ)। Write এ human approval।

### Performance concerns
Progressive disclosure context ছোট রাখে।

### Cost concerns
Stable procedure cache হয়। ঘন ঘন বদলালে cache miss।

### What did I learn?
- Procedural memory = agent এর "অভ্যাস আর দক্ষতা", code এর মতো review করতে হয়।

### What can I explain to someone else?
> "নতুন কর্মীর জন্য SOP manual: প্রতিটা কাজের ধাপ লেখা থাকে। কাজের সময় শুধু দরকারি অধ্যায় খোলে, আর manual বদলাতে manager এর অনুমোদন লাগে।"

### GitHub Evidence
- [ ] Skill files + load_skill tool demo

### LinkedIn Post
- [ ] "AI agent এর skill: দরকারমতো জ্ঞান load করা"

---

<a id="conversation-memory"></a>

## 4. Conversation Memory

### What is it?
User আর agent এর কথোপকথনের history: চলমান thread (short-term) আর অতীত session (long-term, summary বা searchable archive হিসেবে)। Agent এর ক্ষেত্রে এতে tool call আর result ও থাকে।

### Why does it exist?
User আগের কথার উপর ভিত্তি করে কথা বলে ("আগেরটার মতো আরেকটা করো", "গত সপ্তাহে যে report বানিয়েছিলে")।

### What problem does it solve?
Multi-turn continuity, আর session পার হলেও প্রসঙ্গ ধরে রাখা।

### How does it work?
```text
current thread:  full messages (budget এর মধ্যে) → overflow হলে rolling summary/compaction
past sessions:   session শেষে summary → store (embedding সহ)
                 নতুন session: "আগের relevant session summary" retrieve → context
```

### Important concepts
- **Thread vs cross-session**।
- **Tool call history trimming**: পুরনো tool result clear করো, কিন্তু সিদ্ধান্ত রাখো।
- **Rolling summary / compaction** (দেখো [Phase 3](../03-context/context-optimization.md#rolling-memory))।
- **Session summary as episodic memory**।

### Trade-offs
পূর্ণ history = সর্বোচ্চ continuity, সর্বোচ্চ খরচ। Summary = সস্তা, তথ্য হারায়।

### Alternatives
Stateless agent, user নিজে context দেয়।

### When should I use it?
Chat-based agent, assistant, support।

### When should I NOT use it?
Batch/background agent যেখানে কোনো conversation নেই।

### Hands-on experiment
দুইটা session চালাও। Session 1 শেষে summary বানিয়ে save করো। Session 2 তে "গতবার যে plan করেছিলাম সেটা আপডেট করো" বলো, summary load করে ও না করে তুলনা করো।

### Production implementation
`conversations` + `messages` + `session_summaries` table, thread-level token budget, user কে conversation delete এর অধিকার।

### Failure cases
Session summary তে মূল সিদ্ধান্ত বাদ পড়েছে, তাই agent বিপরীত কাজ করেছে।

### Security concerns
Conversation এ sensitive তথ্য, retention policy, encryption, per-user access।

### Performance concerns
লম্বা thread মানে বড় TTFT। Caching + trimming।

### Cost concerns
History resend খরচ। Prompt caching সবচেয়ে বড় সাশ্রয়।

### What did I learn?
- Conversation memory = short-term (thread) + long-term (session summary)।

### What can I explain to someone else?
> "Doctor এর কাছে আজকের visit এ যা বলছ সেটা তার মনে আছে (thread), আর আগের visit গুলোর সারাংশ ফাইলে (session summary)।"

### GitHub Evidence
- [ ] Cross-session conversation summary demo

### LinkedIn Post
- [ ] —

---

⬅️ Back to [Phase 5 — Agents & Agentic Systems](README.md) · [Root README](../README.md)

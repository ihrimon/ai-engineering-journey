# Phase 4 — RAG & Knowledge Systems: GraphRAG (Learning Log)

> প্রতিটা topic [Learning Log template](../notes/templates/learning-log.md) follow করে লেখা। ভাষা বাংলা, technical term ইংরেজিতে — যাতে docs/interview এ একই শব্দ চিনতে পারো।
>
> ⚠️ "GraphRAG" শব্দটা দুই অর্থে ব্যবহার হয়: (১) Microsoft এর নির্দিষ্ট GraphRAG পদ্ধতি/library (2024), (২) knowledge graph ব্যবহার করে যেকোনো RAG। এখানে দুটোই আলোচনা করা হয়েছে।

## Quick Cheat Sheet

| Concept | এক লাইনে |
|---|---|
| [Knowledge graphs](#knowledge-graphs) | তথ্যকে node (entity) আর edge (relationship) হিসেবে রাখা |
| [Entities](#entities) | গুরুত্বপূর্ণ "জিনিস": মানুষ, company, product, জায়গা, ধারণা |
| [Relationships](#relationships) | Entity গুলোর মাঝের সম্পর্ক: `works_at`, `owns`, `depends_on` |
| [Graph retrieval](#graph-retrieval) | Entity খুঁজে graph traverse করে সম্পর্কিত তথ্য আনা |
| [GraphRAG architecture](#graphrag-architecture) | Document → extract → graph → community summary → local/global search |
| [When GraphRAG is useful](#when-graphrag-is-useful) | সম্পর্ক-ভিত্তিক, multi-hop, "পুরো corpus জুড়ে" প্রশ্নে |

```text
Vector RAG:  "কোন chunk প্রশ্নের মতো দেখতে?"        → similarity
GraphRAG:    "কোন entity কার সাথে কীভাবে যুক্ত?"    → relationship

   (Rimon) ──works_at──► (Acme) ──uses──► (Postgres)
      │                     │
   manages               client_of
      ▼                     ▼
  (Project X) ◄─funds── (Globex)
```

---

<a id="knowledge-graphs"></a>

## 1. Knowledge Graphs

### What is it?
তথ্যকে **graph** হিসেবে রাখা: **node** = entity (প্রতিটার type ও property), **edge** = relationship (type ও property)। Triple আকারে: `(subject, predicate, object)` যেমন `(Acme, uses, Postgres)`। Storage: Neo4j, Memgraph, Amazon Neptune, বা Postgres এ table দিয়েও।

### Why does it exist?
অনেক প্রশ্নের উত্তর text এর মিলে না, **সম্পর্কে**: "কে কাকে report করে", "কোন service কোন DB এর উপর নির্ভর করে"। Chunk-based RAG এ এই সম্পর্ক বিভিন্ন document এ ছড়িয়ে থাকে।

### What problem does it solve?
Explicit, traversable সম্পর্ক: multi-hop প্রশ্ন, aggregation ("Acme এর কয়টা client?"), ব্যাখ্যাযোগ্য reasoning path।

### How does it work?
```text
(:Person {name:"Rimon"})-[:WORKS_AT {since:2024}]->(:Company {name:"Acme"})

Cypher query:
MATCH (p:Person)-[:WORKS_AT]->(c:Company {name:"Acme"}) RETURN p.name
```

### Important concepts
- **Schema/ontology**: কোন entity type আর relation type allowed।
- **Property graph** (Neo4j) বনাম **RDF triple store** (SPARQL)।
- **Graph traversal**: n-hop neighborhood।
- **Graph + vector**: node এ embedding রেখে semantic entry point।

### Trade-offs
শক্তিশালী relationship reasoning, কিন্তু graph বানানো ও maintain করা কঠিন ও দামি।

### Alternatives
Relational DB (join), vector RAG + metadata, document এর ভেতরে cross-reference।

### When should I use it?
Data আসলেই সম্পর্ক-ভিত্তিক: org chart, supply chain, dependency, fraud network, medical (drug-disease-gene)।

### When should I NOT use it?
FAQ, policy, সাধারণ documentation, যেখানে প্রশ্নের উত্তর এক জায়গায় লেখা।

### Hands-on experiment
```bash
docker run -p 7474:7474 -p 7687:7687 -e NEO4J_AUTH=neo4j/password neo4j:5
```
হাতে ১০টা triple দিয়ে একটা ছোট company graph বানাও, তারপর ২-hop Cypher query লিখো।

### Production implementation
Schema আগে design, entity resolution pipeline, graph DB backup, incremental update।

### Failure cases
Schema না থাকায় একই relation ২০ নামে (`works_at`, `employed_by`, `job_at`), তাই query অসম্পূর্ণ।

### Security concerns
Graph traversal দিয়ে permission ছাড়া তথ্য জোড়া লাগানো সম্ভব (A দেখতে পারো, B দেখতে পারো, কিন্তু A-B সম্পর্ক গোপন)। Node/edge-level ACL।

### Performance concerns
গভীর traversal (৪+ hop) explode করতে পারে। Hop limit দাও।

### Cost concerns
Graph DB hosting, আর LLM দিয়ে graph বানানোর বিশাল indexing খরচ।

### What did I learn?
Graph = সম্পর্ককে first-class citizen বানানো।

### What can I explain to someone else?
> "Facebook এর friend network এর মতো: প্রত্যেকে একটা বিন্দু, বন্ধুত্ব একটা রেখা। 'আমার বন্ধুর বন্ধু কে কে?' এটা রেখা ধরে হাঁটলেই পাওয়া যায়।"

### GitHub Evidence
- [ ] Neo4j toy company graph + 2-hop Cypher

### LinkedIn Post
- [ ] —

---

<a id="entities"></a>

## 2. Entities

### What is it?
Graph এর node: text থেকে বের করা নির্দিষ্ট "জিনিস", যেমন মানুষ, প্রতিষ্ঠান, product, জায়গা, তারিখ, ধারণা। প্রতিটার type, নাম, description আর property।

### Why does it exist?
Graph বানাতে আগে জানতে হবে "কী কী জিনিস নিয়ে কথা হচ্ছে"।

### What problem does it solve?
Unstructured text কে structured, linkable unit এ রূপান্তর।

### How does it work?
```text
text: "Rimon ২০২৪ সালে Acme এ যোগ দেন এবং Project X পরিচালনা করেন।"
LLM extraction (structured output):
  entities: [{name:"Rimon", type:"Person"}, {name:"Acme", type:"Company"}, {name:"Project X", type:"Project"}]
```

### Important concepts
- **NER** (Named Entity Recognition): classic model বা LLM।
- **Entity resolution / dedup**: "Acme", "Acme Inc.", "ACME Corp" = একই entity। এটা GraphRAG এর সবচেয়ে কঠিন অংশ।
- **Canonical ID** প্রতিটা entity র।
- **Type schema** দিয়ে extraction সীমিত রাখো।

### Trade-offs
LLM extraction = flexible ও accurate, কিন্তু দামি আর মাঝে মাঝে inconsistent। Classic NER = সস্তা, কিন্তু domain-specific type এ দুর্বল।

### Alternatives
Existing structured data (CRM, HR system) থেকে সরাসরি entity।

### When should I use it?
Graph বানানোর প্রথম ধাপ হিসেবে। Metadata enrichment এও (chunk এ কোন entity আছে)।

### When should I NOT use it?
—(GraphRAG না করলেও entity metadata প্রায়ই উপকারী)

### Hands-on experiment
```ts
const EXTRACT = `Extract entities from the text. Types: Person, Company, Project, Product.
Return JSON: {"entities":[{"name":"...","type":"..."}]}`;
```
১০টা paragraph এ চালিয়ে দেখো একই entity কতভাবে নাম পায়। তারপর একটা simple resolver লিখো (lowercase + suffix বাদ + embedding similarity)।

### Production implementation
Extraction → normalize → resolve (exact + fuzzy + embedding) → canonical ID → upsert।

### Failure cases
"Apple" (company) আর "apple" (ফল) একই node এ merge।

### Security concerns
Entity তে PII (মানুষের নাম, phone)। Graph এ কী রাখবে, policy ঠিক করো।

### Performance concerns
Corpus জুড়ে extraction ধীর, batch/parallel করো।

### Cost concerns
প্রতি chunk এ LLM call, বড় corpus এ উল্লেখযোগ্য খরচ। ছোট model দিয়ে করো।

### What did I learn?
Entity extraction সহজ, entity **resolution** কঠিন।

### What can I explain to someone else?
> "একটা গল্প পড়ে চরিত্রের তালিকা বানানো। কঠিন অংশ হলো বোঝা যে 'রাজা', 'মহারাজ', আর 'বিক্রম' একই মানুষ।"

### GitHub Evidence
- [ ] Entity extraction + resolution demo

### LinkedIn Post
- [ ] —

---

<a id="relationships"></a>

## 3. Relationships

### What is it?
দুইটা entity র মাঝের typed, directed সম্পর্ক (graph এর edge): `(Rimon)-[MANAGES]->(Project X)`। Property থাকতে পারে (since, weight, source chunk)।

### Why does it exist?
Entity আলাদা আলাদা থাকলে শুধু একটা তালিকা। সম্পর্ক যোগ করলে সেটা জ্ঞান হয়ে ওঠে।

### What problem does it solve?
"কে, কার সাথে, কীভাবে" প্রশ্ন আর chain reasoning।

### How does it work?
```text
LLM extraction (entity এর সাথে একসাথে):
relations: [{source:"Rimon", target:"Project X", type:"MANAGES", evidence:"…পরিচালনা করেন"}]
```

### Important concepts
- **Relation schema** (allowed types) না থাকলে chaos।
- **Direction** গুরুত্বপূর্ণ: `A OWNS B` ≠ `B OWNS A`।
- **Evidence/provenance**: কোন chunk থেকে এসেছে, citation ও debug এর জন্য।
- **Temporal**: সম্পর্ক সময়ের সাথে বদলায় (`valid_from`, `valid_to`)।

### Trade-offs
Rich structure, কিন্তু extraction error (ভুল relation, ভুল direction) graph এ স্থায়ী ভুল।

### Alternatives
Co-occurrence (একই chunk এ থাকলে সম্পর্কিত ধরা), যা সহজ কিন্তু অস্পষ্ট।

### When should I use it?
সম্পর্ক যেখানে প্রশ্নের কেন্দ্রে।

### When should I NOT use it?
সম্পর্ক অস্পষ্ট বা খুব বেশি (প্রতিটা জিনিস সবকিছুর সাথে যুক্ত)।

### Hands-on experiment
একই paragraph ৩ বার extract করো (temperature > 0)। কয়টা relation consistent? তারপর schema (allowed types) দিয়ে আবার করো, consistency বাড়ে কিনা দেখো।

### Production implementation
Relation সাথে `source_chunk_id`, `confidence`, `extracted_at`, `model_version`।

### Failure cases
"Acme Globex কে কিনতে চেয়েছিল" থেকে `Acme OWNS Globex` বের হয়েছে, অথচ "চেয়েছিল" মানে ঘটেনি।

### Security concerns
ভুল relation (যেমন "X fraud এ জড়িত") reputation/legal ঝুঁকি। Evidence সহ রাখো, sensitive relation এ human review।

### Performance concerns
Edge সংখ্যা বেশি হলে traversal ভারী।

### Cost concerns
Entity এর সাথে একই call এ extract করো।

### What did I learn?
Relationship এর মান = provenance এর মান। Source ছাড়া relation বিশ্বাস করবে না।

### What can I explain to someone else?
> "পরিবারের family tree তে নামের পাশাপাশি দাগগুলো (বাবা, মা, ভাই) না থাকলে সেটা শুধু নামের তালিকা।"

### GitHub Evidence
- [ ] Schema-guided relation extraction consistency test

### LinkedIn Post
- [ ] —

---

<a id="graph-retrieval"></a>

## 4. Graph Retrieval

### What is it?
প্রশ্নের জন্য graph থেকে relevant subgraph/তথ্য আনা: প্রশ্নের entity খুঁজে বের করা (entry point), তারপর traversal (neighbor, path), তারপর সেই তথ্য text আকারে LLM এর context এ।

### Why does it exist?
Graph থেকে "কাছের chunk" না, "সম্পর্কিত entity ও path" আনতে হয়।

### What problem does it solve?
Multi-hop আর relationship প্রশ্ন, explainable path সহ।

### How does it work?
```text
Q: "Project X এর manager যে company তে কাজ করে, তার DB কী?"
1. Entity linking: "Project X" → node#42  (exact/fuzzy/embedding)
2. Traverse:  (Project X)<-[MANAGES]-(Rimon)-[WORKS_AT]->(Acme)-[USES]->(Postgres)
3. Serialize: "Rimon manages Project X. Rimon works at Acme. Acme uses Postgres."
4. + related source chunks → LLM
```
বা **text-to-Cypher**: LLM সরাসরি graph query লেখে।

### Important concepts
- **Entity linking** প্রশ্ন থেকে node।
- **k-hop neighborhood** বনাম **path finding**।
- **Hybrid**: graph facts + vector chunks একসাথে context এ।
- **Text-to-Cypher** এর ঝুঁকি text-to-SQL এর মতো (read-only)।

### Trade-offs
সুনির্দিষ্ট ও ব্যাখ্যাযোগ্য, কিন্তু entity linking ব্যর্থ হলে কিছুই আসে না, আর graph অসম্পূর্ণ হলে উত্তর ও অসম্পূর্ণ।

### Alternatives
Multi-hop vector retrieval, SQL join।

### When should I use it?
Graph ইতিমধ্যে আছে বা সম্পর্ক-প্রশ্ন বেশি।

### When should I NOT use it?
Graph quality কম হলে vector RAG ই নির্ভরযোগ্য।

### Hands-on experiment
তোমার toy graph এ ৫টা ২-hop প্রশ্ন: (১) vector RAG (original text chunk), (২) graph retrieval (Cypher + serialize)। Accuracy তুলনা করো।

### Production implementation
Read-only graph user, query timeout, hop limit, serialized facts এ source chunk id।

### Failure cases
প্রশ্নে "PX" লেখা, node এর নাম "Project X", তাই linking fail আর ফাঁকা context।

### Security concerns
Text-to-Cypher এ write query (`DELETE`) রোধ করো, read-only credential।

### Performance concerns
Traversal indexed হলে দ্রুত, কিন্তু supernode (হাজার edge) এ ধীর।

### Cost concerns
Query time এ সাধারণত কম। খরচ মূলত indexing এ।

### What did I learn?
Graph retrieval এর দুর্বলতম জায়গা হলো প্রশ্ন থেকে সঠিক entry node খুঁজে পাওয়া।

### What can I explain to someone else?
> "Metro map এ আগে নিজের station খুঁজে বের করো (entity linking), তারপর লাইন ধরে গন্তব্যে যাও (traversal)।"

### GitHub Evidence
- [ ] Vector vs graph retrieval on 2-hop questions

### LinkedIn Post
- [ ] —

---

<a id="graphrag-architecture"></a>

## 5. GraphRAG Architecture

### What is it?
Microsoft Research এর GraphRAG (2024) pipeline: LLM দিয়ে document থেকে entity/relation graph বানানো, graph কে **community** তে ভাগ করা (Leiden algorithm), প্রতিটা community র **summary** আগে থেকে লিখে রাখা, তারপর query তে **local search** বা **global search**।

### Why does it exist?
Vector RAG "পুরো corpus এর মূল theme গুলো কী?" ধরনের **global/sensemaking** প্রশ্নে ব্যর্থ, কারণ কোনো একটা chunk এ সেই উত্তর নেই।

### What problem does it solve?
Dataset-wide summary প্রশ্ন ("এই ৫০০ customer interview এর মূল অভিযোগ কী?") আর entity-centric প্রশ্ন।

### How does it work?
```text
INDEXING (দামি, একবার):
docs → chunks → LLM: entities + relations + claims → graph
     → community detection (hierarchical) → LLM: প্রতিটা community র report/summary

QUERY:
Local search : প্রশ্নের entity → তার neighborhood + related chunks + community summary → answer
Global search: সব community summary → map (প্রতিটা থেকে partial answer) → reduce (final)
DRIFT        : local + global এর মিশ্রণ (পরবর্তী উন্নতি)
```
হালকা variant: LightRAG, nano-graphrag, LazyGraphRAG (indexing খরচ কমাতে)।

### Important concepts
- **Community** = ঘনিষ্ঠভাবে যুক্ত entity এর group।
- **Pre-computed summaries**: query time এ সস্তা, indexing এ দামি।
- **Map-reduce** global answer এর জন্য।
- **Incremental update** কঠিন: নতুন doc এলে community আবার হিসাব।

### Trade-offs
Global প্রশ্নে অসাধারণ, কিন্তু indexing অনেক দামি (প্রতিটা chunk এ একাধিক LLM call), ধীর, আর update কঠিন।

### Alternatives
Vector RAG + map-reduce summarization, RAPTOR (hierarchical summary tree), LazyGraphRAG।

### When should I use it?
Static বা ধীরে বদলানো corpus, যেখানে global/thematic প্রশ্ন মূল use case (research, intelligence analysis)।

### When should I NOT use it?
দ্রুত বদলানো data, সাধারণ factual Q&A, সীমিত budget।

### Hands-on experiment
Microsoft `graphrag` (Python) দিয়ে ছোট একটা corpus (যেমন ২০টা blog post) index করো, local Ollama model দিয়ে। Indexing time ও token মাপো। তারপর একটা global প্রশ্ন vector RAG আর GraphRAG global search এ তুলনা করো।

### Production implementation
Indexing batch job (offline), versioned graph, cost estimate আগে (chunk × call × token), community summary cache।

### Failure cases
১০,০০০ document index করতে অপ্রত্যাশিত বিশাল LLM bill। ছোট sample এ আগে খরচ মাপো।

### Security concerns
Community summary বিভিন্ন document এর তথ্য মেশায়, তাই permission-mixed summary একজন user কে এমন তথ্য দিতে পারে যা তার দেখার কথা না। Multi-tenant এ graph tenant অনুযায়ী আলাদা।

### Performance concerns
Global search এ অনেক community summary map, তাই query ও ধীর হতে পারে।

### Cost concerns
Indexing খরচ vector RAG এর তুলনায় বহুগুণ। এটাই সবচেয়ে বড় সিদ্ধান্তের বিষয়।

### What did I learn?
GraphRAG আগে থেকে "পুরো corpus বুঝে রাখে", তাই global প্রশ্নে জেতে, কিন্তু সেই বোঝাপড়ার দাম indexing এ দিতে হয়।

### What can I explain to someone else?
> "পরীক্ষার আগে পুরো বইয়ের প্রতিটা অধ্যায়ের সারাংশ আর চরিত্রের সম্পর্কের ছক বানিয়ে রাখা। সময় লাগে, কিন্তু 'বইয়ের মূল বক্তব্য কী' প্রশ্নে সেটাই কাজে দেয়।"

### GitHub Evidence
- [ ] Microsoft GraphRAG on small corpus + cost log

### LinkedIn Post
- [ ] "GraphRAG: global প্রশ্নে vector RAG কেন হেরে যায়"

---

<a id="when-graphrag-is-useful"></a>

## 6. When GraphRAG Is Useful

### What is it?
সিদ্ধান্ত নেওয়ার framework: কোন পরিস্থিতিতে GraphRAG এর বাড়তি খরচ ও জটিলতা যুক্তিসঙ্গত, আর কোথায় vector/hybrid RAG যথেষ্ট।

### Why does it exist?
GraphRAG নিয়ে hype আছে। ভুল জায়গায় ব্যবহার করলে অনেক খরচ, অল্প লাভ।

### What problem does it solve?
Architecture decision সঠিকভাবে নেওয়া।

### How does it work?
```text
প্রশ্নের ধরন                                   সেরা পছন্দ
───────────────────────────────────────────   ──────────────────────
"X এর refund policy কী?" (এক জায়গায় উত্তর)     Vector/hybrid RAG
"ERR_4471 কী?"                                 Hybrid (BM25)
"গত মাসে revenue কত?"                          Structured (SQL)
"A আর B কীভাবে যুক্ত?" (relationship)           Graph retrieval
"৩ ধাপের চেইন প্রশ্ন"                           Graph বা multi-hop/agentic
"পুরো dataset এর মূল theme কী?" (global)        GraphRAG global / map-reduce
```

### Important concepts
- **Question taxonomy** আগে বানাও: real user query sample করে label করো।
- **Data shape**: entity-dense ও interconnected? নাকি স্বাধীন document?
- **Hybrid approach**: vector RAG default + graph শুধু relationship প্রশ্নে (router)।
- **Start simple**: vector RAG → eval → যেখানে ব্যর্থ সেখানে graph।

### Trade-offs
Graph যোগ = নতুন ক্ষমতা, কিন্তু ২টা system maintain।

### Alternatives
Multi-hop/agentic RAG, structured DB, RAPTOR।

### When should I use it?
- Relationship-heavy domain (org, supply chain, compliance, fraud, biomedical)।
- Global sensemaking প্রশ্ন।
- Explainable reasoning path দরকার।

### When should I NOT use it?
- Simple support/FAQ bot।
- দ্রুত বদলানো data।
- Eval এ vector RAG ইতিমধ্যে ভালো করছে।

### Hands-on experiment
তোমার target project (Enterprise RAG) এর জন্য ৫০টা সম্ভাব্য user প্রশ্ন লিখে উপরের table অনুযায়ী label করো। কত শতাংশ প্রশ্নে graph আসলেই লাগবে?

### Production implementation
Query router (classifier): `{ type: "lookup" | "relationship" | "global" | "metric" }` → উপযুক্ত retriever।

### Failure cases
Team ৩ মাস GraphRAG বানাল, পরে দেখা গেল ৯০% প্রশ্ন simple lookup, যেখানে hybrid RAG একই ফল দিত।

### Security concerns
দুই system মানে দুই জায়গায় ACL ঠিক রাখতে হবে।

### Performance concerns
Router এর extra ছোট call।

### Cost concerns
Indexing খরচ vs লাভ, ছোট pilot দিয়ে ROI মাপো।

### What did I learn?
Tool বাছাই করো প্রশ্ন দেখে, hype দেখে না।

### What can I explain to someone else?
> "হাতুড়ি পেলেই সব কিছু পেরেক না। আগে দেখো প্রশ্নগুলো কী ধরনের, তারপর tool বাছো।"

### GitHub Evidence
- [ ] Question taxonomy (50 Qs labeled) + router prototype

### LinkedIn Post
- [ ] "GraphRAG দরকার কিনা বুঝবেন কীভাবে"

---

⬅️ Back to [Phase 4 — RAG & Knowledge Systems](README.md) · [Root README](../README.md)

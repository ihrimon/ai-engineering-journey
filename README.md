# AI Engineering Journey

## From Full-Stack Developer to Production-Ready AI Engineer

> **Goal:** Become a strong, production-ready AI Engineer who can design, build, evaluate, secure, optimize, deploy, and explain real-world AI-powered software systems.
>
> **Understand → Design → Build → Evaluate → Secure → Optimize → Deploy → Monitor → Explain**

**Target stack:** TypeScript-first AI Engineer + productive Python AI/ML Engineer + strong Software Engineering.

📘 Full roadmap → [roadmap/AI_ENGINEERING_ROADMAP.md](roadmap/AI_ENGINEERING_ROADMAP.md)

---

## 🗂️ Repository Structure

```text
ai-engineering-journey/
│
├── roadmap/                    # Master roadmap (+ archive of older versions)
│   └── AI_ENGINEERING_ROADMAP.md
│
├── 00-software-engineering/    # Phase 0
├── 01-llm/                     # Phase 1
├── 02-prompting/               # Phase 2
├── 03-context/                 # Phase 3
├── 04-rag/                     # Phase 4
├── 05-agents/                  # Phase 5
├── 06-tools/                   # Phase 6
├── 06-mcp/                     # Phase 6 — MCP
├── 07-inference/               # Phase 7
├── 08-llmops/                  # Phase 8
├── 09-evaluation/              # Phase 9
├── 10-cost-performance/        # Phase 10
├── 11-security/                # Phase 11
├── 12-multimodal/              # Phase 12
├── 13-architecture/            # Phase 13
│
├── python/                     # Python Track A–E (parallel track)
│
├── projects/
│   ├── ai-support-platform/
│   ├── enterprise-rag/
│   ├── ai-crm/
│   ├── voice-agent/
│   ├── workflow-platform/
│   ├── multimodal-intelligence/
│   └── evaluation-platform/
│
└── notes/                      # Learning-log & weekly-review templates, legacy v1 material
```

---

## 📑 Table of Contents

| Phase | Topic | Folder | Status |
|---|---|---|---|
| 0 | [Software Engineering Foundation](#phase-0) | [00-software-engineering](00-software-engineering/README.md) | ✅ Core done |
| 1 | [LLM Fundamentals](#phase-1) | [01-llm](01-llm/README.md) | 🟡 In progress |
| 2 | [Prompt Engineering](#phase-2) | [02-prompting](02-prompting/README.md) | 🟡 In progress |
| 3 | [Context Engineering](#phase-3) | [03-context](03-context/README.md) | ⬜ |
| — | [Python Learning Track](#python) | [python](python/README.md) | ⬜ |
| 4 | [RAG & Knowledge Systems](#phase-4) | [04-rag](04-rag/README.md) | ⬜ |
| 5 | [Agents & Agentic Systems](#phase-5) | [05-agents](05-agents/README.md) | ⬜ |
| 6 | [Tool Use & Integrations](#phase-6) | [06-tools](06-tools/README.md) · [06-mcp](06-mcp/README.md) | ⬜ |
| 7 | [Inference & AI Infrastructure](#phase-7) | [07-inference](07-inference/README.md) | ⬜ |
| 8 | [LLMOps & Observability](#phase-8) | [08-llmops](08-llmops/README.md) | ⬜ |
| 9 | [Evaluation Engineering](#phase-9) | [09-evaluation](09-evaluation/README.md) | ⬜ |
| 10 | [Cost & Performance Engineering](#phase-10) | [10-cost-performance](10-cost-performance/README.md) | ⬜ |
| 11 | [Safety, Security & Guardrails](#phase-11) | [11-security](11-security/README.md) | ⬜ |
| 12 | [Multimodal AI Engineering](#phase-12) | [12-multimodal](12-multimodal/README.md) | ⬜ |
| 13 | [AI Application Architecture](#phase-13) | [13-architecture](13-architecture/README.md) | ⬜ |
| ★ | [Flagship Portfolio Projects](#projects) | [projects](projects/README.md) | ⬜ |

---

<a id="phase-0"></a>

## Phase 0 — Software Engineering Foundation ✅

- [x] Programming & Object-Oriented Foundations
- [x] Backend, APIs & Databases
- [x] Git, GitHub & Professional Workflow
- [x] AI Awareness & Engineering Mindset
- [ ] Systems (OS, networking, TLS, Linux, Docker, CI/CD, cloud)
- [ ] System Design (scalability, CAP, queues, distributed systems, observability)

[📖 Deep dive](00-software-engineering/README.md) · [🧠 Interview Q&A](00-software-engineering/interview-qa.md)

<a id="phase-1"></a>

## Phase 1 — LLM Fundamentals

- [x] **Core Concepts** — tokens, tokenization, context windows, token cost, input vs output tokens, latency, capabilities, limitations → [📝 বাংলা Learning Log](01-llm/core-concepts.md)
- [x] Model Families — GPT, Claude, Gemini, Llama, Mistral, Qwen, DeepSeek → [📝 বাংলা Learning Log](01-llm/model-families.md)
- [x] Sampling — temperature, top-p, top-k, penalties, determinism → [📝 বাংলা Learning Log](01-llm/sampling.md)
- [x] Generation — streaming, structured outputs, function calling, tool use → [📝 বাংলা Learning Log](01-llm/generation.md)
- [x] Multimodal Inputs — vision, images, audio, PDFs, documents, tables, charts → [📝 বাংলা Learning Log](01-llm/multimodal-inputs.md)
- [x] Embeddings — vectors, similarity, cosine, dimensions, model selection → [📝 বাংলা Learning Log](01-llm/embeddings.md)
- [ ] Rerankers — why rerank, cross-encoder, pipeline, trade-offs → [📝 বাংলা Learning Log](01-llm/rerankers.md)
- [ ] Reasoning models
- [ ] 🛠️ Project: **LLM Model Comparison Lab**

[📖 Deep dive](01-llm/README.md)

<a id="phase-2"></a>

## Phase 2 — Prompt Engineering

- [ ] Prompt architecture — system/user/assistant, instruction hierarchy
- [ ] Techniques — zero/few-shot, role prompting, CoT concepts, ReAct, reflection
- [ ] Structure — XML, Markdown, delimiters, output contracts
- [ ] Engineering — templates, versioning, output parsers, prompt testing
- [ ] Security — prompt injection & defense
- [ ] 🛠️ Project: **Prompt Evaluation Laboratory**

[📖 Deep dive](02-prompting/README.md) · [🧠 Interview Q&A](02-prompting/interview-qa.md)

<a id="phase-3"></a>

## Phase 3 — Context Engineering

- [ ] Context architecture & budgeting
- [ ] Optimization — summarization, compression, pruning, lost-in-the-middle
- [ ] Prompt caching
- [ ] Memory — short/long-term, episodic, semantic, procedural
- [ ] 🛠️ Project: **Production AI Memory System**

[📖 Deep dive](03-context/README.md)

<a id="python"></a>

## Python Learning Track

- [ ] Track A — Fundamentals
- [ ] Track B — Engineering (uv, pytest, asyncio, packaging)
- [ ] Track C — AI/Data (NumPy, Pandas, Hugging Face, PyTorch)
- [ ] Track D — AI Backend (FastAPI, Node.js ↔ Python)
- [ ] Track E — Advanced AI (fine-tuning, LoRA/QLoRA, benchmarking)

[📖 Deep dive](python/README.md)

<a id="phase-4"></a>

## Phase 4 — RAG & Knowledge Systems

- [ ] Chunking, vector databases, retrieval (dense/sparse/hybrid, BM25, RRF)
- [ ] Query optimization — rewriting, HyDE, multi-hop, agentic retrieval
- [ ] Reranking, GraphRAG, document parsing, multimodal RAG
- [ ] 🛠️ Flagship: **Enterprise RAG & Document Intelligence Platform**

[📖 Deep dive](04-rag/README.md)

<a id="phase-5"></a>

## Phase 5 — Agents & Agentic Systems

- [ ] Patterns — ReAct, Plan-and-Execute, Reflexion, ReWOO
- [ ] Architecture, frameworks (LangGraph, Mastra, Inngest), memory
- [ ] Human-in-the-loop, multi-agent, reliability
- [ ] Long-running agents (Temporal, durable execution), computer use
- [ ] 🛠️ Project: **AI Business Operations Agent**

[📖 Deep dive](05-agents/README.md)

<a id="phase-6"></a>

## Phase 6 — Tool Use & Integrations

- [ ] Tool design & tool calling
- [ ] MCP — servers, clients, resources, prompts, tools
- [ ] Code execution sandboxes
- [ ] API integrations
- [ ] 🛠️ Project: **AI Integration Agent**

[📖 Tools](06-tools/README.md) · [📖 MCP](06-mcp/README.md)

<a id="phase-7"></a>

## Phase 7 — Inference & AI Infrastructure

- [ ] API vs self-hosted, inference engines (vLLM, TGI, SGLang, Ollama, llama.cpp)
- [ ] Quantization, KV cache, speculative decoding, continuous batching
- [ ] Metrics (TTFT, TPS), GPU economics, edge AI, providers
- [ ] 🛠️ Project: **LLM Inference Benchmark Lab**

[📖 Deep dive](07-inference/README.md)

<a id="phase-8"></a>

## Phase 8 — LLMOps & Observability

- [ ] Observability platforms, tracing, metrics
- [ ] Versioning, experimentation, debugging
- [ ] Drift detection, feedback loops
- [ ] 🛠️ Project: **AI Observability Dashboard**

[📖 Deep dive](08-llmops/README.md)

<a id="phase-9"></a>

## Phase 9 — Evaluation Engineering

- [ ] Offline evaluation, LLM-as-judge
- [ ] RAG & agent evaluation, synthetic data
- [ ] Frameworks (Promptfoo, DeepEval, Inspect, Braintrust, OpenAI Evals)
- [ ] Online evaluation, red teaming
- [ ] 🛠️ Project: **AI Evaluation Platform**

[📖 Deep dive](09-evaluation/README.md)

<a id="phase-10"></a>

## Phase 10 — Cost & Performance Engineering

- [ ] Model cascading & routing
- [ ] Caching, batch processing
- [ ] Distillation, prompt compression, token budgets
- [ ] Model gateways (LiteLLM, Portkey, OpenRouter)
- [ ] 🛠️ Project: **AI Cost Optimization Gateway**

[📖 Deep dive](10-cost-performance/README.md)

<a id="phase-11"></a>

## Phase 11 — Safety, Security & Guardrails

- [ ] Prompt, data, and tool security
- [ ] Moderation & guardrails
- [ ] Application security — PII, rate limiting, tenant isolation, audit logs
- [ ] 🛠️ Project: **Secure AI Agent Platform**

[📖 Deep dive](11-security/README.md)

<a id="phase-12"></a>

## Phase 12 — Multimodal AI Engineering

- [ ] Vision, OCR, document AI
- [ ] Voice AI — STT, TTS, realtime
- [ ] Image & video generation, generation infrastructure
- [ ] 🛠️ Flagship: **Multimodal Document & Media Intelligence Platform**

[📖 Deep dive](12-multimodal/README.md)

<a id="phase-13"></a>

## Phase 13 — AI Application Architecture

- [ ] Communication (REST, SSE, WebSockets, streaming)
- [ ] Background processing & reliability
- [ ] Multi-tenant AI, provider architecture
- [ ] AI API patterns — start, stream, cancel, retry, resume
- [ ] 🛠️ Project: **Production AI Platform Architecture**

[📖 Deep dive](13-architecture/README.md)

<a id="projects"></a>

## ★ Flagship Portfolio Projects

- [ ] [AI Support Platform](projects/ai-support-platform/README.md)
- [ ] [Enterprise RAG & Document Intelligence](projects/enterprise-rag/README.md)
- [ ] [AI CRM & Sales Automation](projects/ai-crm/README.md)
- [ ] [AI Voice Booking Agent](projects/voice-agent/README.md)
- [ ] [AI Workflow & API Automation Platform](projects/workflow-platform/README.md)
- [ ] [Multimodal Document & Media Intelligence](projects/multimodal-intelligence/README.md)
- [ ] [AI Engineering Evaluation Platform](projects/evaluation-platform/README.md)

---

## 📝 How I Learn

Every major topic follows the **[Learning Log template](notes/templates/learning-log.md)** and is reviewed weekly with the **[Weekly Engineering Review](notes/templates/weekly-review.md)**.

```text
Learn → Experiment → Build → Test → Evaluate → Document → Explain → Publish
```

### Definition of Done

A topic is complete only when I can: explain it · implement it · debug it · compare alternatives · identify trade-offs · test it · evaluate it · explain failure cases · apply it to a real project · document it · show GitHub evidence.

---

> **Don't become someone who only knows AI tools. Become the engineer who understands why those tools exist, when to use them, how they fail, how to evaluate them, and how to turn them into reliable production systems.**

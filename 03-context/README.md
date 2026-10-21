# Phase 3 — Context Engineering

How to decide what the model sees, how much, and in what order: context architecture, budgeting, optimization (summarization, compression, pruning), prompt caching, and memory systems.

## Context Architecture

📝 Learning Log (বাংলা): [context-architecture.md](context-architecture.md)

- [x] [System context](context-architecture.md#system-context)
- [x] [Conversation history](context-architecture.md#conversation-history)
- [x] [Tool results](context-architecture.md#tool-results)
- [x] [Retrieved documents](context-architecture.md#retrieved-documents)
- [x] [User context](context-architecture.md#user-context)
- [x] [Memory](context-architecture.md#memory)
- [x] [Context budgeting](context-architecture.md#context-budgeting)

## Context Optimization

📝 Learning Log (বাংলা): [context-optimization.md](context-optimization.md)

- [x] [Token budgeting](context-optimization.md#token-budgeting)
- [x] [Summarization](context-optimization.md#summarization)
- [x] [Rolling memory](context-optimization.md#rolling-memory)
- [x] [Compression](context-optimization.md#compression)
- [x] [Pruning](context-optimization.md#pruning)
- [x] [Tool result truncation](context-optimization.md#tool-result-truncation)
- [x] [Lost-in-the-middle problem](context-optimization.md#lost-in-the-middle)
- [x] [Context ordering](context-optimization.md#context-ordering)

## Prompt Caching

📝 Learning Log (বাংলা): [prompt-caching.md](prompt-caching.md)

- [x] [OpenAI prompt caching](prompt-caching.md#openai-prompt-caching)
- [x] [Anthropic prompt caching](prompt-caching.md#anthropic-prompt-caching)
- [x] [Cacheable context](prompt-caching.md#cacheable-context)
- [x] [Cache invalidation](prompt-caching.md#cache-invalidation)
- [x] [Cost optimization](prompt-caching.md#cost-optimization)

## Memory

📝 Learning Log (বাংলা): [memory.md](memory.md)

- [x] [Short-term memory](memory.md#short-term-memory)
- [x] [Long-term memory](memory.md#long-term-memory)
- [x] [Episodic memory](memory.md#episodic-memory)
- [x] [Semantic memory](memory.md#semantic-memory)
- [x] [Procedural memory](memory.md#procedural-memory)
- [x] [Hierarchical memory](memory.md#hierarchical-memory)

## Practical Project — Production AI Memory System

Conversation memory, summary memory, semantic memory, user preferences, context compression, token budget management.

- [ ] Conversation memory (thread-scoped history)
- [ ] Summary memory (rolling summary)
- [ ] Semantic memory (user facts + preferences)
- [ ] Context compression
- [ ] Token budget management
- [ ] Prompt caching + hit-rate report

⬅️ Back to [Root README](../README.md) · [Full Roadmap](../roadmap/AI_ENGINEERING_ROADMAP.md)

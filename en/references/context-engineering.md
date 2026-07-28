# Context engineering: long context, cache, agents, economy

Read this file for: production prompts, agents, work with long documents, cost/latency optimization. Prompt engineering is a subset of context engineering — managing EVERYTHING in the window (system prompt, tools, history, retrieved documents, memory).

## Three empirical facts that drive the design

1. **"It fits in the window" ≠ "the model uses it".** Chroma's "Context Rot" (18 frontier models): quality degrades as input length grows for ALL models, even on simple tasks; it's a gradient, not a cliff. NoLiMa (ICML 2025): at 32K tokens, 10 of 12 models fall below 50% of their short-context level. A focused ~300-token prompt consistently beats a full ~113K one on the same tasks.
2. **The U-shaped attention curve ("Lost in the Middle", Liu et al., TACL 2024).** The beginning and end of the context are read well, the middle sags. Put the most important material at the edges; don't bury the gold document in the middle of the pile.
3. **The KV cache is a design factor, not just an optimization.** Contexts with identical prefixes reuse the cache: a cache read costs ≈10% of base input price at Anthropic/Gemini 2.5+ (50% at OpenAI, automatic). Manus calls cache hit-rate the single most important metric for a production agent.

## Window layout for a production prompt

```
[1] SYSTEM PROMPT (stable) — role, rules, output format
[2] TOOL DEFINITIONS (stable) — 3–5 key tools
[3] MEMORY / RETRIEVED DOCS (semi-stable)
--- cache breakpoint here ---
[4] MESSAGE HISTORY (grows)
[5] SCRATCHPAD / current state (volatile)
[6] USER QUERY (volatile)
```

Rules:
- **Stable first, volatile last.** Even a one-token difference in the prefix (an exact timestamp!) invalidates the cache. Put the date, if needed, after the breakpoint, or coarsen it to the day.
- **Append-only + deterministic serialization** (JSON key order!) — otherwise the cache breaks silently.
- **Don't add or remove tools dynamically mid-session** — it breaks the cache and confuses the model with references to non-existent tools.
- Anthropic: explicit `cache_control` breakpoints (up to 4), caching the prefix tools → system → messages; write +25%, read 10%. OpenAI: automatic, ≥1024 tokens. Gemini: implicit by default plus explicit.

## The system prompt: the "right altitude"

The Goldilocks zone: don't hardcode brittle if-else logic, but don't give vague guidance either. Structure it in sections (XML or headings — one format). Aim for the minimal set of information that fully describes the behavior; a rough target is ~2000 tokens for the system part. Every instruction should fix a measured failure — start with a minimal prompt and add blocks based on eval results (the OpenAI and Anthropic principle).

## Agent tools

- A minimal, non-overlapping set; "every tool justifies its existence". Anthropic's heuristic: if a human engineer can't say unambiguously which tool to use, an agent certainly can't.
- Threshold: >10–15 tools → dynamic loading (Anthropic's Tool Search: −85% of tokens on definitions).

## Long runs: compaction and external memory

- **Compaction** — summarizing the history as you approach the limit: keep architectural decisions, unresolved bugs, implementation details; drop redundant tool output. The risk: overly aggressive compression loses subtle but critical context — maximize the summary's recall first, then improve precision. Anthropic offers server-side compaction and context editing (clearing old tool results — "the safest form of compaction").
- **External memory / note-taking:** for tasks longer than one session, use note files (NOTES.md, a todo list, a memory tool). Promote learned decisions into memory, not raw tool output. In the agent's prompt: "Always review the memory directory before starting work."
- **Subagents:** justified for breadth-first tasks with independent parallel branches (research) — each works in its own window and returns a compressed 1–2K-token summary; the cost is ~15× the tokens. For tasks with dense dependencies (most coding), a single agent with compaction is more reliable.

## Token economy — the techniques with the best effect/effort ratio

1. Prompt caching (the structure above) — up to 90% off input.
2. Batch API — −50% on non-urgent work (OpenAI, Anthropic).
3. Model routing: cheap models (Haiku/Flash/mini) for simple tasks.
4. Reasoning-effort control: on GPT-5, "minimal" vs. "high" differs by tens of times in tokens.
5. Reducing output: ask for brevity, `max_tokens`, structured output, a diff instead of the whole file.
6. RAG instead of dumping everything into a long window.
7. Cheap data formats (YAML/CSV instead of JSON) — ~28–33% on uniform tabular data.
8. Writing the prompt in English — 20–40% fewer tokens for non-Latin languages.

## RAG specifics for the prompt author

Hallucinations in RAG are most often a retrieval failure, not a prompt failure (up to 67% of hallucinations in conversational RAG are "extractive": the model faithfully reproduces incorrectly retrieved content). Before "fixing the prompt", diagnose:

- Low context recall → fix retrieval (embeddings/chunking/search); the prompt won't help.
- Retrieval is good but faithfulness is low → now it's a prompt/model problem: strengthen grounding ("first quote in `<quotes>`, then answer in `<answer>`"; "if the answer isn't in the documents, say so explicitly").
- If the corpus is <200K tokens (~500 pages), RAG may be unnecessary: put everything in a cacheable prompt.

Metrics and diagnosis — in `evals.md`.

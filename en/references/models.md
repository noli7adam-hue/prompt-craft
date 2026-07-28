# Model cheat sheet: platform specifics (2025–2026)

Read this file when the target model isn't Claude, or when you need precise platform specifics. Specific version names are volatile — check against the vendor's current documentation, but the categories (reasoning vs. standard) and the principles are stable.

## Reasoning vs. standard models — summary table

| Aspect | Standard (GPT-class, Haiku/Flash/mini) | Reasoning (GPT-5.x, Claude thinking, Gemini 3 thinking) |
|---|---|---|
| Metaphor (OpenAI) | "junior colleague" — precise instructions | "senior colleague" — goal + done criterion |
| CoT "think step by step" | helps on multi-step tasks | redundant/harmful (+2.9…3.1% for +20–80% latency; Gemini Flash −3.3%) |
| Few-shot | 2–5 examples are useful | often hurts; start zero-shot |
| Context in RAG | can be generous | only the filtered, relevant part |
| Depth control | through the prompt | through an API knob |
| Self-checking | ask for it explicitly | partly built in; give a done criterion |

Where CoT is still needed: non-reasoning models on math/logic/debugging; some open-source families (Gemma3, Qwen-Math) are still benchmarked with few-shot CoT.

---

## Claude (Anthropic)

**Strengths:** XML structuring, literal instruction-following, long documents, agentic code.

- **XML is the native format.** `<instructions>`, `<context>`, `<examples>`, multi-document `<document>` → `<source>` + `<document_content>`. The model is trained to respect arbitrary hierarchical tags.
- **Literalness.** Ask for "above and beyond" behavior explicitly; Claude will reproduce any accidental pattern from the examples. Motivating context (the "why") improves generalization.
- **Long context:** data (20k+ tokens) at the top, the query at the end — up to +30% quality. For long documents: quote the relevant parts first, then answer.
- **Reasoning:** adaptive thinking + the `effort` parameter (low → max); `budget_tokens` is deprecated. Use extended thinking only when it noticeably improves quality: "When in doubt, answer directly." Against overthinking: "Choose an approach and stick with it. Don't revisit decisions without new contradicting information."
- **With thinking disabled**, the model is sensitive to the word "think" — use "consider", "evaluate", "reason through".
- **Prefill is unsupported on 4.6+** (error 400) — set the format with an explicit instruction or through tool use.
- **Temperature/top_p can't be changed on the newest models.**
- **Dosing action** — two official blocks: `<default_to_action>` (act by default, for agents) vs. `<do_not_act_before_instructions>` (don't change anything without an explicit request).
- **Parallel tool calls:** the official block:
```
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between the calls,
make all of the independent calls in parallel. If a call depends on the result of a
previous one, call them sequentially. Never use placeholders or guess parameters.
</use_parallel_tool_calls>
```
The inverse problem: the new Sonnets parallelize aggressively — sometimes you need to ask for sequencing explicitly.
- **Anti-over-engineering (agentic code):**
```
Avoid over-engineering. Do only what is explicitly requested or clearly necessary.
- Don't add features, refactoring, or "improvements" beyond the request.
- Don't add docstrings/comments to code you didn't change.
- Don't add error handling for scenarios that can't happen.
```
- **Prose instead of bullets:** the special block `<avoid_excessive_markdown_and_bullet_points>`; plus the principle "match the prompt's style to the desired response style" — remove markdown from the prompt and you'll get less markdown in the answer.
- **Softening insistence:** "CRITICAL: YOU MUST" from the 3.x era causes overtriggering — replace it with "Use this tool when…".

## GPT-5.x / OpenAI

**Strengths:** agentic pipelines, Structured Outputs, parameter-level control.

- **Two model classes:** GPT ("workhorses") — precise instructions; reasoning — goal + constraints + done criterion. Officially, for the newest ones: "State the expected outcome and success criteria. Reduce or remove detailed step-by-step process guidance. Remove output schema definitions from the prompt — use Structured Outputs instead." When migrating old prompts — "start from a clean slate".
- **Knobs:** `reasoning_effort` (minimal/low/medium/high/xhigh; there's a `none` mode with no reasoning tokens) + `verbosity` (low/medium/high). The rule: **one change at a time** — model first, then effort, then the prompt, with evals after each step.
- **Agency:** the blocks `<persistence>` (keep going until the request is fully resolved; document assumptions instead of asking again), `<tool_preambles>` (restate the goal → plan → progress notes → summary), and a tool-call limit ("no more than N calls"). The Responses API preserves reasoning between tool calls.
```
<persistence>
You are an agent — keep going until the request is fully resolved. Don't stop at uncertainty.
Don't ask the user to confirm assumptions — document them, act, and correct course.
</persistence>
```
- **Output contract:**
```
<output_contract>
- Return exactly the requested sections in the requested order.
- If a format is specified (JSON, Markdown, SQL, XML), output only that.
</output_contract>
```
- **RAG for reasoning models:** only filtered, relevant context, not "everything available".
- The system message on the reasoning series is converted into a developer message — in practice it works like the former system prompt.

## Gemini 3 (Google)

**Strengths:** long context, multimodality, grounding through Google Search / Vertex.

- **The order is the reverse of Claude's:** with long context, data goes ON TOP, instructions and the question AT THE VERY END, with the anchor "Based on the entire document above…" and an explicit "use ONLY the provided document as the source of truth".
- **temperature=1.0 — don't change it** (risk of looping on the newer models).
- **Brevity is mandatory:** "please" and polite filler are perceived as noise. Gemini 3 is terse by default — ask for talkativeness explicitly.
- **Don't mix XML and markdown** — pick one format.
- **Broad negative constraints are forbidden:** "do not infer/guess" breaks the basic logic and arithmetic — replace it with the positive "use the provided context for the calculations". On complex requests the model "drops" negative constraints — put critical rules at the end, next to the question.
- **The persona outweighs the instructions:** Gemini takes an assigned persona so seriously that it may ignore the rules for its sake — don't create contradictions.
- **Reasoning:** `thinking_level` (minimal/low/medium/high); for speed — LOW + "think silently".
- **Multimodality:** label the inputs explicitly ("Image 1", "Video 2") and reference the modality in the instruction.
- **Few-shot:** a strictly uniform format across all examples, otherwise the answer drifts; don't overload with examples (overfitting to the format).
- **Grounding:** temperature 1.0 for grounding mode; anti-hallucination through two-step verification: "First check whether you have access to X. If not, answer 'No Info' and STOP."

## Perplexity (Sonar)

**The key point: it's a search engine, not a chat.** Officially: "only the user message is used to drive that search. The system prompt is not visible to search". Which means "query engineering" matters more than prompt engineering.

- **Separation of roles:** `instructions` (the system prompt) sets the role, tone, formatting, and grounding rules; `input` (the user message) seeds the search query — specificity here directly improves retrieval. A polished system prompt won't rescue a vague query.
- **Filters go in parameters, not prose:** "search only Wikipedia" in the prompt doesn't work. Use `search_domain_filter` (up to 20 domains, `-` to exclude), `search_recency_filter` (hour/day/week/month), `search_context_size` (low/medium/high).
- **Don't ask for URLs in prose** — the model cites by index; the authoritative list of sources is in the `citations`/`search_results` fields.
- **Don't few-shot with content** — the search will latch onto the example's topic. Few-shot for structure is fine; for a guaranteed shape use `response_format`.
- **Permission to say "not found":** a mandatory block against hallucination:
```
Answer only from the search results. If the results don't contain the answer,
say so explicitly rather than guessing. If the results are adjacent but not matching
(a different year, a parent company, a similar product), state the mismatch before answering.
```
- **A good query** = specifics + a time horizon + a source type: "What guidance has the FDA issued on AI in medical devices over the past year, and which device categories does it cover?" beats "Tell me about FDA rules on AI". "Find peer-reviewed studies" beats "find studies".
- **Models:** sonar (fast), sonar-pro (synthesis), sonar-reasoning-pro (CoT + search), sonar-deep-research (autonomous, hundreds of sources).
- Setting `instructions` with a preset REPLACES the preset's system prompt rather than extending it.

## Choosing a platform for the task

- Sources, fresh data, citations → Perplexity (or Gemini grounding)
- Long documents, multimodality → Gemini
- Structural control, agentic code, reliable instruction-following → Claude
- Agentic pipelines, Structured Outputs → OpenAI GPT-5.x

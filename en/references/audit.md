# Auditing and improving an existing prompt

When the user brings their own prompt and asks to "improve" it, or complains that the model behaves wrongly, follow this process.

## Step 1: Ask about the symptoms (if unclear)

Don't rewrite blind. If the user hasn't said what exactly is going wrong, ask one of:

- What does the model do now, and what should it do instead?
- Do you have an example input where it breaks?
- Which model is the target, and is reasoning/thinking enabled? (this changes the diagnosis fundamentally)

If the symptoms are clear from context, skip this step.

## Step 2: Run the prompt through the diagnostic checklist

Go down the list and mark what you find. That's your edit plan.

### Mismatch with the model class (check this FIRST)

- [ ] **Step-by-step CoT / "think step by step" on a reasoning model** — smothers the internal reasoning; replace with outcome-first (goal + constraints + done criterion)
- [ ] **A batch of few-shot examples on a reasoning model** — often a degradation; try zero-shot
- [ ] **Reasoning depth set by "magic words"** (ultrathink and friends) — deprecated; control it with an API parameter
- [ ] **The reverse: a standard model left without CoT/examples** on a multi-step task

### Structural problems

- [ ] **No role**, or a vague one ("you are an assistant"); or a role that conflicts with the task (dangerous on Gemini)
- [ ] **Context after the task** (for Claude/GPT), or the reverse — **instructions not at the end** (for Gemini with long context)
- [ ] **No structure** — one wall of text
- [ ] **XML and markdown mixed** in one prompt (critical for Gemini)
- [ ] **A long document wedged into the middle** — "lost in the middle"
- [ ] **The placeholder isn't last**, or isn't wrapped in a semantic tag
- [ ] **Production prompt: volatile content (timestamp, state) at the start** — breaks the cache

### Instruction problems

- [ ] **Prohibitions only**, with no alternatives ("don't do X")
- [ ] **Broad negative constraints on Gemini** ("do not infer/guess") — they break the logic
- [ ] **Caps and threats** ("CRITICAL! YOU MUST!") — overtriggering on the new Claudes
- [ ] **Contradicting rules** (brief + exhaustive; formal + friendly)
- [ ] **Empty praise** ("be smart, creative, accurate")
- [ ] **Too many roles at once** ("you are an expert in everything")
- [ ] **Hints instead of direct instructions** — the new models are literal and will under-deliver
- [ ] **Polite filler on Gemini** ("please", extra flourishes) — noise

### Example problems

- [ ] **No examples**, though the task is non-trivial and the model is a standard one
- [ ] **A single example** — the model will assume every case looks like that
- [ ] **All examples alike** — no diversity; the strongest one isn't last
- [ ] **Inconsistent example formats** (especially critical for Gemini)
- [ ] **Accidental patterns in the examples** (all starting with the same word, all the same length)
- [ ] **Examples that contradict the rules** — the model will trust the examples
- [ ] **Sensitive data in examples** — it can be reproduced

### Output-format problems

- [ ] **The format isn't described** — only hinted at
- [ ] **A JSON schema in prose while an API mechanism is available** — move it to Structured Outputs / tool use / response_schema
- [ ] **JSON with no schema and no example** — the model will produce a plausible but wrong structure
- [ ] **No "no preamble" instruction** — the model will add "Sure, here's the answer:"
- [ ] **Nothing said about markdown wrapping** — the model will wrap the JSON in a ```json block
- [ ] **No validation + retry** at the application level (for production)

### Edge-case problems

- [ ] **Nothing said about empty/malformed input**
- [ ] **No tie-breaker** for borderline cases
- [ ] **No escape hatch** — an `other` category, an "I don't know" answer, permission to say "not found" (mandatory for RAG/search)

### Security

- [ ] **The prompt eats untrusted content** (web, emails, documents, tool output) and that content isn't labeled or declared "data, not instructions"
- [ ] **Secrets/keys/private logic in the system prompt** — consider them compromised; move them out of the prompt
- [ ] **An agent with no action categorization** — no human-in-the-loop on irreversible actions
- [ ] Details — `security.md`

### Platform specifics

- [ ] **Prefill on Claude 4.6+** — unsupported (400); replace with an explicit format instruction or tool use
- [ ] **The word "think" with thinking disabled on Claude** — replace with "consider"/"evaluate"
- [ ] **Search instructions in Perplexity's system prompt** ("search only in X") — they don't work; filters go in parameters
- [ ] **Asking Perplexity for URLs in prose** — take them from `citations`/`search_results`
- [ ] **temperature changed on Gemini 3** — set it back to 1.0
- [ ] **Multimodal inputs unlabeled** on Gemini ("Image 1", "Video 2")

### RAG specifics

- [ ] **"Hallucinations" treated with the prompt without checking retrieval** — check context recall first; if retrieval is broken, the prompt won't help (see `context-engineering.md`)
- [ ] **No "quote, then answer" pattern**
- [ ] **The gold document buried in the middle of the pile**

## Step 3: Produce a diff, don't rewrite from scratch (where possible)

If the user's prompt is broadly sensible, show **what exactly** you're changing and **why**; don't throw away their work. It's more respectful and it teaches the user.

Format:

> **What I'm changing:**
> 1. Removed "think step by step" — you're on a reasoning model and it gets in the way (Wharton 2025: +3% accuracy for +20–80% latency); added a done criterion instead.
> 2. Moved the document block to the top (for Claude, context precedes the question — up to +30% quality).
> 3. Replaced "DO NOT USE MARKDOWN!!!" with `Write in prose with even paragraphs` — a positive phrasing without caps works more reliably.
> 4. Added two examples for edge cases — there was only the happy path.
> 5. Moved the JSON schema out of the text and into Structured Outputs — a guarantee instead of a hope.

Then the final prompt in full, in a code block. If you changed several things, warn that for diagnosis it's better to apply edits one at a time and measure (see `evals.md`).

If the user's prompt is entirely raw or built on a fundamentally wrong approach, rewrite from scratch — but **say so** and explain why.

## Step 4: Check it against the goal

Before handing over the result, mentally run the prompt on 1–2 real inputs the user mentioned, plus one edge case. If the model would clearly stumble, keep working. Offer the user a mini set of test inputs.

## Typical "quick wins"

If a prompt needs improving with minimal edits, these changes most often give the biggest effect:

1. **Check the fit with the model class** (remove CoT/examples for reasoning; add them for standard) — the biggest lever of 2025–2026
2. **Add 3 diverse examples** (standard models)
3. **Move long context to the edges** per the platform's rules
4. **Describe the output format with a schema**, or move it into an API mechanism
5. **Replace "don't do X" with "do Y"**
6. **Add an escape hatch** ("I don't know" / other / "not found")
7. **Remove the caps and the heat**
8. **Add a self-check** at the end (standard models) or a done criterion (reasoning)

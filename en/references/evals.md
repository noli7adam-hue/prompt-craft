# Evaluating prompts: evals, iteration, regressions

Read this file when: the prompt is going to production; the user complains that "it works inconsistently"; you need to compare prompt variants; you need to survive a model version change.

## The basic loop (Anthropic)

Define success criteria → build evals → write the prompt → add context → iterate. Skipping evals is the main cause of "it worked yesterday" and of inconsistency within a team. A prompt with no way to test it isn't finished.

## The minimum for any prompt (even a one-off)

A run over 3–5 real inputs, necessarily including:
- the typical "happy path";
- an edge case (empty/malformed/ambiguous input);
- a borderline case between categories/modes;
- for prompts over untrusted content — an input with an embedded "command" (a check of injection hygiene).

If even one input breaks the prompt, fix it before handing it over.

## Iteration rules

- **One change at a time** (OpenAI's rule): model first, then parameters (effort/thinking), then the prompt text; measure after each step. Otherwise you won't know what worked.
- **Add a block to the prompt only when it fixes a measured failure.** Start with the minimal prompt; every line must justify its existence.
- **Positive AND negative examples** in the eval set: negatives define the boundaries and prevent overtriggering.
- **Pin the model snapshot** in production. Behavior drifts between versions; on an upgrade, a regression run over the full set is mandatory before rollout. Cache doesn't carry over between models.
- **Every production failure → a new regression test.**

## Metrics

Move from vibes to measurement:
- **Deterministic checks** — for the precise things: exact match, JSON validity against a schema, presence of required fields, tool names. Cheap; run them on 100% of cases.
- **LLM-as-judge** — for the subjective: relevance, faithfulness/groundedness, completeness, clarity. Always validate the judge against human labels (disagreement → fix the judge's prompt); include reasoning before the score in the judge's prompt (a judge template is in `patterns.md`). The judge evaluates the result, not "the right path" (agentic runs are non-deterministic).
- **Human labeling** — 2–5% for calibration and edge cases.

The cascade: deterministic on everything → LLM judge on a sample and on 100% of CI regressions → humans on calibration.

## Diagnosis by symptom

| Symptom | Likely cause | What to do |
|---|---|---|
| Output format drifts | Schema only in prose | Structured Outputs / tool use; one format example; validation + retry |
| The model answers off-document (RAG) | Check retrieval first | Low recall → fix retrieval; recall high → strengthen grounding in the prompt |
| Answers vary run to run | Ambiguous rules, no tie-breakers | Explicit conflict-resolution rules; for critical work — self-consistency (several runs + majority) |
| The model overdoes it / panics | Caps, threats, "CRITICAL" | Soften to an even tone |
| The model under-delivers | Relying on it to infer | Ask for completeness explicitly (4.x/5.x literalness) |
| A reasoning model got worse after "improvements" | CoT/examples were added | Remove them; go back to outcome-first |
| Costly/slow without a quality gain | Surplus thinking | Lower effort/thinking_level; if the wrong-answer rate is unchanged — turn it off |

## Building a golden dataset

- Create it early, for the specific use case. Each example: input → expected output (or acceptance criteria) → a "positive/negative/edge" tag.
- You can generate candidates with an LLM, but human verification is mandatory.
- 10–20 examples are enough to start; expand as failures surface.

## Phrasing asserts for a prompt

A good criterion is binary and checkable: "the answer contains exactly one word from the category list", "the JSON is valid against the schema and all dates are ISO 8601", "the answer contains no URLs", "empty input returns other". A bad one: "the answer is high quality".

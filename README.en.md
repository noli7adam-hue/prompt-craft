# Prompt Craft

*[Русская версия](README.md)*

> **Note:** the English skill in `en/` corresponds to version 1. The Russian version at the repository root is at version 2, which adds a third model class (frontier-agentic: Opus 5 and newer), five new practices, a cross-vendor migration protocol, and a source registry — see [CHANGELOG.md](CHANGELOG.md). The English translation of v2 is in progress; everything below still holds.

A Claude Code skill: writing, improving, and auditing prompts for modern LLMs — Claude 4.x/5, GPT-5.x, Gemini 3, Perplexity Sonar and their peers.

The premise: there is no longer such a thing as a universally "good prompt". The practices that worked in 2023 are useless or harmful on the reasoning models of 2025–2026. The skill first determines the model class and the platform, and only then assembles the prompt — from a canonical structure and through a quality checklist.

## Why this exists

A plain "write me a prompt" gets you the average of everything: a role, "think step by step", a couple of examples, "be helpful". On a reasoning model that prompt performs worse than a short statement of the goal.

The skill rests on a few principles:

- **The model class matters more than the vendor.** A reasoning model is a "senior colleague": you give it a goal and a done criterion. A standard model is a "junior colleague": it needs precise steps and examples.
- **CoT stopped being free.** The Wharton GenAI Lab (arXiv 2506.07142) measured it: on reasoning models "think step by step" buys +2.9…+3.1% accuracy for 20–80% more latency, and on Gemini Flash 2.5 it costs −3.3%.
- **Model attention is U-shaped.** The beginning and end of the context are read better than the middle ("Lost in the Middle", TACL 2024) — the placement of data and question accounts for this.
- **"It fits in the window" ≠ "the model uses it".** Focused context consistently beats a dump (Chroma Context Rot).
- **A prompt isn't finished until it's tested.** The minimum is a run over 3–5 real inputs, including edge cases, changing one element at a time.

## What's inside

`SKILL.md` is the core: the reasoning/standard fork, the canonical prompt structure, twelve modern practices, a quality checklist, a list of anti-patterns, and a full worked example.

References load on demand, so they don't clog the context up front:

| File | When it's needed |
|---|---|
| `references/template.md` | Ready skeletons: canonical, minimal, reasoning, Gemini, cache-friendly |
| `references/patterns.md` | A standard task: classification, extraction, generation, agent, RAG, judge, structured output |
| `references/models.md` | The target model isn't Claude: GPT-5.x, Gemini 3, Perplexity Sonar |
| `references/audit.md` | Reviewing and improving an existing prompt |
| `references/security.md` | The prompt processes untrusted content; protection against injection |
| `references/context-engineering.md` | Production, agents, long context, caching, compaction |
| `references/evals.md` | Evals, comparing prompts, LLM-as-judge |

## What it doesn't do

- It's a method, not a guarantee. The skill removes the standard mistakes; it doesn't replace evals on your own data.
- The practices are time-bound. The recommendations reflect vendor documentation from 2025–2026; models drift, and a version change calls for a regression run.
- It won't fully protect you from indirect prompt injection — no complete solution exists (the position of OWASP, Microsoft, and Google). It reduces the risk rather than eliminating it.
- It won't write your success criteria for you. If it's unclear what counts as a good answer, no prompt will save you.

## Installation

The English version lives in `en/` and is installed by copying that directory:

```bash
git clone https://github.com/kravcovtech/prompt-craft.git /tmp/prompt-craft \
  && cp -r /tmp/prompt-craft/en ~/.claude/skills/prompt-craft
```

Windows (PowerShell):

```powershell
git clone https://github.com/kravcovtech/prompt-craft.git $env:TEMP\prompt-craft
Copy-Item -Recurse "$env:TEMP\prompt-craft\en" "$env:USERPROFILE\.claude\skills\prompt-craft"
```

For a single project, copy `en/` to `.claude/skills/prompt-craft` inside the repository instead.

Restart Claude Code. The skill picks itself up whenever prompts come up; you can also invoke it explicitly with `/prompt-craft`.

## Usage

Just state the task in plain words:

- "Write a prompt to classify tickets into 4 categories, target model Claude Sonnet"
- "Here's my system prompt, it ignores some of the instructions — fix it"
- "Make a prompt for RAG answers over our docs, resistant to injection"
- "A prompt for GPT-5 with reasoning: extract entities from a contract"

You get back a finished prompt in a single code block, a short explanation of the decisions taken, an "API parameters" block where relevant (reasoning_effort, thinking_level, response_schema), and a suggestion for how to test the result.

The skill also works without Claude Code: `SKILL.md` can be pasted into any agent as a system instruction.

## Repository layout

```
.
├── README.md                             # Russian
├── README.en.md                          # English
├── LICENSE
├── SKILL.md                              # Russian skill (repository root)
├── references/                           # Russian references
│   ├── template.md
│   ├── patterns.md
│   ├── models.md
│   ├── audit.md
│   ├── security.md
│   ├── context-engineering.md
│   └── evals.md
└── en/                                   # self-contained English skill
    ├── SKILL.md
    └── references/
```

The Russian version sits at the root so a plain `git clone` into `~/.claude/skills/` installs it directly; `en/` is a self-contained copy of the same skill.

## Roadmap

- [ ] A set of eval cases for testing the skill itself
- [ ] Patterns for multi-agent systems and orchestration
- [ ] Before/after examples on real prompts

## Contributing

Found a practice that has gone stale, or a recommendation that performs worse on your model? Open an issue with a concrete example or send a pull request. Measurements are especially valuable: "on model X, technique Y produced effect Z". Keep the Russian and English versions in sync.

## License

[MIT](LICENSE) — free to use, including commercially. Attribution is appreciated but not required.

---

Maintained by [@kravcovtech](https://github.com/kravcovtech).

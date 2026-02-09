---
name: best-of-n
description: This skill should be used when the user asks to "best of n", "best-of-n query", "run best of n", "bon query", "bon", "/best-of-n", "sample multiple responses", "sample each model", or mentions wanting to query each AI model multiple times and pick the best response before synthesising across models.
---

# Best-of-N query

Query each AI model N times with temperature variation, pick the best response per model, then synthesise across models. This produces higher-quality input for the final synthesis at the cost of N× more API calls.

## When this skill is invoked

**IMPORTANT**: When triggered, follow the execution steps below. Do NOT just describe what the skill does.

### Execution steps

#### Step 1: Get or draft the prompt

**A) Cold start** (no prior discussion): Use the provided prompt, or ask "What question would you like to send to multiple models?"

**B) Mid-conversation** (substantive prior discussion): Draft a comprehensive prompt that captures full context (2-4 paragraphs minimum), includes relevant file contents or code, states the core question clearly, and specifies what format/depth of response is useful. Save to `/tmp/bon-prompt-draft.md`, open it, and show the user for approval.

#### Step 2: Model selection

First, fetch available presets and models (these are defined in ask-many-models config and may change):

```bash
cd /Users/ph/.claude/skills/best-of-n && yarn query presets 2>/dev/null
```

Present the presets as a numbered menu, plus a "Pick models" option. Add "(Recommended)" to the quick preset. Wait for user input.

Deep research and browser models are automatically excluded—they don't benefit from temperature sampling.

If the user selects **Pick models**, fetch the eligible model list:

```bash
cd /Users/ph/.claude/skills/best-of-n && yarn query models 2>/dev/null
```

Show the output and ask the user to enter model IDs (comma-separated).

#### Step 3: Ask N and temperature

Ask the user (or use defaults):
- **N** (samples per model): default 4
- **Temperature**: default 0.8

Show a cost warning: "This will make {models × N} API calls."

#### Step 4: Run the query

If the user chose a preset, use `--preset <name>`. If they picked individual models, use `--models <comma-separated-ids>`.

Generate a slug from the prompt (lowercase, non-alphanumeric to hyphens, max 50 chars).

For brainstorming prompts, add `--brainstorm` to merge all unique ideas across samples instead of picking one best response per model.

```bash
cd /Users/ph/.claude/skills/best-of-n && yarn query \
  --preset <preset-name> \
  --num-samples <n> \
  --temperature <temp> \
  --live-file "/Users/ph/.claude/skills/best-of-n/multi-model-responses/$(date +%Y-%m-%d-%H%M)-bon-<slug>.md" \
  --synthesise \
  --output-format both \
  "<prompt>"
```

Or with explicit models:

```bash
cd /Users/ph/.claude/skills/best-of-n && yarn query \
  --models "<model-id-1>,<model-id-2>,..." \
  --num-samples <n> \
  --temperature <temp> \
  --live-file "/Users/ph/.claude/skills/best-of-n/multi-model-responses/$(date +%Y-%m-%d-%H%M)-bon-<slug>.md" \
  --synthesise \
  --output-format both \
  "<prompt>"
```

#### Step 5: Open results

Open the live file (HTML version if available, otherwise markdown):
```bash
open "<live-file-path with .md replaced by .html>"
```

---

## How it works

```
Prompt → [Model₁ × N] → Per-model comparison → Best₁ ─┐
         [Model₂ × N] → Per-model comparison → Best₂ ─┤→ Cross-model synthesis → Final
         [Model₃ × N] → Per-model comparison → Best₃ ─┘
```

1. All model × N queries run in parallel with 100ms stagger per same-provider call
2. Per-model comparison uses Gemini 3 Flash (fallback: Claude 4.5 Sonnet) to pick the best sample
3. Cross-model synthesis uses Claude Opus 4.6 with extended thinking

## Direct script invocation

```bash
cd /Users/ph/.claude/skills/best-of-n

# Basic usage
yarn query "What are the pros and cons of TypeScript?"

# With options
yarn query -n 4 -T 0.8 -p quick "Your question"
yarn query -n 2 -m gpt-5.2,gemini-3-flash "Your question"

# List eligible models and presets
yarn query models
yarn query presets
```

### CLI options

| Option | Default | Description |
|--------|---------|-------------|
| `-n, --num-samples <n>` | 4 | Samples per model |
| `-T, --temperature <temp>` | 0.8 | Temperature for all runs |
| `-m, --models <list>` | — | Comma-separated model IDs |
| `-p, --preset <name>` | quick | Preset: quick, comprehensive |
| `-t, --timeout <seconds>` | 180 | Timeout per call |
| `-l, --live-file <path>` | — | Live markdown file |
| `--output-format <format>` | markdown | markdown, html, both |
| `-s, --synthesise` | true | Run cross-model synthesis |
| `-B, --brainstorm` | false | Merge all unique ideas instead of picking one best |
| `-o, --output <dir>` | auto | Output directory |

## Temperature guidance

- **0.8** (default): Meaningful variation while staying coherent
- **0.3–0.5**: Less variation, useful for factual queries
- **1.0+**: More variation, can become erratic on some models
- Reasoning models (Anthropic thinking, OpenAI reasoning) have inherent stochasticity; temperature may be silently ignored by the SDK

## Output structure

```
multi-model-responses/2026-02-08-1430-bon-slug/
├── responses.json
├── synthesis.md
└── per-model/
    ├── gpt-5.2-thinking/
    │   ├── sample-0.md … sample-3.md
    │   ├── comparison.md
    │   └── best-response.md
    └── claude-4.5-opus-thinking/
        └── …
```

## Configuration

Uses the same API keys as ask-many-models (`.env` is symlinked). Model definitions are read from `/Users/ph/.claude/skills/ask-many-models/config.json`.


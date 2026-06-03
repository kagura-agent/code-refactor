---
name: code-refactor
description: "Multi-model refactoring analysis — 3 models analyze code, produce proposals as GitHub issues on the target repo. Refactor ≠ rewrite."
---

# Code Refactor

Trigger: any message implying "refactor this" + a repo/scope reference.

## Analysts

- 🏗️ Arch — `default-llm-sg/gpt-5.5` — Architecture: module boundaries, dependency direction, layering
- 🔬 Lens — `default-llm-sg/claude-opus-4.7` — Code quality: readability, patterns, duplication
- ⚡ Flux — `default-llm-sg/gemini-3.1-pro-preview` — Efficiency: dead code, complexity, simplification

## How It Works

1. 3 models independently analyze the target code
2. Proposals are consolidated (consensus = high confidence)
3. Each proposal becomes a `[refactor]` issue on the target repo
4. The repo's own agent decides whether to accept and implement

**We propose, they dispose.** This service is advisory — we never modify target code directly.

## Execution

**Always use FlowForge. Never manually spawn analysts.**

```bash
flowforge run code-refactor --input '{"owner":"<owner>","repo":"<repo>","scope":"<path-or-full>","direction":"<optional>"}'
```

## Direction Support

Refactoring can be purely structural ("clean this up") or directional ("make it more like X"):

- `refactor kagura-agent/cove src/components/` — general cleanup
- `refactor kagura-agent/cove "architecture should be more like Discord"` — directional
- `refactor kagura-agent/abti full` — whole repo analysis

## Core Rule

**Refactor ≠ Rewrite.** Tests must pass before and after. If a proposal would change behavior, it's not a refactor.

## Analysis Standards

- `prompts/<repo>.prompt.md` — project-specific
- `prompts/default.prompt.md` — fallback

## Key Files

- `workflow.yaml` — FlowForge workflow (source of truth)
- `prompts/` — analysis standard prompts
- `runs/` — run records
- `stats.md` — per-analyst assessment

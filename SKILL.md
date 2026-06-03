---
name: code-refactor
description: "Multi-model refactoring analysis — spawn 3 analysts (GPT-5.5, Claude Opus 4.7, Gemini 3.1 Pro), consolidate proposals, create issues on target repo."
---

# Code Refactor

Trigger: any message implying "refactor this repo/module" + a repo reference.

## Analysts

- 🏗️ Arch — `default-llm-sg/gpt-5.5`
- 🔬 Lens — `default-llm-sg/claude-opus-4.7`
- ⚡ Flux — `default-llm-sg/gemini-3.1-pro-preview`

All 3 receive the same prompt — no assigned focus areas. Model differences produce natural divergence; consensus = high-confidence signal.

## Execution

**Always use FlowForge. Never manually spawn analysts.**

```bash
flowforge run code-refactor --input '{"repo":"<owner>/<repo>","scope":"<path>","direction":"<optional>"}'
```

The workflow handles everything: context loading, analyst spawning, consolidation, issue creation, and reflection.

## Input Formats

```
refactor kagura-agent/cove
refactor kagura-agent/cove src/server/
refactor owner/repo --direction "adopt event-driven architecture"
refactor owner/repo path/ --direction "decouple X from Y"
```

## Output

- GitHub issues on the **target repo** (each independent refactoring point = 1 issue)
- Channel summary with issue links

## Cross-channel Callers

```
sessions_send(sessionKey="agent:kagura:discord:channel:1511591264863125586", message="refactor kagura-agent/cove src/api/")
```

## Key Files

- `workflow.yaml` — FlowForge workflow (source of truth)
- `prompts/` — analysis standard prompts
- `runs/` — run records

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

**必须用 FlowForge。不接受手动替代。**

```bash
# 开始 workflow
flowforge run code-refactor
# 每步执行完后推进：
flowforge advance --result '<step output>' -w code-refactor
# 查看当前进度：
flowforge status -w code-refactor
```

FlowForge 逐步推进，不能跳步。workflow 保证 reflection 和 tracking 不被遗漏。

**手动 spawn 只在 FlowForge 完全不可用时才允许。** 如果手动执行，必须逐条对照 workflow.yaml 的每个节点，确认全部完成。

## Analyst Output

Analysts write to `analyses/<repo>-<date>-<name>.md` (e.g. `analyses/cove-2026-06-05-arch.md`).
Parent reads from files, not session history. **Prevents truncation + creates persistent record.**

> Learned from code-review: session history gets truncated. File output is the only reliable path.

## Analysis Standards

- `prompts/<repo>.prompt.md` — project-specific (check first)
- `prompts/default.prompt.md` — fallback

## Re-analysis Protocol

For follow-up analyses of the same repo:
1. Include previous run's consolidated findings
2. Check each previous proposal — was it implemented?
3. **Escalation rule**: Unaddressed proposals from last round that are still relevant → escalate priority
4. Fresh analysis of any new/changed code

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

## Issue Follow-up

After creating issues, track whether they get implemented:
- `tracking.json` — issue status tracking
- Check on next analysis of the same repo

## Key Files

- `workflow.yaml` — FlowForge workflow (source of truth)
- `prompts/` — analysis standard prompts (default + per-repo)
- `analyses/` — analyst output files (persistent, git-tracked)
- `runs/` — run records + reflection
- `stats.md` — per-analyst capability assessment
- `tracking.json` — issue follow-up tracking

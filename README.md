# Code Refactor

Multi-model refactoring analysis service. 3 AI models independently analyze code and produce structural improvement proposals as GitHub issues.

## How It Works

1. **Parse** — Identify target repo, scope (file/module/full), and optional direction
2. **Analyze** — 3 models with different perspectives examine the code in parallel
3. **Consolidate** — Merge proposals, highlight consensus findings
4. **Issue** — Create `[refactor]` labeled issues on the target repo
5. **Reflect** — Record run, evolve prompts

## Core Principle

**Refactor ≠ Rewrite.** Tests must pass before and after. We propose structural improvements, not behavior changes.

## Analysts

| Analyst | Model | Perspective |
|---------|-------|-------------|
| 🏗️ Arch | GPT-5.5 | Architecture, module boundaries, dependencies |
| 🔬 Lens | Claude Opus 4.7 | Code quality, readability, patterns |
| ⚡ Flux | Gemini 3.1 Pro | Efficiency, dead code, simplification |

## Usage

This is a consultancy, not a contractor. We create issues — the target repo's agent decides what to implement.

## Direction Support

Refactoring can have a direction: "make it more like Discord", "extract shared components", etc. This guides the analysts' focus beyond general cleanup.

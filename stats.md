# Code Refactor — Analyst Stats

_Last updated: 2026-06-05 (Asia/Shanghai)_

## Per-Analyst Performance

| Analyst | Model | Runs | Reliability | Avg Proposals |
|---------|-------|------|-------------|---------------|
| 🏗️ Arch | gpt-5.5 | 1 | 1/1 (100%) | 6 |
| 🔬 Lens | claude-opus-4.7 | 1 | 1/1 (100%) | 7 |
| ⚡ Flux | gemini-3.1-pro-preview | 1 | 1/1 (100%) | 7 |

## Dimension Strengths

### 🏗️ Arch (GPT-5.5)
| Dimension | Strength | Evidence |
|-----------|----------|----------|
| Auth/Identity Logic | ⭐⭐⭐ | OAuth re-derivation bug (unique find), invite atomicity (unique find) |
| Implementation Awareness | ⭐⭐ | Practical approach directions, considers migration paths |

**Superpower:** Finds logic bugs in auth flows that others miss.
**Weakness:** Slowest runtime (3m26s vs ~1m30s). Didn't flag TTL cleanup.

### 🔬 Lens (Claude Opus 4.7)
| Dimension | Strength | Evidence |
|-----------|----------|----------|
| FK/Integrity Analysis | ⭐⭐⭐ | Built complete FK constraint table with current vs desired state |
| Design Reasoning | ⭐⭐⭐ | sender_name as intentional denormalization matching Discord webhooks |
| Thoroughness | ⭐⭐⭐ | Most detailed proposals, proper trade-off analysis |

**Superpower:** Systematic constraint analysis + design insight.
**Weakness:** Output truncated in session history (need file output).

### ⚡ Flux (Gemini 3.1 Pro)
| Dimension | Strength | Evidence |
|-----------|----------|----------|
| Future-proofing | ⭐⭐ | CAST(AS INTEGER) 64-bit overflow risk (unique find) |
| Priority Calibration | ⭐⭐⭐ | Most accurately separated P0/P1/P2 |
| Clean Output | ⭐⭐ | Well-structured, concise proposals |

**Superpower:** Catches edge cases others consider too theoretical.
**Weakness:** None observed yet (only 1 run).

## Cross-Analyst Consensus Patterns

From run #1 (cove schema):
- **3/3 consensus on 4 items** — token security, indexes, FK integrity, snowflake pagination
- **2/3 consensus on 2 items** — guild_members index, TTL cleanup
- **Single-source items: 3** — all from Arch (2) or Flux (1), all well-justified

**Signal:** When all 3 agree, it's reliably the most important issue. Single-source finds tend to be either logic bugs (Arch) or future-proofing (Flux).

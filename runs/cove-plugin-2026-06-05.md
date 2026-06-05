# Cove Plugin Layer Refactor — 2026-06-05

## Request
- **Repo**: kagura-agent/cove
- **Scope**: `packages/plugin/src/` (OpenClaw channel plugin)
- **Direction**: Gateway event coverage, reconnect robustness, Discord API compatibility, consistency with server issues

## Analysts
| Analyst | Status | Proposals |
|---------|--------|-----------|
| 🏗️ Arch (GPT-5.5) | ❌ Failed (lost execution context, 6m38s) | 0 |
| 🔬 Lens (Claude Opus 4.7) | ✅ | 14 (most thorough ever) |
| ⚡ Flux (Gemini 3.1 Pro) | ✅ | 5 |

## Consensus (2/2)
1. Gateway RESUME + seq tracking → #235 (P0)
2. REST retry + 429 handling → #236 (P0)
3. Multi-guild support → #237 (P0)
4. Reconnect state recovery → #238 (P1, bundled with small fixes)
5. Opcode 4 lockout → #239 (P2)

## Lens Unique Finds (single-source, quality)
- `cleanupAndSend` missing `isCurrent()` — race condition (bundled into #238)
- Dynamic import on hot path — perf (bundled into #238)
- IDENTIFY payload shape — future-proof (bundled into #238)
- Gateway URL discovery via REST — correctness (bundled into #238)
- outbound vs streaming conflict — P2
- DM vs channel routing — P2
- restClients cache lifecycle — P2

## Arch Failure Analysis
GPT-5.5 timed out at 6m38s with "lost active execution context". This is the 2nd time Arch has been slowest; first time it completely failed. The Copilot API ~60s stream idle timeout may have killed it during a long code-reading phase.

**Action**: Consider reducing Arch's scope or switching to a faster model for plugin-sized codebases.

## Issues Created
#235-#239 (5 total, 3 P0 + 1 P1 + 1 P2)

## Process Notes
- 2/3 analysts is still enough for high-quality output
- Lens's 14 proposals (17KB file) is the most thorough single-analyst output in any run
- File output mechanism continues to work perfectly
- Bundling small P1 fixes into one issue (#238) reduces noise

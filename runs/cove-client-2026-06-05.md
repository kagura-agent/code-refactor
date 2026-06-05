# Cove Client Layer Refactor — 2026-06-05

## Request
- **Repo**: kagura-agent/cove
- **Scope**: `packages/client/src/stores/` + `packages/client/src/lib/` (Zustand stores, Gateway dispatcher, API client)
- **Direction**: Discord client cache model 对齐 — store 结构、事件处理、缓存层级、乐观更新、重连恢复

## Analysts
| Analyst | Runtime | Proposals |
|---------|---------|-----------|
| 🏗️ Arch (GPT-5.5) | ~3m | 6 + cross-cutting notes |
| 🔬 Lens (Claude Opus 4.7) | ~1.5m | 7 + gap table |
| ⚡ Flux (Gemini 3.1 Pro) | ~1.5m | 5 |

## Consensus Matrix — ALL 3/3 consensus
Every single proposal had full consensus this round. The client layer's gaps are that obvious.

1. **GuildStore + guild-scoped channels** → #228 (P0)
2. **Reconnect state recovery** → #229 (P0)
3. **READY as canonical cache snapshot** → #230 (P1)
4. **MemberStore + guild-scoped presence** → #232 (P1)
5. **Optimistic send with nonce** → #233 (P1)
6. **Cascade invalidation + full event surface + zombie detection** → #234 (P0)

## Analyst Observations
- **Lens** 最突出：gap table 格式极好（Discord vs Cove 对比），reconnect blindness 描述最精准，提了 zombie detection
- **Arch** 最完整：cross-cutting notes 发现了 api.resetGuildId() 未在 logout 调用、setMessages 竞态、initPresences 语义
- **Flux** 最简洁：5 个提案全中，代码示例最实用

## Process
- 文件输出机制第二轮验证成功 ✅
- Arch 依然最慢但产出最多 cross-cutting insights
- 全票共识率 100% — client 层的债务非常明显

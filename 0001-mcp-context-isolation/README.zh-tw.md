# RFC-0001: MCP Context 隔離

- **狀態**: 草案
- **作者**: Lucas Wang (Claude World)
- **建立日期**: 2026-01-12
- **更新日期**: 2026-01-12
- **相關 Issues**: [#7336](https://github.com/anthropics/claude-code/issues/7336), [#11364](https://github.com/anthropics/claude-code/issues/11364), [#7172](https://github.com/anthropics/claude-code/issues/7172)

## 摘要

透過 frontmatter 聲明，將 MCP 伺服器指派給特定的 agents 和 skills，並將它們載入到**隔離的 forked context** 而非主 session context。這讓主 context 永遠保持乾淨，同時在需要時仍能使用完整的 MCP 功能。

## 動機

### 問題：Context 預算的壓力

啟用多個 MCP 伺服器時，工具定義在任何工作開始前就消耗大量 context：

| MCP 伺服器 | 大約 Token 消耗 | 來源 |
|-----------|----------------|------|
| GitHub (27 tools) | ~18,000 | [Issue #11364](https://github.com/anthropics/claude-code/issues/11364) |
| AWS MCP servers | ~18,300 | [Issue #7172](https://github.com/anthropics/claude-code/issues/7172) |
| Cloudflare | ~15,000+ | 社群回報 |
| Supabase | ~12,000+ | 社群回報 |
| Sentry | ~14,000 | 社群回報 |
| Playwright (21 tools) | ~13,647 | [Scott Spence](https://scottspence.com/posts/optimising-mcp-server-context-usage-in-claude-code) |
| 平均每個 tool | ~550-850 | Issue #11364 |

**實測影響**：啟用 7 個 MCP 伺服器時，光是工具定義就消耗 **67,300 tokens（200k context 的 33.7%）**。

### 現代工作中樞的困境

現代知識工作者需要管理眾多平台：

| 類別 | 平台 |
|------|------|
| 程式碼託管 | GitHub, GitLab, Bitbucket |
| 專案管理 | Jira, Linear, Asana, Notion |
| 通訊協作 | Slack, Discord, Teams |
| CI/CD | Vercel, Netlify, Cloudflare |
| 資料庫 | Supabase, Postgres, MongoDB |
| 監控 | Sentry, Datadog |

**兩難：**
- **選項 A（全部安裝）**：Session 開始時消耗 50,000+ tokens = 50% context 沒了
- **選項 B（按專案分開）**：失去 Claude Code 作為統一指揮中心的價值

這兩個選項都不能讓 Claude Code 發揮其作為**通用工作協調器**的潛力。

## 詳細設計

### 關鍵區別：Context 隔離，而非延遲載入

> **這個提案與傳統的延遲載入方案有本質上的不同。**

| 方案 | 主 Context | 載入時機 | 複雜度 |
|------|-----------|----------|--------|
| **傳統 Lazy Loading** | 需要時被 MCP 佔用 | Runtime 動態 | 高（狀態管理）|
| **我們的提案：Context 隔離** | **永遠保持乾淨** | Fork 時一次載入 | 低（用現有 `context: fork`）|

```
傳統 Lazy Loading:
Main Context ──[需要 MCP]──> 載入 MCP ──> Main Context（被佔用）

我們的提案（Context 隔離）:
Main Context（保持乾淨）
    └── Fork Agent Context ──> 載入 MCPs ──> 隔離的 Context
                                              └── 結束後釋放
```

### 提案架構

#### 雙向配置

**1. MCP 端：延遲載入標記（settings.json）**

```json
{
  "mcpServers": {
    "memory": {
      "command": "...",
      "lazy": false    // 總是載入（預設，向後相容）
    },
    "github": {
      "command": "...",
      "lazy": true     // 請求前不載入
    },
    "supabase": {
      "command": "...",
      "lazy": true     // 請求前不載入
    }
  }
}
```

**向後相容**：省略 `lazy` 或設定 `lazy: false` 維持現行行為。

**2. Agent/Skill 端：Frontmatter 聲明**

```yaml
# agents/database-specialist.md
---
name: database-specialist
description: 資料庫操作專家
tools: [Read, Bash, Grep]
mcp:
  required: [supabase]    # 必須有
  optional: [redis]       # 有最好
context: fork
---
```

#### 載入邏輯

| MCP `lazy` 設定 | Agent/Skill 聲明 | 結果 |
|----------------|-----------------|------|
| `false`（或省略）| - | ✅ Session 開始時載入（現行行為）|
| `true` | 未聲明 | ❌ 不載入 |
| `true` | `mcp: [xxx]` | ✅ Agent/Skill 執行時載入 |

#### 結果：乾淨的主 Context

```
Main Session（精簡）
    │
    ├── 只有基礎 MCP：filesystem, memory
    │   （~1,300 tokens 而非 67,000+）
    │
    ├── Task: database-specialist（forked）
    │   └── 載入：supabase, redis（隔離）
    │
    └── Skill: /deploy（forked）
        └── 載入：cloudflare, github（隔離）
```

## 好處

1. **Context 效率**：主 context 永遠保持乾淨
2. **細粒度權限**：每個 agent/skill 有自己的 MCP 範圍
3. **漸進式安全**：分層存取控制而非全有或全無
4. **可擴展性**：啟用 20+ MCP 伺服器而不會造成 context 壓力
5. **低實作成本**：重用現有的 `context: fork` 基礎設施

## 參考資料

- [完整文章（claude-world.com）](https://claude-world.com/zh-tw/articles/mcp-lazy-loading)
- [Issue #7336: Lazy Loading Feature Request](https://github.com/anthropics/claude-code/issues/7336)
- [Issue #11364: Lazy-load MCP Tool Definitions](https://github.com/anthropics/claude-code/issues/11364)

---

*本 RFC 由 [Claude World](https://claude-world.com) 發布，一個專注於 Claude Code 最佳實踐的開發者社群。*

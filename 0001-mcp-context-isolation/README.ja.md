# RFC-0001: MCP コンテキスト隔離

- **ステータス**: ドラフト
- **著者**: Lucas Wang (Claude World)
- **作成日**: 2026-01-12
- **更新日**: 2026-01-12
- **関連 Issue**: [#7336](https://github.com/anthropics/claude-code/issues/7336), [#11364](https://github.com/anthropics/claude-code/issues/11364), [#7172](https://github.com/anthropics/claude-code/issues/7172)

## 概要

MCP サーバーを frontmatter 宣言を通じて特定の Agent と Skill に割り当て、メインセッションコンテキストではなく**隔離されたフォークコンテキスト**にロードします。これにより、必要なときに完全な MCP 機能を有効にしながら、メインコンテキストを永続的にクリーンに保ちます。

## 動機

### 問題：コンテキスト予算への圧迫

複数の MCP サーバーが有効な場合、作業を開始する前にツール定義が大量のコンテキストを消費します：

| MCP サーバー | おおよそのトークンコスト | ソース |
|------------|----------------------|--------|
| GitHub (27 tools) | ~18,000 | [Issue #11364](https://github.com/anthropics/claude-code/issues/11364) |
| AWS MCP servers | ~18,300 | [Issue #7172](https://github.com/anthropics/claude-code/issues/7172) |
| Cloudflare | ~15,000+ | コミュニティ報告 |
| Sentry | ~14,000 | コミュニティ報告 |
| Playwright (21 tools) | ~13,647 | [Scott Spence](https://scottspence.com/posts/optimising-mcp-server-context-usage-in-claude-code) |
| Supabase | ~12,000+ | コミュニティ報告 |
| ツールあたり平均 | ~550-850 | Issue #11364 |

**実測影響**：7 つの MCP サーバーがアクティブな場合、ツール定義だけで **67,300 トークン（200k コンテキストの 33.7%）** を消費します。最小限の 3 サーバー構成でも 42,600 トークン（21.3%）を消費します。

### 現代のワークハブ問題

現代のナレッジワーカーは多数のプラットフォームを管理しています：

| カテゴリー | プラットフォーム |
|-----------|-----------------|
| コードホスティング | GitHub, GitLab, Bitbucket |
| プロジェクト管理 | Jira, Linear, Asana, Notion |
| コミュニケーション | Slack, Discord, Teams |
| CI/CD | Vercel, Netlify, Cloudflare |
| モニタリング | Sentry, Datadog |

**ジレンマ：**
- **オプション A（すべてインストール）**：セッション開始時に 50,000+ トークン消費 = コンテキストの 50% が消失
- **オプション B（プロジェクト別に分離）**：統一コマンドセンターとしての Claude Code の価値を損なう

Claude Code が**ユニバーサルワークオーケストレーター**としての可能性を発揮するには、どちらのオプションも受け入れられません。

## 詳細設計

### 重要な違い：コンテキスト隔離であり、遅延ロードではない

> **この提案は従来の遅延ロードアプローチとは根本的に異なります。**

| アプローチ | メインコンテキスト | ロード時間 | 複雑さ |
|-----------|------------------|-----------|--------|
| **従来の遅延ロード** | MCP 必要時に埋まる | ランタイム動的 | 高（状態管理）|
| **私たちの提案：コンテキスト隔離** | **常にクリーン** | フォーク作成時 | 低（`context: fork` を再利用）|

```
従来の遅延ロード:
Main Context ──[MCP が必要]──> MCP ロード ──> Main Context（占有される）

私たちの提案（コンテキスト隔離）:
Main Context（クリーンなまま）
    └── Fork Agent Context ──> MCPs ロード ──> 隔離された Context
                                               └── 完了後に解放
```

### 提案アーキテクチャ

#### 双方向設定

**1. MCP 側：遅延ロードフラグ（settings.json）**

```json
{
  "mcpServers": {
    "memory": {
      "command": "...",
      "lazy": false    // 常にロード（デフォルト、後方互換）
    },
    "github": {
      "command": "...",
      "lazy": true     // リクエストまでロードしない
    },
    "postgres": {
      "command": "...",
      "lazy": true     // リクエストまでロードしない
    }
  }
}
```

**後方互換性**：`lazy` を省略するか `lazy: false` を設定すると、現在の動作が維持されます。

**2. Agent/Skill 側：Frontmatter 宣言**

```yaml
# agents/database-specialist.md
---
name: database-specialist
description: データベース操作エキスパート
tools: [Read, Bash, Grep]
mcp:
  required: [postgres]    # 必須
  optional: [redis]       # あれば良い
context: fork
---
```

```yaml
# skills/deploy/SKILL.md
---
description: 本番環境にデプロイ
mcp: [vercel, github]
context: fork
---
```

#### ロードロジック

| MCP `lazy` 設定 | Agent/Skill 宣言 | 結果 |
|----------------|-----------------|------|
| `false`（または省略）| - | ✅ セッション開始時にロード（現在の動作）|
| `true` | 宣言なし | ❌ ロードしない |
| `true` | `mcp: [xxx]` | ✅ Agent/Skill 実行時にロード |

#### 結果：クリーンなメインコンテキスト

```
メインセッション（スリム）
    │
    ├── 基本 MCP のみ：filesystem, memory
    │   （67,000+ ではなく ~1,300 トークン）
    │
    ├── Task: database-specialist（フォーク）
    │   └── ロード：postgres, redis（隔離）
    │
    └── Skill: /deploy（フォーク）
        └── ロード：vercel, github（隔離）
```

### 段階的権限レイヤー

```
レイヤー 0：メインコンテキスト（最小限）
   └── filesystem（読み取り専用）, memory

レイヤー 1：開発 Agent
   └── code-reviewer: + git（読み取り）
   └── debugger: + bash（サンドボックス）

レイヤー 2：専門 Skill
   └── /deploy: + vercel, github（プッシュ）
   └── /db-migrate: + postgres（書き込み）

レイヤー 3：管理操作
   └── /production-access: すべて（確認付き）
```

## メリット

1. **コンテキスト効率**：メインコンテキストが永続的にクリーン（一時的ではなく）
2. **細粒度権限**：各 Agent/Skill が独自の MCP スコープを持つ
3. **段階的セキュリティ**：全か無かではなく、階層的アクセス制御
4. **スケーラビリティ**：コンテキスト圧迫なしで 20 以上の MCP サーバーを有効化
5. **低い実装コスト**：既存の `context: fork` インフラを再利用

## 検討した代替案

### 1. 従来の遅延ロード（ランタイム動的）

必要時にメインコンテキストに MCP をロード。

**却下理由：**
- メインコンテキストが依然として汚染される
- アンロードのための複雑な状態管理
- 根本的な問題を解決しない

### 2. 手動サーバー管理

ユーザーがセッションごとに MCP を手動で有効/無効化。

**却下理由：**
- ワークフローの中断
- ツールの必要性を事前に予測する必要がある
- セッションの再起動が必要

### 3. キーワードベースのトリガー

特定のキーワードが言及されたときに MCP を自動ロード。

**却下理由：**
- トリガーが不安定
- 依然としてメインコンテキストにロードされる
- 設定が複雑

## 課題

| 課題 | 可能な解決策 |
|-----|-------------|
| MCP 起動遅延 | ウォームプール、最初の言及時に事前接続 |
| フォーク終了後の状態 | ステートレス設計、セッションレベルキャッシュ |
| ツール発見 | 遅延マニフェスト（ツールは宣言済みだが未ロード）|
| 資格情報スコープ | 制限付きの環境継承 |
| MCP が大量データを返す | ストリーミング、チャンキング、フォークごとのサイズ制限 |

## 先行事例

- **Claude Code `context: fork`**（2.1.0）：すでに Skill のコンテキスト隔離を提供
- **Issue #7336**：遅延ロード機能リクエスト（43 upvotes）
- **[machjesusmoto/claude-lazy-loading](https://github.com/machjesusmoto/claude-lazy-loading)**：概念実証の実装
- **React Suspense**：依存関係を宣言してオンデマンドでロードする類似パターン

## 実装ノート

この提案は既存の Claude Code インフラを活用します：

1. `context: fork` はすでに隔離コンテキストを作成
2. Agent/Skill frontmatter はすでに `tools:` 宣言をサポート
3. MCP 設定はすでに settings.json に存在

主な追加：
- MCP 設定の `lazy: true` フラグ
- Agent/Skill frontmatter の `mcp:` フィールド
- フォークコンテキストに MCP を注入するロジック

## 参考資料

- [claude-world.com の詳細記事](https://claude-world.com/articles/mcp-lazy-loading)
- [Issue #7336: 遅延ロード機能リクエスト](https://github.com/anthropics/claude-code/issues/7336)
- [Issue #11364: MCP ツール定義の遅延ロード](https://github.com/anthropics/claude-code/issues/11364)
- [Scott Spence: MCP サーバーコンテキスト使用量の最適化](https://scottspence.com/posts/optimising-mcp-server-context-usage-in-claude-code)

---

*この RFC は Claude Code のベストプラクティスに焦点を当てた開発者コミュニティ [Claude World](https://claude-world.com) によって公開されています。*

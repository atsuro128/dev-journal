# （タイトル：LSP server 連携方針が未定義）

## 発見日
2026-03-24

## カテゴリ
ai-ops

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 7（基盤構築）

## 問題

AI がコード実装・レビュー時に参照するジャンプ機能として、LSP server をどの言語・どの環境で有効化するかの方針が未定義。
現状の work-breakdown には、LSP server を開発基盤の必須要件として扱うか、AI 運用改善として別管理するかの整理がない。

## 影響

- AI によるコード参照・定義ジャンプ・シンボル探索の効率が安定しない
- Go / TypeScript で必要な言語サーバや設定が後付けになり、環境差分が出やすい
- AI 運用設計として切り出す際に、基盤要件との境界が曖昧になる

## 提案

- LSP server 連携は Step 7 の完了条件には含めず、AI 運用改善の ops issue として管理する
- 対象言語、導入方法、Claude Code からの利用前提、ローカル環境差分の吸収方法を整理する
- 将来的に `ai-dev-framework/operations/` 配下の運用設計へ反映する

---

## 解決内容

LSP サーバー統合を完了した。

### 対象言語と LSP サーバー
| 言語 | LSP サーバー | インストール方法 |
|------|-------------|-----------------|
| Go | gopls | Claude Code プラグイン（`gopls@claude-code-lsps`） |
| TypeScript | vtsls | Claude Code プラグイン（`vtsls@claude-code-lsps`）+ `@vtsls/language-server` グローバルインストール |

### 設定
- `ENABLE_LSP_TOOL=1`（`.claude/settings.json` の `env`）
- `enabledPlugins` に `gopls@claude-code-lsps`, `vtsls@claude-code-lsps` を設定
- Dockerfile に `@vtsls/language-server` を追加（`typescript-language-server` は不要なため削除）

### 動作検証結果（2026-04-04）
- **gopls**: `documentSymbol`, `hover` — 正常動作
- **vtsls**: `documentSymbol` — 正常動作
- 検証対象: `expense-saas/internal/domain/entity.go`, `expense-saas/frontend/src/api/types.ts`

### 修正履歴
1. `ENABLE_LSP_TOOL=1` を settings.json に追加
2. gopls, typescript-language-server を Dockerfile に追加
3. claude-code-lsps マーケットプレイスからプラグインをインストール
4. `typescript-language-server` を `@vtsls/language-server` に置き換え（vtsls プラグインが必要とするバイナリ）

## 解決日
2026-04-04

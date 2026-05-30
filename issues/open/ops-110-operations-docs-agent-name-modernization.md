# ai-dev-framework/operations/ 配下の旧エージェント名を現行 8 体構成に現行化

## 発見日
2026-05-30

## カテゴリ
project-management

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
post-MVP（運用フレームワーク整備）

## ブロッカー
なし

## 問題

`ai-dev-framework/` 配下の運用資料に、現行の `.claude/agents/` 定義（8 体）と一致しない旧エージェント名が複数残存している。Phase 3（`ai-dev-framework/README.md` 整備）のレビューで判明。

### 現状の不整合

| 資料 | 現状 | 旧資料における記述 |
|---|---|---|
| `ai-dev-framework/README.md`（Phase 3 で新規作成） | 現行 8 体（architect / designer / backend-developer / frontend-developer / test-implementer / platform-builder / ops-writer / reviewer）を記載 | — |
| `ai-dev-framework/operations/subagent-design.md` | 旧 15 体構成 | design-architect / basic-designer / impl-architect / design-unit-reviewer / design-cross-reviewer / impl-unit-reviewer / impl-cross-reviewer / infra-builder / ui-designer / api-designer / db-designer / security-designer / test-designer / monitoring-designer 等 |
| `ai-dev-framework/operations/subagent-workflow.md` | 旧構成（basic-designer / design-unit-reviewer 等） | line 115 周辺等 |
| `ai-dev-framework/operations/branch-strategy.md` | 旧エージェント名の言及 1 箇所 | （要確認） |

### 検出した旧エージェント名・出現箇所数

- subagent-design.md: 39 箇所
- subagent-workflow.md: 23 箇所
- branch-strategy.md: 1 箇所
- 合計: 63 箇所

## 影響

- **README と参照先の矛盾**: ai-dev-framework/README.md が現行 8 体を案内している一方、リンク先（operations/subagent-design.md など）は旧 15 体構成。読者がどちらを正と見ればよいか迷う
- **採用担当者・エンジニアの信頼性低下**: ポートフォリオ提示時、設計判断の正本性が崩れていると感じられる
- **新規参加者の混乱**: フレームワークを参照しようとする人が旧資料に騙される可能性

## 提案

### ゴール

`ai-dev-framework/operations/` 配下の資料を、現行 `.claude/agents/` の 8 体構成に一致させる。

### 対応手順

1. **対象範囲確定**: operations/ 配下全ファイルを grep で再確認、旧エージェント名の全出現箇所を列挙
2. **書き換え方針決定**: 旧 15 体 → 現行 8 体の対応関係を整理（例: ui-designer / api-designer / db-designer / security-designer / monitoring-designer / test-designer → designer に統合）
3. **一括書き換え**: 旧エージェント名を全て現行に置換、構造（章立て）も現行 8 体に合わせて再構成
4. **整合性確認**: README + operations/ 配下が一貫した内容になっているか確認
5. **branch-strategy.md の旧エージェント言及も同時修正**

### 担当

`ops-writer` サブエージェント（運用資料・プロセス文書の更新が責務範囲）に委譲予定。

### 補足

本 issue は Phase 3 のレビュー指摘から派生して起票。ops-writer による対応はその場で開始する見込み（issue 起票と並行）。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

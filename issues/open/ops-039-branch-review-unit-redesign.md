# （タイトル：ブランチ単位・レビュー単位・横断レビューの再設計）

## 発見日
2026-03-24

## カテゴリ
design

## 影響度
高

## 発見経緯
Step 8 task-plan の codex レビュー（review-findings 058）対応中に、ブランチ運用ガイドの「1タスク=1ブランチ」ルールが Step 8 の実態に合わないことが判明。議論を深めた結果、以下の3つの論点が浮上。

## 関連ステップ
Step 8〜11（全実装 Step に影響）

## 問題

### 論点1: ブランチ単位
- 現在のガイド（parallel-branch-operation.md）は「1タスク=1短命ブランチ」を原則とする
- しかし「タスク」の粒度が Step によって異なる（Step 8: 作業手順分割、Step 9/10: 機能グループ単位）
- 「1機能=1ブランチ」に変えても「1機能」の定義が曖昧（1 API? 1機能グループ?）
- codex 提案: デフォルトは 1 task = 1 branch、task-plan で明示した場合のみ統合可（二層ルール）

### 論点2: レビュー単位
- Step 8 task-plan では Phase ごとに impl-unit-reviewer による内部レビューを実施
- しかし全 Phase 完了後の横断レビュー（impl-cross-reviewer）が未定義
- Phase 間の整合性（ミドルウェア ↔ API クライアント、スケルトン ↔ 共通基盤、CI ↔ 全成果物）を誰がいつ検証するか不明

### 論点3: ブランチ単位とレビュー単位の関係
- ブランチ単位 = レビュー単位にすべきか？
- 横断レビューはどのタイミングで行うか（Phase 完了ごと？ Step 完了後？）
- PR 単位とレビュー単位がズレた場合の追跡方法

## 影響

- Step 8 の review-findings 058 が解消できない（ガイドの前提が不適切なため）
- Step 9/10 でも同じ問題が発生する
- 横断レビューが欠落したまま実装に入ると、Phase 間の不整合が Step 完了後まで発見されない

## 提案

1. parallel-branch-operation.md のブランチ単位ルールを二層ルールに改定:
   - 基本原則: 1 レビュー可能単位 = 1 短命ブランチ
   - デフォルト: 1 task = 1 branch
   - 例外条件: 逐次依存 + 同一担当 + reviewable → task-plan に理由を明記して統合可
   - 並列タスクの同一ブランチ混在は禁止（維持）
2. Step 8 task-plan に横断レビュー（impl-cross-reviewer）を Step 完了時に追加
3. work-breakdown テンプレートにレビューフロー（unit → cross → codex）を標準化

## 関連ファイル
- `dev-journal/guide/parallel-branch-operation.md`（修正対象）
- `dev-journal/progress-management/task-plans/step8-foundation.md`（横断レビュー追加）
- `dev-journal/review-findings/open/058-step7-parallel-branch-rules-incomplete.md`（この issue で解消）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

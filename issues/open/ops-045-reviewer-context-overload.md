# レビュワーのコンテキスト負荷が高い

## 発見日
2026-03-29

## カテゴリ
performance

## 影響度
中

## 発見経緯
セッション内の議論で発見

## 関連ステップ
全実装 Step（Step 8-11）

## 問題

reviewer エージェントがレビュー実行前に読む資料が多すぎる。

現状の読み込み対象：
1. `.claude/agents/reviewer.md`（エージェント定義）
2. `ai-dev-framework/agents/review-procedure.md`（手順書）
3. 対象 Step の work-breakdown
4. 対象チケット
5. `.claude/rules/implementation-refs.md`（expense-saas 触る場合）
6. 実際のコード/成果物

コードレビューに辿り着く前にコンテキストが圧迫される。

## 提案

指揮役が reviewer 起動時のプロンプトで必要な情報（レビュー観点、対象ファイル、チケットの要点）を事前に絞り込み、reviewer が自分で work-breakdown を探しに行く必要を減らす。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

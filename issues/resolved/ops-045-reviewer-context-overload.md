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

以下の対応でコンテキスト負荷を軽減した。

1. **review-procedure.md 圧縮**（111→74行）: work-breakdown と重複する Step 対応表・上流資料マッピング表を削除
2. **implementation-refs.md を implementation-workflow.md に統合・削除**: ルールファイルを1つ削減
3. **レビュー結果の出力先ルール追加**: PR ベース/ドキュメントベースで出力先を切り替え（reviewer.md, workflow.md）
4. **実装エージェントの完了手順を共通化**: 4エージェントの PR 手順重複を implementation-workflow.md に統合

work-breakdown 自体の分割（reviewer が全文読む問題）は ops-047 として別途起票済み。

## 解決日
2026-03-30

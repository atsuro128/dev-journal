# レビューが work-breakdown のタスクに含まれていない

## 発見日
2026-03-28

## カテゴリ
project-management

## 影響度
高

## 発見経緯
proactive

## 関連ステップ
Step 8〜11（全実装 Step に影響）

## 問題

レビュー（impl-unit-reviewer, impl-cross-reviewer）が workflow.md の「成果物作成フロー」にルールとして記載されているだけで、各 Step の work-breakdown のタスク一覧に含まれていない。

結果として、8-1 の実装完了後にレビューを飛ばしてコミット・完了扱いにしてしまった。タスクとして見えないものはスキップされる。

## 影響

- レビューがスキップされ、品質ゲートが機能しない
- progress.md でレビューの進捗が追跡できない
- 「どのタスクグループがレビュー済みか」が不明

## 提案

レビューを work-breakdown のタスク一覧に明示的に追加する（案 A）。

- Step 8 例: `8-R1: プロジェクト初期化レビュー（依存: 8-1, 8-2, 8-3, 8-10）`
- 対象: Step 8〜11 の work-breakdown + work-breakdown テンプレート
- レビュータスクのチケット起票も必要か検討する

### 変更対象

- work-breakdown テンプレート（レビュータスクの導出ルール追加）
- Step 8〜11 の work-breakdown（レビュータスクをタスク一覧に追加）
- Step 8 のチケット（レビュー分の追加起票）
- progress.md（レビュータスクの追加）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

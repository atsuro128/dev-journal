# 060: Step7 task plan に計画レビューの blocking 指摘対応方針がない

## 指摘概要
work-breakdown は、作業計画レビューで出た blocking 指摘への対応方針を task plan に整理することを要求しているが、対象 task plan にはその記載がない。Phase 実行後の内部レビュー手順はあるものの、「計画レビューで blocking が出た場合に、誰がどこを更新し、解消確認後に着手可とするか」が未定義のため、レビューゲートを通過する条件が曖昧なままになっている。

## 根拠
- `dev-journal/guide/work-breakdown/step7-foundation.md:31-37`
  - 計画レビューゲートの確認項目として「作業計画レビューで出た blocking 指摘への対応方針が整理されているか」を要求している。
- `dev-journal/progress-management/task-plans/step7-foundation.md:52-65`
  - 各 Phase の実行前後手順はあるが、計画レビューの blocking 指摘の扱いは定義していない。
- `dev-journal/progress-management/task-plans/step7-foundation.md:69-260`
  - 以降の各 Phase・統合ルール・品質基準にも、計画レビュー findings の反映先や着手再開条件が見当たらない。

## 判定
中 / レビューゲート未定義

## 修正方針案
- task plan に「計画レビュー findings 管理」節を追加し、blocking 指摘が open の間は成果物作成を開始しないことを明記する。
- 各指摘について、更新対象セクション、解消確認者、着手再開条件を定義する。
- `dev-journal/review-findings/` との対応付け方法も明示する。

## 対応内容
対応不要。以下の既存ルールで blocking 指摘の対応方針はカバー済みであり、task-plan に重複して記載する必要はない。

- `.claude/rules/workflow.md` 品質ゲート判定基準: 「blocker があれば修正 → 再レビュー（LGTM まで繰り返す）」
- `.claude/rules/workflow.md` 意思決定権限: 「blocker の修正方針はユーザー承認が必須」
- `dev-journal/guide/work-breakdown/step7-foundation.md` 計画レビューゲート: 「通過するまで成果物作成に入ってはならない」

これらのルールが指揮役の行動を拘束しており、task-plan 単体に同内容を再記述しても重複管理になるだけで実効性は変わらない。

## 再レビュー結果（2026-03-24）

対応不十分（差し戻し）。

### 確認内容
- `.claude/rules/workflow.md` には blocker 修正と再レビューの一般ルールがあり、`work-breakdown/step7-foundation.md` には計画レビュー通過前に成果物作成へ入らないことが記載されている。
- しかし `work-breakdown/step7-foundation.md` は、Step 7 の task plan 自体について「作業計画レビューで出た blocking 指摘への対応方針が整理されているか」をレビュー観点として明示している。
- 現在の `step7-foundation.md` には、計画レビュー findings の反映先、blocking が open の間の着手可否、解消確認後に再開する条件が task-plan 固有ルールとして整理されていない。

### 差し戻し理由
既存 workflow ルールで一般論はカバーできても、Step 7 の task-plan に明記すべきだという上流要求は満たしていない。`対応不要` の根拠は「重複回避」だが、今回の論点は task-plan にレビューゲート接続を明示すること自体なので、上流要求との整合が取れていない。

### 追加で確認・修正すべき点
- task-plan に、計画レビュー findings の管理方法、blocking finding が `open` の間は成果物作成を開始しないこと、解消確認後に着手再開する条件を明記すること。
- `dev-journal/review-findings/` との対応関係を task-plan から追えるようにすること。

## 追加対応予定
次セッションで task-plan に計画レビュー指摘管理セクションを追加し再レビューを実施する。

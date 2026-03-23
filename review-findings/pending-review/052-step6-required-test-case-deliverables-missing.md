# 052: Step 6 の必須成果物（作業計画・機能別テストケース）が未作成

## 指摘概要
Step 6 は `test_strategy.md` だけが存在し、作業分解で必須とされている作業計画ファイルと 8 本の `test_cases/*.md` が未作成である。これでは Step 6 の完了条件である「重要領域のテストケースが網羅されている」を満たせず、Step 7 の実装者が機能別テストを直接起こせない。

## 根拠
- `dev-journal/guide/work-breakdown/step6-testing.md:25`-`29`
  - Step 着手時に `progress-management/task-plans/step6-testing.md` を保存するよう要求している。
- `dev-journal/guide/work-breakdown/step6-testing.md:33`-`36`
  - Step 全体の完了条件として「重要領域のテストケースが網羅されている」を要求している。
- `dev-journal/guide/work-breakdown/step6-testing.md:83`-`91`
  - `auth.md` から `cross-cutting.md` まで 8 本のテストケース成果物を定義している。
- `dev-journal/guide/work-breakdown/step6-testing.md:72`-`75`
  - `test_cases/*.md` が Step 7 の実装に必要な情報を単独で提供することを reviewer 観点としている。
- `dev-journal/deliverables/docs/60_test/test_strategy.md`
  - 現在存在する Step 6 成果物はこの 1 ファイルのみ。

## 判定
- 重大度: 高
- 分類: 完了条件未達 / 下流作業阻害

## 修正方針案
- `progress-management/task-plans/step6-testing.md` を追加し、Step 6 の作業順序と完了基準を明示する。
- `deliverables/docs/60_test/test_cases/` を作成し、`auth.md` `reports.md` `items.md` `attachments.md` `workflow.md` `dashboard.md` `tenant.md` `cross-cutting.md` を成果物として揃える。
- 各ファイルに少なくとも「テストID / テストレベル / レイヤー / テスト関数名候補 / 入力 / 期待結果」を記載し、Step 6 のレビュー観点 6 を満たす粒度まで落とし込む。

## 対応内容

本指摘は Phase 1（テスト戦略のみ）のコミット時点でのレビューであり、test_cases/*.md 8ファイルは Phase 2（6-B-1〜7）・Phase 3（6-B-8）で作成予定。

- 作業計画: `progress-management/task-plans/step6-testing.md` に作成済み（コミット c4ba5ed）
- Phase 2: 機能別テストケース 7ファイル（6-B-1〜7、並列実行）
- Phase 3: 横断テストケース 1ファイル（6-B-8）

Step 6 は Phase 1 完了の段階であり、Step 全体の完了条件は Phase 3 完了後に満たされる。

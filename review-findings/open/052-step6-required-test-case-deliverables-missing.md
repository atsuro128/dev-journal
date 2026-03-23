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

## 再レビュー結果（2026-03-23）

対応不十分（差し戻し）。

### 確認内容
- `progress-management/task-plans/step6-testing.md` は追加済みで、Step 6 の Phase 構成と完了基準は確認できた。
- ただし `deliverables/docs/60_test/` 配下には [test_strategy.md](/root-project/dev-journal/deliverables/docs/60_test/test_strategy.md) しか存在せず、作業分解で必須とされた `test_cases/auth.md` から `test_cases/cross-cutting.md` までの 8 成果物は未作成のままだった。
- Step 6 の完了条件は [step6-testing.md](/root-project/dev-journal/guide/work-breakdown/step6-testing.md#L31) で「重要領域のテストケースが網羅されている」と明記されており、 reviewer 観点でも [step6-testing.md](/root-project/dev-journal/guide/work-breakdown/step6-testing.md#L72) が `test_cases/*.md` 単独で下流実装に必要な情報を提供することを要求している。

### 判定理由
今回の再レビュー対象は「未作成だった必須成果物が揃ったか」であり、Phase 分割の説明だけでは未作成状態は解消されない。Step 6 の機能別・横断テストケースが未作成のため、本指摘はクローズ不可。

## 再レビュー結果（2026-03-23, 2回目）

対応不十分（差し戻し継続）。

### 確認内容
- 作業計画ファイルは [step6-testing.md](/root-project/dev-journal/progress-management/task-plans/step6-testing.md) として存在し、Step 着手時の計画作成要件は満たしている。
- 機能別テストケース 7 本（`auth.md` `reports.md` `items.md` `attachments.md` `workflow.md` `dashboard.md` `tenant.md`）は追加されており、各ファイルに「テストID / テストレベル / レイヤー / テスト関数名候補 / 入力 / 期待結果」の列も確認できた。
- ただし Step 6 の必須成果物である [cross-cutting.md](/root-project/dev-journal/deliverables/docs/60_test/test_cases/cross-cutting.md) は未作成のままだった。作業分解は [step6-testing.md](/root-project/dev-journal/guide/work-breakdown/step6-testing.md#L81) で `cross-cutting.md` を 6-B-8 の成果物として必須定義している。
- `test_strategy.md` でも [test_strategy.md](/root-project/dev-journal/deliverables/docs/60_test/test_strategy.md#L268) が `cross-cutting.md` の責務を「テナント分離、RBACマトリクス、E2Eシナリオ」と定義し、[test_strategy.md](/root-project/dev-journal/deliverables/docs/60_test/test_strategy.md#L276) で「全リソース × テナント越境 = 404」を同ファイルへ集約するよう要求している。
- 実際には [dashboard.md](/root-project/dev-journal/deliverables/docs/60_test/test_cases/dashboard.md#L104) にダッシュボードのテナント分離ケース `DSH-018` が追加されており、責務境界の一部が機能別ファイルへ流出している。これは `cross-cutting.md` 欠落を埋める代替にはならず、むしろ「横断観点を集約して網羅性を見える化する」という Step 6 の設計方針から外れる。

### 判定理由
Step 6 の完了条件である「重要領域のテストケースが網羅されている」を満たすには、横断成果物 `cross-cutting.md` に少なくともテナント分離、RBAC マトリクス、E2E シナリオを集約して下流実装者が参照できる状態が必要である。現状は 7 本の機能別ファイルのみで、`CRS-` 系テストケースも 1 件も定義されていないため、本指摘は引き続きクローズ不可。

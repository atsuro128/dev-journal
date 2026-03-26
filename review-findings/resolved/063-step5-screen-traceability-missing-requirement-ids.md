# 063: 一部 screen 仕様の冒頭に要件 ID がなく Step 6 へのトレーサビリティが切れている

## 指摘概要
Step 5 の正本では `screens/*.md` の冒頭に対応する UC-ID と要件 ID を記載することを求めているが、管理系とワークフロー系の一部画面仕様では冒頭に UC-ID のみがあり、要求 ID が示されていない。これでは Step 6 が画面仕様から要件へ直接追跡できず、品質ゲートの「要件 ID・設計 ID から追跡できる」を満たせない。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step5-detail-design.md:208-215`
  - `screens/*.md` は「対応する UC-ID と要件 ID を冒頭に記載する」と定義している
- `dev-journal/deliverables/docs/50_detail_design/screens/admin-all-reports.md:17-23`
  - 冒頭の基本情報に UC はあるが要件 ID がない
- `dev-journal/deliverables/docs/50_detail_design/screens/admin-tenant.md:17-23`
  - 冒頭の基本情報に UC はあるが要件 ID がない
- `dev-journal/deliverables/docs/50_detail_design/screens/workflow-pending.md:17-23`
  - 冒頭の基本情報に UC はあるが要件 ID がない
- `dev-journal/deliverables/docs/50_detail_design/screens/workflow-payable.md:17-23`
  - 冒頭の基本情報に UC はあるが要件 ID がない

## 判定
- 重大度: 中
- 分類: トレーサビリティ不足 / 完了条件未達

## 修正方針案
- 各画面の基本情報テーブルに `対応要件ID` または `対応機能ID` を追加する
- `requirements.md` と `policies.md` を突き合わせ、画面ごとに実装・検証で参照すべき機能要件 ID と業務ルール ID を固定する
- Step 6 が画面仕様単体から要件へ辿れることを確認する

## 再レビュー結果
- **差し戻し**: 当初指摘していた 4 画面は是正されているが、Step 5 正本が要求する「`screens/*.md` の冒頭に UC-ID と要件 ID を記載する」条件はまだ全画面で満たされていない
- 確認できた未解消箇所:
  - `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` の基本情報に要件 ID がない（UC のみ）
  - `dev-journal/deliverables/docs/50_detail_design/screens/report-list.md` の基本情報に要件 ID がない（UC のみ）
  - `dev-journal/deliverables/docs/50_detail_design/screens/report-create.md` の基本情報に要件 ID がない（UC のみ）
  - `dev-journal/deliverables/docs/50_detail_design/screens/report-edit.md` の基本情報に要件 ID がない（UC のみ）
  - `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` の基本情報に要件 ID がない（UC のみ）
- 追加で確認すべき観点:
  - `対応要件ID` / `対応機能ID` の表記ゆれを解消し、Step 6 が screen 冒頭だけで要件トレースできる形式に揃えること
  - 複数要件にまたがる画面は、参照すべき要件 ID を冒頭で明示し、本文参照に依存しないこと
- このまま進めた場合の問題:
  - Step 6 が画面仕様から要件へ直接追跡できず、品質ゲートの「要件 ID・設計 ID から追跡できる」を満たせない

## 今回の再レビュー結果
- **解消済み**: 前回未解消だった 5 画面に `対応要件ID` が追加され、今回確認した 9 画面（`dashboard.md`, `report-list.md`, `report-create.md`, `report-edit.md`, `report-detail.md`, `admin-all-reports.md`, `admin-tenant.md`, `workflow-pending.md`, `workflow-payable.md`）の冒頭で UC-ID と要件 ID を直接追跡できる状態になっている
- 確認内容:
  - `ai-dev-framework/guide/work-breakdown/step5-detail-design.md` のトレーサビリティ要件 `screens/*.md: 対応する UC-ID と要件 ID を冒頭に記載する` と照合
  - 上記 9 画面の基本情報テーブルで `対応要件ID` が明示されていることを確認
  - 要件書 `dev-journal/deliverables/docs/10_requirements/requirements.md` に各 ID（`DASH-F01`, `RPT-F01`〜`RPT-F07`, `ADM-F01`, `WFL-F04`, `WFL-F05`）の定義元が存在することを確認
- 判定理由:
  - 本指摘の論点だった「画面仕様の冒頭から Step 6 が要件へ直接トレースできない」状態は解消された
  - 今回の修正により、前回差し戻しで列挙した未解消箇所はすべて塞がっている

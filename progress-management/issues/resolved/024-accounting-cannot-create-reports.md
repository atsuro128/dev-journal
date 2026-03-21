# Accounting がレポート作成不可は業界標準から乖離

## 発見日
2026-03-19

## 発見経緯
proactive（市場調査による発見）

## 関連ステップ
Step 1（rbac.md, usecases.md, requirements.md）、Step 4（screens.md, ui_flow.md）

## カテゴリ
requirements / rbac

## 問題
現行設計では Accounting ロールはレポート作成不可（経費閲覧・支払完了記録のみ）だが、市場調査の結果、調査した全8製品で経理担当者も自分の経費を申請可能。

「経理担当者も従業員である」という基本前提に基づき、自己申請は許可しつつ、自己処理（自分の申請を自分で経理処理）は禁止するのが業界標準。

## 影響
- Accounting ロールのユーザーが自分の経費を精算できない
- 実運用では経理担当者に別途 Member ロールを持たせるなどの回避策が必要になるが、1ユーザー1ロール制約（RBC-002）と矛盾
- 業界標準から乖離しており、製品としての実用性に疑問

## 提案
1. Accounting に Member 相当の経費申請権限を付与（レポート作成・編集・提出・明細追加・添付）
2. 自己処理禁止ルールを追加: Accounting は自分が作成したレポートの支払完了を記録できない
3. Approver と同様の「自分のレポートは自分で処理できない」パターン

### 影響する上流成果物
- `10_requirements/rbac.md` — 権限マトリクス修正（Accounting の申請系操作を ○/※ に変更）
- `10_requirements/usecases.md` — UC-SYS04 ダッシュボードの Accounting 表示修正
- `10_requirements/requirements.md` — DASH-003 の修正
- `40_basic_design/screens.md` — Accounting のサイドナビにマイレポート・レポート作成を追加
- `40_basic_design/ui_flow.md` — Accounting の遷移パスに申請フローを追加

## 解決内容

提案どおり Accounting に Member 相当の経費申請権限を付与し、自己処理禁止ルールを追加。以下の8ファイルを修正。

**要件定義（Step 1）**:
- rbac.md — 権限マトリクス拡大、RBC-012 に自己処理禁止条件を統合、特殊ルール追加
- usecases.md — 全 Member UC のアクター欄に Accounting 追加、UC-AC02 に自己処理禁止の例外系追加、サマリ表整合
- requirements.md — ロール定義修正、RBAC-F02 修正、WFL-F03 に自己処理禁止明示、ビジネスルール・MVP スコープ判断追加、DASH-003 修正
- workflow.md — 状態遷移図・T1/T4/T5 の実行者・事前条件修正

**ドメイン設計（Step 2）**:
- domain_model.md — 不変条件に RBC-012（自己処理禁止）追加、SelfPaymentNotAllowed エラー新設
- state_machine.md — T1/T4/T5 の実行者・事前条件修正

**基本設計（Step 4）**:
- screens.md — ダッシュボード・サイドナビ・レポート系画面の Accounting 対応、支払完了ボタン条件修正
- ui_flow.md — Accounting の申請フロー追加、Mermaid 図更新

**ルールID整理**: 自己処理禁止を RBC-012 に統一。WFL-013 は元の遷移ルール定義に復元。

**preliminary 文書**:
- 04_business-rules.md — RPT-010/WFL-010/WFL-013/WFL-014/RBC-010/RBC-012 の Accounting 関連修正、確定論点追加
- 01_business-overview.md — Accounting 説明に自己申請可能・自己処理禁止を反映

## 解決日
2026-03-21

## レビュー結果

### 2026-03-21 Codex Issue 解決レビュー

- 判定: 差し戻し
- 確認1: `rbac.md` `requirements.md` `usecases.md` `workflow.md` `domain_model.md` `state_machine.md` `screens.md` `ui_flow.md` では、Accounting の申請権限付与と自己処理禁止が一貫して反映されている。
- 確認2: `workflow.md` の `T4` は `RBC-012` を参照する定義に更新され、自己処理禁止ルールのトレーサビリティも Step 1 / Step 2 間で回復している。
- 指摘1: Step 1 の上流整理文書 `preliminary/01_business-overview.md` が依然として Accounting を「承認済み経費の確認・支払処理」の役割としてのみ説明しており、今回の解決内容である「自分の経費申請も可能」という定義が未反映。詳細は `review-findings/open/038-preliminary-business-overview-still-excludes-accounting-self-application.md` を参照。
- 再対応方針: `preliminary/01_business-overview.md` のロール概要と業務フローを、最終要件と同じ Accounting 定義へ更新すること。

### 2026-03-21 Codex 最終レビュー

- 判定: 解決
- 確認1: 前回指摘 038 の対象だった `preliminary/01_business-overview.md` で、Accounting が「自分の経費申請も可能」であり、かつ「自分のレポートの支払完了は記録不可」であることが反映された。
- 確認2: `rbac.md` `usecases.md` `requirements.md` `workflow.md` `domain_model.md` `state_machine.md` `screens.md` `ui_flow.md` `04_business-rules.md` `01_business-overview.md` の10ファイルで、Accounting の申請権限付与と自己処理禁止ルールが整合している。
- 確認3: Step 1 の上流整理文書、Step 1 要件、Step 2 ドメイン設計、Step 4 基本設計の間で、今回の変更による新たな矛盾・未反映は確認されなかった。

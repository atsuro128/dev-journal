# 125: test_strategy §4.2 の Test Member Empty 用途が seed.go と一致していない

## 指摘概要
`test_strategy.md` §4.2 はテナント A の Member 2 名を「明細あり用・明細なし用」と説明しているが、`seed.go` の実装では `test-member-empty@example.com` はユーザー・メンバーシップのみ投入され、明細なしレポート `report_draft_empty` の作成者ではない。

`report_draft_empty` は `UserMemberID`（`test-member@example.com`）を `user_id` として投入されているため、`test-member-empty@example.com` は「レポート 0 件のユーザー」として扱うのが実装に一致する。§4.2 の説明は seed.go から導けない推測値になっている。

## 根拠
- `dev-journal/deliverables/docs/60_test/test_strategy.md:163`
  - 「Member は明細あり用・明細なし用の 2 名」
- `expense-saas/internal/seed/seed.go:180`
  - `UserMemberEmptyID` は `test-member-empty@example.com` として投入される
- `expense-saas/internal/seed/seed.go:210`
  - `UserMemberEmptyID` は Tenant A の Member としてメンバーシップ登録される
- `expense-saas/internal/seed/seed.go:275`
  - `ReportDraftEmptyID` は `memberID`（`UserMemberID`）を作成者として投入される
- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md:160`
  - SMK-084 では `test-member-empty@example.com` を「レポート 0 件のユーザー」として参照している

## 判定
重大度: 中
分類: 設計書と実装の不整合 / テストデータ定義の誤記

Step 6 の品質ゲート「Step 11-A の実施者が前提データを見て迷わず実施できる」に抵触する。`test-member-empty@example.com` を明細なしレポート用ユーザーと読ませると、手動確認やテスト実装で誤ったログインユーザーを選ぶ可能性がある。

## 修正方針案
`test_strategy.md` §4.2 の説明を seed.go に合わせて修正する。

- `Test Member`: レポートフィクスチャの作成者。`report_draft_empty` を含むテナント A レポートを所有する
- `Test Member Empty`: レポート 0 件の空状態表示確認用ユーザー

「明細あり用・明細なし用」という説明は削除し、§4.3 の「テナントA所属、作成者: Test Member」と矛盾しない表現にする。

---

## 対応内容（2026-06-01）
§4.2 テナントA ユーザー構成の説明文を修正。「Member は明細あり用・明細なし用の 2 名」→「Test Member（全レポートフィクスチャの作成者、`report_draft_empty` を含むテナントA レポートを所有）と Test Member Empty（空状態表示確認用のレポート 0 件ユーザー、SMK-084 用）の 2 名」に変更し、seed.go（`report_draft_empty` 作成者は `memberID`=`UserMemberID`）と SMK-084 の定義に整合させた。

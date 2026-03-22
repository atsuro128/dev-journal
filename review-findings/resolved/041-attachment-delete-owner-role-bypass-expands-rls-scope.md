---
step: 5
severity: high
status: resolved
---

# 041: 添付削除だけ業務フローでオーナーロールを使う設計が RLS 境界を崩している

## 指摘概要
`attachments.deleted_at` を更新するために、通常の業務操作である添付削除だけ `expense_owner` ロールの専用接続を使う設計になっています。これは「オーナーロールはマイグレーションと認証処理のみ」「通常リクエストは `expense_app` で RLS 適用」という上流方針と衝突します。添付削除を実装すると、業務 API の一部が RLS バイパス経路を持つことになり、TenantContext と RLS の二重保証が崩れます。

## 根拠
- [architecture.md](/root-project/dev-journal/deliverables/docs/30_arch/architecture.md#L66) ではリポジトリ層が `tenant_id` 強制と論理削除を担うとしている
- [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L279) では `expense_owner` は「マイグレーション + 認証処理」、`expense_app` は「通常リクエスト」と定義している
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L551) では `attachments` に UPDATE ポリシーを作らず、`deleted_at` 設定をオーナーロールで実行するとしている
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L613) では「添付ファイル論理削除」を RLS バイパスが必要なケースに追加している
- [files.md](/root-project/dev-journal/deliverables/docs/50_detail_design/files.md#L387) では添付削除 API が通常の業務エンドポイント `DELETE /api/reports/:id/items/:itemId/attachments/:attId` として定義されている

## 判定
高 / テナント分離・認可設計の破綻リスク

## 修正方針案
添付削除も通常の業務フローとして `expense_app` + RLS の枠内で処理できるように設計を戻す。
- `attachments` に `UPDATE` ポリシーを追加し、`deleted_at` 更新を `expense_app` で実行可能にする
- もしくは添付削除を論理削除ではなく物理 `DELETE` に寄せるが、その場合は `files.md` と `db_schema.md` の論理削除方針を全面的に揃え直す
- いずれにしても、`expense_owner` の用途を業務 API に拡張しない方針を明文化する

## 再レビュー結果（2026-03-22）

対応妥当（クローズ）。

### 確認内容
- `attachments` に `FOR UPDATE` の RLS ポリシーが追加され、`deleted_at` を通常の `expense_app` 接続で更新できる設計に修正された
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L541)
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L559)
- `RLS バイパスが必要なケース` から添付削除が除去され、オーナーロール利用は認証系とマイグレーションに限定された
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L617)
- `security.md` でも `expense_owner` を「マイグレーション + 認証処理」に限定し、業務用リポジトリ層へ公開しない方針が維持されている
  - [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L279)
  - [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L285)
- `files.md` の添付削除フローは引き続き通常業務 API として定義されており、RLS 適用下の論理削除方針と整合している
  - [files.md](/root-project/dev-journal/deliverables/docs/50_detail_design/files.md#L387)
  - [files.md](/root-project/dev-journal/deliverables/docs/50_detail_design/files.md#L399)
- `db_schema.md` の完了チェックにも「オーナーロール不要」が明記され、後続レビュー観点へ反映された
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L926)

### 判定理由
添付削除が通常の業務リクエストとして `expense_app` + RLS の境界内に戻され、上流のテナント分離方針と矛盾しない設計に整理されたため。

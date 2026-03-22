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

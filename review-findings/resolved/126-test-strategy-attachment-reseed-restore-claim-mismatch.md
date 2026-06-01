# 126: test_strategy §4.6 の添付削除後復元説明が seed.go の実処理と一致していない

## 指摘概要
`test_strategy.md` §4.6 は、SMK-038 で添付を削除した後も seed 再実行で削除前状態を復元できると説明している。しかし `seed.go` の添付投入は `ON CONFLICT (attachment_id) DO NOTHING` であり、既存の添付行に対して `deleted_at = NULL` に戻す UPDATE は行っていない。

添付削除は論理削除（`deleted_at` 設定）なので、削除済みの `AttachmentDraftID` 行が残っている場合、seed 再実行時の INSERT は競合してスキップされ、削除前状態には戻らない。§4.6 の「復元できる」は seed.go の実装から導けない推測値になっている。

## 根拠
- `dev-journal/deliverables/docs/60_test/test_strategy.md:260`
  - 「SMK-038 で削除した後も seed 再実行で削除前状態を復元できる（`ON CONFLICT DO NOTHING` による冪等投入）」
- `expense-saas/internal/seed/seed.go:594-608`
  - `AttachmentDraftID` を INSERT し、`ON CONFLICT (attachment_id) DO NOTHING` で競合時は何もしない
- `expense-saas/db/queries/attachments.sql:22-29`
  - 添付削除は `UPDATE attachments SET deleted_at = now()` による論理削除

## 判定
重大度: 中
分類: 設計書と実装の不整合 / テストデータ復元手順の誤記

Step 6 のレビュー観点「テストデータ戦略（テナント分離を考慮したシード設計）が定義されているか」「下流作業可能性」に抵触する。設計書を信じて SMK-038 後に seed 再実行だけで前提データが戻ると判断すると、削除済み添付が一覧・ダウンロード対象から消えたままになり、後続の手動確認や統合テストの前提が崩れる。

## 修正方針案
正本を seed.go の実装とするなら、`test_strategy.md` §4.6 の復元説明を削除または修正する。

推奨表現:

「`ON CONFLICT DO NOTHING` による冪等投入のため、未投入環境には添付レコードを追加する。既に論理削除済みの同一 `attachment_id` は seed 再実行だけでは復元されない。」

もし seed 再実行で削除前状態を復元する仕様にしたい場合は、別途 seed.go 側に `deleted_at = NULL` の補完 UPDATE を追加し、設計書と実装を同時に更新する。

---

## 対応内容（2026-06-01）
§4.6 添付フィクスチャの復元説明を修正。「SMK-038 で削除した後も seed 再実行で削除前状態を復元できる」という誤記を削除し、「`ON CONFLICT (attachment_id) DO NOTHING` のため未投入環境には追加するが、論理削除（`deleted_at` 設定）済みの同一 attachment_id は seed 再実行では `deleted_at` を戻す UPDATE を行わないため復元されない。復元には DB クリーンアップ後の再 seed が必要」に修正。seed.go L594-608 / attachments.sql の論理削除実装に整合させた。

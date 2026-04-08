# 添付ファイル

- 担当: backend-developer, frontend-developer
- 依存: 10-E
- ブランチ: `step10/10-G-attachments`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 添付ファイルエンドポイント（/reports/{id}/items/{id}/attachments/*） |
| 添付ファイル設計 | deliverables/docs/50_detail_design/files.md | ファイル形式制限、サイズ制限、S3 パス規約 |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | attachments |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | 添付ファイルの認可ルール |
| 画面仕様（レポート詳細） | deliverables/docs/50_detail_design/screens/report-detail.md | 添付セクション |
| UI コンポーネント設計（レポート系） | deliverables/docs/55_ui_component/screens/report-detail.md | 添付コンポーネントツリー・Props 型 |
| テストケース（添付ファイル） | deliverables/docs/60_test/test_cases/attachments.md | 全テストケース |
| BE テストコード | expense-saas/internal/handler/attachment_handler_test.go | ハンドラテスト |
| FE テストコード（部品） | expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.test.tsx, AttachmentArea.integration.test.tsx, AttachmentList.test.tsx, AttachmentUploader.test.tsx | コンポーネントテスト |
| FE テストコード（Hooks） | expense-saas/frontend/src/hooks/__tests__/useAttachments.test.tsx, useUploadAttachment.test.tsx, useAttachmentDownload.test.tsx, useDeleteAttachment.test.tsx | 添付関連 Hooks |

## 責務

- 署名付き URL 発行 API の機能実装（アップロード・ダウンロード用）
- オブジェクトストレージクライアント（S3）の機能実装
- ファイルバリデーション（形式・サイズ制限）の実装
- 添付ファイルメタデータの CRUD（作成・一覧・削除）
- 添付エリア UI の機能実装（アップロード・プレビュー・削除）
- 含めない: 明細 CRUD（10-E）、レポート CRUD（10-B）

## 完了条件

- test_cases/attachments.md の全テストケースが通過している
- Step 9 で実装済みの添付ファイル関連テストコード（BE/FE）が全て PASS する

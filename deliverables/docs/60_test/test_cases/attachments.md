# 添付ファイルテストケース一覧

## 概要

本書は添付ファイル（Attachment）に関する4エンドポイントのテストケースを定義する。

| 項目 | 内容 |
|------|------|
| テストID プレフィックス | `ATT-` |
| 対象エンドポイント | uploadAttachment, listAttachments, getAttachmentDownload, deleteAttachment |
| 対応ハンドラファイル | `attachment_handler_test.go` |
| テスト戦略参照 | `60_test/test_strategy.md` |

### 責務境界

| テスト観点 | 本ファイルに書く | 書かない |
|-----------|---------------|---------|
| RBAC（エンドポイント別権限） | o | |
| 署名付きURL認可チェック | o | |
| ファイルバリデーション（MIME・サイズ） | o | |
| レポート状態による操作制限 | o | |
| テナント分離（クロステナントアクセス） | | `cross-cutting.md` に記載 |

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `50_detail_design/openapi.yaml` | 添付ファイル4エンドポイントのスキーマ・エラーコード |
| `50_detail_design/files.md` | MIME制限・サイズ制限・署名付きURL設計・認可フロー |
| `50_detail_design/authz.md` | §6.5 添付ファイル認可ルール、§7.3 認可継承、§4.5 閲覧権限 |

---

## フィクスチャ

`test_strategy.md` §4 の標準フィクスチャに加え、添付テスト専用のフィクスチャを定義する。

### 添付ファイルフィクスチャ（テナントA所属）

| フィクスチャ名 | attachment_id | 親レポート | 親明細 | MIME タイプ | ファイルサイズ | 備考 |
|-------------|--------------|----------|--------|------------|------------|------|
| `att_draft_jpeg` | `ffffffff-0001-0001-0001-000000000001` | `report_draft` | `dddddddd-0001-0001-0001-000000000001` | `image/jpeg` | 245,760 B | 削除・ダウンロードテスト用 |
| `att_submitted_pdf` | `ffffffff-0002-0002-0002-000000000002` | `report_submitted` | `dddddddd-0002-0002-0002-000000000002` | `application/pdf` | 102,400 B | submitted レポートへのダウンロードテスト用 |
| `att_approved_png` | `ffffffff-0003-0003-0003-000000000003` | `report_approved` | `dddddddd-0003-0003-0003-000000000003` | `image/png` | 51,200 B | approved レポートへのダウンロードテスト用 |

**注意**: `report_submitted` の明細IDおよび `report_approved` の明細IDは、各テストケースで使用するフィクスチャとして `dddddddd-0002-0002-0002-000000000002` および `dddddddd-0003-0003-0003-000000000003` を追加定義する。

### テストファイル定義（バイナリ）

| ファイル識別子 | MIME タイプ | サイズ | マジックバイト | 備考 |
|-------------|------------|--------|--------------|------|
| `file_valid_jpeg` | `image/jpeg` | 1,024 B | `FF D8 FF` | 正常なJPEGファイル |
| `file_valid_png` | `image/png` | 1,024 B | `89 50 4E 47 0D 0A 1A 0A` | 正常なPNGファイル |
| `file_valid_pdf` | `application/pdf` | 1,024 B | `25 50 44 46`（`%PDF`） | 正常なPDFファイル |
| `file_too_large` | `image/jpeg` | 5,242,881 B | `FF D8 FF` | 5MB超過（1バイトオーバー） |
| `file_exactly_5mb` | `image/jpeg` | 5,242,880 B | `FF D8 FF` | ちょうど5MB（境界値・許可） |
| `file_invalid_type_gif` | `image/gif` | 1,024 B | `47 49 46 38` | 許可されていないGIF |
| `file_spoofed_jpeg` | `image/jpeg` と宣言するが中身はGIF | 1,024 B | `47 49 46 38` | MIME偽装ファイル（マジックバイト不一致） |
| `file_no_content_type` | なし | 1,024 B | `FF D8 FF` | Content-Type ヘッダーなし |

---

## 1. uploadAttachment（POST /api/reports/{id}/items/{itemId}/attachments）

### 1-1. 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-001 | 統合 | handler | `TestUploadAttachment_Success_JPEG` | 前提: `report_draft`（draft）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 201 Created。レスポンス body に `data.id`, `data.item_id`, `data.file_name`, `data.file_size`（1024）, `data.mime_type`（image/jpeg）, `data.created_at` を含む。`attachments` テーブルにレコードが挿入される | 高 |
| ATT-002 | 統合 | handler | `TestUploadAttachment_Success_PNG` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_valid_png` | 201 Created。`data.mime_type` が `image/png` | 高 |
| ATT-003 | 統合 | handler | `TestUploadAttachment_Success_PDF` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_valid_pdf` | 201 Created。`data.mime_type` が `application/pdf` | 高 |
| ATT-004 | 統合 | handler | `TestUploadAttachment_Success_ExactlyMaxSize` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_exactly_5mb`（5,242,880 B） | 201 Created。サイズ境界値（ちょうど5MB）は許可される | 高 |
| ATT-005 | 統合 | handler | `TestUploadAttachment_Success_Approver` | 前提: `report_draft`（Test Approver が作成したレポート）、認証: Test Approver（所有者）、ファイル: `file_valid_jpeg` | 201 Created。Approver が所有者の場合はアップロード可能 | 中 |

### 1-2. ファイルバリデーション異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-006 | 統合 | handler | `TestUploadAttachment_FileTooLarge` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_too_large`（5,242,881 B） | 413 FILE_TOO_LARGE。`attachments` テーブルにレコードが挿入されない | 高 |
| ATT-007 | 統合 | handler | `TestUploadAttachment_InvalidMimeType_GIF` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_invalid_type_gif`（Content-Type: image/gif） | 422 INVALID_FILE_TYPE。GIF は許可リスト外 | 高 |
| ATT-008 | 統合 | handler | `TestUploadAttachment_SpoofedMimeType` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_spoofed_jpeg`（Content-Type: image/jpeg と宣言、マジックバイトはGIF） | 422 INVALID_FILE_TYPE。Content-Type とマジックバイトの不一致で拒否 | 高 |
| ATT-009 | 統合 | handler | `TestUploadAttachment_MissingFilePart` | 前提: `report_draft`、認証: Test Member（所有者）、リクエストボディ: multipart/form-data だが `file` パートなし | 400 INVALID_REQUEST（BAD_REQUEST） | 高 |
| ATT-010 | 統合 | handler | `TestUploadAttachment_NoContentType` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_no_content_type`（Content-Type ヘッダーなし） | 422 INVALID_FILE_TYPE | 中 |

### 1-3. RBAC 異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-011 | 統合 | handler | `TestUploadAttachment_Unauthorized` | 認証: なし（Authorization ヘッダーなし）、ファイル: `file_valid_jpeg` | 401 UNAUTHORIZED | 高 |
| ATT-012 | 統合 | handler | `TestUploadAttachment_Forbidden_NotOwner` | 前提: `report_draft`（所有者: Test Member）、認証: Test Approver（別ユーザー、所有者でない）、ファイル: `file_valid_jpeg` | 403 FORBIDDEN。同一テナント内の非所有者は拒否（authz.md §7.2） | 高 |

### 1-4. レポート状態異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-013 | 統合 | handler | `TestUploadAttachment_ReportNotEditable_Submitted` | 前提: `report_submitted`（submitted）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE。submitted 状態のレポートへはアップロード不可（ATT-020） | 高 |
| ATT-014 | 統合 | handler | `TestUploadAttachment_ReportNotEditable_Approved` | 前提: `report_approved`（approved）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-015 | 統合 | handler | `TestUploadAttachment_ReportNotEditable_Rejected` | 前提: `report_rejected`（rejected）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-016 | 統合 | handler | `TestUploadAttachment_ReportNotEditable_Paid` | 前提: `report_paid`（paid）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE | 高 |

### 1-5. リソース不存在

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-017 | 統合 | handler | `TestUploadAttachment_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member、ファイル: `file_valid_jpeg` | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-018 | 統合 | handler | `TestUploadAttachment_ItemNotFound` | 前提: `report_draft` は存在するが、存在しない明細ID、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-019 | 統合 | handler | `TestUploadAttachment_ItemBelongsToDifferentReport` | 前提: `report_draft` と別の `report_draft_empty` に属する明細IDを組み合わせたURL、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 404 RESOURCE_NOT_FOUND。明細がURLのレポートに属しない（authz.md §7.3 ステップ4） | 中 |

---

## 2. listAttachments（GET /api/reports/{id}/items/{itemId}/attachments）

### 2-1. 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-020 | 統合 | handler | `TestListAttachments_Success_Owner` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data` 配列に `att_draft_jpeg` の情報を含む。各要素に `id`, `item_id`, `file_name`, `file_size`, `mime_type`, `created_at` を含み、`download_url` を含まない | 高 |
| ATT-021 | 統合 | handler | `TestListAttachments_Success_NoAttachments` | 前提: `report_draft_empty`（明細も添付も0件）、認証: Test Member（所有者） | 200 OK。`data` が空配列 `[]` | 中 |
| ATT-022 | 統合 | handler | `TestListAttachments_Success_Admin` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Admin（所有者でない）| 200 OK。Admin は同一テナントの全レポートの添付を閲覧可能（files.md §4.5） | 高 |
| ATT-023 | 統合 | handler | `TestListAttachments_Success_Accounting` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Accounting（所有者でない） | 200 OK。Accounting は同一テナントの全レポートの添付を閲覧可能 | 高 |
| ATT-024 | 統合 | handler | `TestListAttachments_Success_Approver_SubmittedReport` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Approver（所有者でない） | 200 OK。Approver は submitted レポートの添付を閲覧可能 | 高 |
| ATT-025 | 統合 | handler | `TestListAttachments_NoDownloadUrl` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。レスポンス body の各要素に `download_url` フィールドが含まれない（files.md §5） | 高 |

### 2-2. RBAC 異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-026 | 統合 | handler | `TestListAttachments_Unauthorized` | 認証: なし | 401 UNAUTHORIZED | 高 |
| ATT-027 | 統合 | handler | `TestListAttachments_Forbidden_Member_OtherOwner` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない、かつレポートは draft なので Approver も閲覧不可） | 403 FORBIDDEN。Member 以外のロールであっても、draft レポートは所有者のみ閲覧可能 | 高 |

**注記**: Approver が draft レポートの添付を閲覧できないのは、files.md §4.5 の閲覧権限ルールで Approver の閲覧範囲が「submitted レポート + 自分が承認/却下したレポート + 自分のレポート」に限定されているため。

### 2-3. リソース不存在

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-028 | 統合 | handler | `TestListAttachments_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-029 | 統合 | handler | `TestListAttachments_ItemNotFound` | 前提: `report_draft` は存在するが、存在しない明細ID、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 中 |

---

## 3. getAttachmentDownload（GET /api/reports/{id}/items/{itemId}/attachments/{attId}）

### 3-1. 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-030 | 統合 | handler | `TestGetAttachmentDownload_Success_Owner` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data.download_url`（S3署名付きURL）、`data.file_name`（元ファイル名）、`data.mime_type`（image/jpeg）、`data.file_size`、`data.expires_at` を含む | 高 |
| ATT-031 | 統合 | handler | `TestGetAttachmentDownload_Success_Admin` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Admin（所有者でない） | 200 OK。Admin は同一テナントの全レポートの添付をダウンロード可能 | 高 |
| ATT-032 | 統合 | handler | `TestGetAttachmentDownload_Success_Accounting` | 前提: `report_approved` + `att_approved_png`、認証: Test Accounting（所有者でない） | 200 OK。Accounting は同一テナントの全レポートの添付をダウンロード可能 | 高 |
| ATT-033 | 統合 | handler | `TestGetAttachmentDownload_Success_Approver_SubmittedReport` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Approver（所有者でない） | 200 OK。Approver は submitted レポートの添付をダウンロード可能 | 高 |
| ATT-034 | 統合 | handler | `TestGetAttachmentDownload_ExpiresAt_15min` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data.expires_at` が現在時刻から15分後であること（files.md §4.4、ATT-012） | 高 |

### 3-2. 署名付きURL認可チェック（ATT-011）

下記テストは「URL 発行前の認可チェック」を明示的に検証する。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-035 | 統合 | handler | `TestGetAttachmentDownload_Unauthorized_NoToken` | 認証: なし（Authorization ヘッダーなし） | 401 UNAUTHORIZED。JWT 検証失敗時は署名付きURL を発行しない（files.md §4.5 チェック1） | 高 |
| ATT-036 | 統合 | handler | `TestGetAttachmentDownload_Forbidden_Member_OtherOwner` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない、draft レポートは閲覧不可ロール） | 403 FORBIDDEN。閲覧権限がない場合は署名付きURL を発行しない（files.md §4.5 チェック3） | 高 |
| ATT-037 | 統合 | handler | `TestGetAttachmentDownload_Forbidden_Member_NotOwner` | 前提: `report_submitted`（所有者: Test Member）+ `att_submitted_pdf`、認証: 別の Member ユーザー（同一テナント・所有者でない） | 403 FORBIDDEN。Member は自分が作成したレポートの添付のみ閲覧可能（files.md §4.5） | 高 |
| ATT-038 | 統合 | handler | `TestGetAttachmentDownload_AuthzCheckedBeforeUrlIssue` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない） | 403 FORBIDDEN が返り、S3 署名付きURL生成処理（`s3.PresignGetObject`相当）が呼び出されないこと。モックで検証 | 高 |

### 3-3. RBAC 異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-039 | 統合 | handler | `TestGetAttachmentDownload_Approver_DraftReport_Forbidden` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（別ユーザー） | 403 FORBIDDEN。Approver の閲覧範囲は submitted レポートのみ（自分のレポートは除く） | 高 |

### 3-4. リソース不存在

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-040 | 統合 | handler | `TestGetAttachmentDownload_AttachmentNotFound` | 前提: `report_draft`、存在しない attachment_id、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-041 | 統合 | handler | `TestGetAttachmentDownload_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member | 404 RESOURCE_NOT_FOUND | 中 |

---

## 4. deleteAttachment（DELETE /api/reports/{id}/items/{itemId}/attachments/{attId}）

### 4-1. 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-042 | 統合 | handler | `TestDeleteAttachment_Success` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 204 No Content。`attachments` テーブルの対象レコードに `deleted_at` が設定される（論理削除）。S3 オブジェクトは即時削除されない（files.md §6.1） | 高 |
| ATT-043 | 統合 | handler | `TestDeleteAttachment_Success_SoftDelete_S3NotDeleted` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 204 No Content。削除後も S3 オブジェクトが存在すること（物理削除はバッチ処理） | 高 |

### 4-2. RBAC 異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-044 | 統合 | handler | `TestDeleteAttachment_Unauthorized` | 認証: なし | 401 UNAUTHORIZED | 高 |
| ATT-045 | 統合 | handler | `TestDeleteAttachment_Forbidden_NotOwner_Approver` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない） | 403 FORBIDDEN。所有者でない場合は削除不可（authz.md §6.5、RBC-010） | 高 |
| ATT-046 | 統合 | handler | `TestDeleteAttachment_Forbidden_NotOwner_Admin` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Admin（所有者でない） | 403 FORBIDDEN。Admin であっても他者のレポートの添付を削除不可（authz.md §7.2、RBC-014） | 高 |
| ATT-047 | 統合 | handler | `TestDeleteAttachment_Forbidden_NotOwner_Accounting` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Accounting（所有者でない） | 403 FORBIDDEN | 中 |

### 4-3. レポート状態異常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-048 | 統合 | handler | `TestDeleteAttachment_ReportNotEditable_Submitted` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Member（所有者） | 422 REPORT_NOT_EDITABLE。submitted 状態のレポートは編集不可（ATT-020） | 高 |
| ATT-049 | 統合 | handler | `TestDeleteAttachment_ReportNotEditable_Approved` | 前提: `report_approved` + `att_approved_png`、認証: Test Member（所有者） | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-050 | 統合 | handler | `TestDeleteAttachment_ReportNotEditable_Paid` | 前提: `report_paid`（paid）、認証: Test Member（所有者）、存在する添付ID | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-051 | 統合 | handler | `TestDeleteAttachment_ReportNotEditable_Rejected` | 前提: `report_rejected`（rejected）、認証: Test Member（所有者）、存在する添付ID | 422 REPORT_NOT_EDITABLE。rejected は終端状態であり編集不可（ATT-020） | 高 |

### 4-4. リソース不存在

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|--------------|-----------------|---------|--------|
| ATT-052 | 統合 | handler | `TestDeleteAttachment_NotFound` | 前提: `report_draft`、存在しない attachment_id、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-053 | 統合 | handler | `TestDeleteAttachment_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member | 404 RESOURCE_NOT_FOUND | 中 |
| ATT-054 | 統合 | handler | `TestDeleteAttachment_AlreadyDeleted` | 前提: `report_draft` + 論理削除済みの `att_draft_jpeg`（`deleted_at` が設定済み）、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND。削除済みリソースは存在しないものとして扱う | 中 |

---

## 5. テストID一覧サマリー

| テストID | エンドポイント | 分類 | 優先度 |
|---------|------------|------|--------|
| ATT-001 | uploadAttachment | 正常系（JPEG） | 高 |
| ATT-002 | uploadAttachment | 正常系（PNG） | 高 |
| ATT-003 | uploadAttachment | 正常系（PDF） | 高 |
| ATT-004 | uploadAttachment | 正常系（境界値5MB） | 高 |
| ATT-005 | uploadAttachment | 正常系（Approver所有者） | 中 |
| ATT-006 | uploadAttachment | ファイルサイズ超過 | 高 |
| ATT-007 | uploadAttachment | MIME不正（GIF） | 高 |
| ATT-008 | uploadAttachment | MIME偽装（マジックバイト不一致） | 高 |
| ATT-009 | uploadAttachment | fileパートなし | 高 |
| ATT-010 | uploadAttachment | Content-Typeなし | 中 |
| ATT-011 | uploadAttachment | RBAC（未認証） | 高 |
| ATT-012 | uploadAttachment | RBAC（非所有者） | 高 |
| ATT-013 | uploadAttachment | 状態（submitted） | 高 |
| ATT-014 | uploadAttachment | 状態（approved） | 高 |
| ATT-015 | uploadAttachment | 状態（rejected） | 高 |
| ATT-016 | uploadAttachment | 状態（paid） | 高 |
| ATT-017 | uploadAttachment | レポート不存在 | 高 |
| ATT-018 | uploadAttachment | 明細不存在 | 高 |
| ATT-019 | uploadAttachment | 明細が別レポートに属する | 中 |
| ATT-020 | listAttachments | 正常系（所有者） | 高 |
| ATT-021 | listAttachments | 正常系（添付なし） | 中 |
| ATT-022 | listAttachments | 正常系（Admin） | 高 |
| ATT-023 | listAttachments | 正常系（Accounting） | 高 |
| ATT-024 | listAttachments | 正常系（Approver, submitted） | 高 |
| ATT-025 | listAttachments | 正常系（download_urlなし） | 高 |
| ATT-026 | listAttachments | RBAC（未認証） | 高 |
| ATT-027 | listAttachments | RBAC（Approver+draft） | 高 |
| ATT-028 | listAttachments | レポート不存在 | 高 |
| ATT-029 | listAttachments | 明細不存在 | 中 |
| ATT-030 | getAttachmentDownload | 正常系（所有者） | 高 |
| ATT-031 | getAttachmentDownload | 正常系（Admin） | 高 |
| ATT-032 | getAttachmentDownload | 正常系（Accounting） | 高 |
| ATT-033 | getAttachmentDownload | 正常系（Approver, submitted） | 高 |
| ATT-034 | getAttachmentDownload | 正常系（有効期限15分） | 高 |
| ATT-035 | getAttachmentDownload | 署名付きURL認可（未認証） | 高 |
| ATT-036 | getAttachmentDownload | 署名付きURL認可（Approver+draft） | 高 |
| ATT-037 | getAttachmentDownload | 署名付きURL認可（Member非所有者） | 高 |
| ATT-038 | getAttachmentDownload | 署名付きURL発行前認可チェック | 高 |
| ATT-039 | getAttachmentDownload | RBAC（Approver+draft） | 高 |
| ATT-040 | getAttachmentDownload | 添付不存在 | 高 |
| ATT-041 | getAttachmentDownload | レポート不存在 | 中 |
| ATT-042 | deleteAttachment | 正常系（論理削除） | 高 |
| ATT-043 | deleteAttachment | 正常系（S3即時削除なし） | 高 |
| ATT-044 | deleteAttachment | RBAC（未認証） | 高 |
| ATT-045 | deleteAttachment | RBAC（Approver非所有者） | 高 |
| ATT-046 | deleteAttachment | RBAC（Admin非所有者） | 高 |
| ATT-047 | deleteAttachment | RBAC（Accounting非所有者） | 中 |
| ATT-048 | deleteAttachment | 状態（submitted） | 高 |
| ATT-049 | deleteAttachment | 状態（approved） | 高 |
| ATT-050 | deleteAttachment | 状態（paid） | 高 |
| ATT-051 | deleteAttachment | 状態（rejected） | 高 |
| ATT-052 | deleteAttachment | 添付不存在 | 高 |
| ATT-053 | deleteAttachment | レポート不存在 | 中 |
| ATT-054 | deleteAttachment | 削除済み添付への再削除 | 中 |

**合計: 54件**（優先度「高」: 44件、優先度「中」: 10件）

---

## 6. 実装ガイド（attachment_handler_test.go 向け）

### 6.1 テストセットアップ

```go
// TestMain でフィクスチャを投入する
func TestMain(m *testing.M) {
    db = setupTestDB()     // テスト用PostgreSQL接続
    s3 = setupMinIO()      // MinIO接続（ローカルS3互換）
    truncateAllTables(db)  // 全テーブルをTRUNCATE
    insertFixtures(db)     // 標準フィクスチャ + 添付フィクスチャを挿入
    os.Exit(m.Run())
}
```

### 6.2 テストファイルの生成

テスト用バイナリファイルはヘルパー関数で生成する。実際のファイルをリポジトリに含めない。

```go
// 正常なJPEGファイル（マジックバイト: FF D8 FF）
func makeJPEGFile(size int) []byte {
    buf := make([]byte, size)
    copy(buf, []byte{0xFF, 0xD8, 0xFF, 0xE0}) // JPEGマジックバイト
    return buf
}

// MIME偽装ファイル（Content-Typeはimage/jpeg、実体はGIF）
func makeSpoofedFile() []byte {
    buf := make([]byte, 1024)
    copy(buf, []byte{0x47, 0x49, 0x46, 0x38}) // GIFマジックバイト
    return buf
}
```

### 6.3 multipart/form-data リクエストの構築

```go
func buildMultipartRequest(t *testing.T, fieldName, fileName string, content []byte, contentType string) (*http.Request, error) {
    var body bytes.Buffer
    writer := multipart.NewWriter(&body)
    part, err := writer.CreateFormFile(fieldName, fileName)
    // part に content を書き込む
    // ...
    return http.NewRequest(http.MethodPost, url, &body)
}
```

### 6.4 ATT-038（署名付きURL発行前認可チェック）の検証方法

S3クライアントをモックし、認可拒否の場合にS3署名付きURL生成処理が呼ばれないことを検証する。

```go
type mockS3Client struct {
    presignCalled bool
}

func (m *mockS3Client) PresignGetObject(ctx context.Context, key string) (string, error) {
    m.presignCalled = true
    return "https://mock-url", nil
}

// テスト内でモックを注入し、403が返った後に presignCalled == false を確認する
```

### 6.5 論理削除の検証（ATT-042, ATT-043）

```go
// 削除後にDBを直接クエリして deleted_at が設定されていることを確認
var deletedAt sql.NullTime
err := db.QueryRow(
    "SELECT deleted_at FROM attachments WHERE attachment_id = $1",
    attachmentID,
).Scan(&deletedAt)
assert.True(t, deletedAt.Valid)   // deleted_at が NULL でないこと

// S3オブジェクトがまだ存在することを確認
_, err = s3Client.HeadObject(ctx, &s3.HeadObjectInput{
    Bucket: aws.String(testBucket),
    Key:    aws.String(s3Key),
})
assert.NoError(t, err) // HeadObject が成功する = オブジェクトが存在する
```

### 6.6 フィクスチャの注意事項

- テナントAのフィクスチャのみを本ファイルのテストで使用する
- テナントBのフィクスチャを使ったクロステナントテストは `cross-cutting.md` のテストケースで実装する
- `att_draft_jpeg` 等の添付フィクスチャは対応するS3オブジェクト（MinIO上）も投入すること
- S3オブジェクトキーのフォーマット: `{tenant_id}/{report_id}/{attachment_id}`（files.md §2.2）

# 添付ファイルテストケース一覧

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 添付ファイルエンドポイント（uploadAttachment, listAttachments, getAttachmentDownload, getAttachmentPreview, deleteAttachment）のテストケースを定義する |
| 正本情報 | ATT-F01〜F04 に対応するテストケース一覧 |
| 扱わない内容 | テナント横断テスト（→ cross-cutting.md）、S3 インフラ詳細 |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/reports/{id}/items/{itemId}/attachments/*`, `50_detail_design/files.md`, `10_requirements/requirements.md#ATT-F*` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

## 概要

本書は添付ファイル（Attachment）に関する5エンドポイントのテストケースを定義する。

| 項目 | 内容 |
|------|------|
| テストID プレフィックス | `ATT-` |
| 対象エンドポイント | uploadAttachment, listAttachments, getAttachmentDownload, getAttachmentPreview, deleteAttachment |
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

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-001 | 統合 | handler | 正常系 | ATT-F01, ATT-002 | openapi.yaml#uploadAttachment, files.md#3, db_schema.md#attachments | `TestUploadAttachment_Success_JPEG` | 前提: `report_draft`（draft）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 201 Created。レスポンス body に `data.id`, `data.item_id`, `data.file_name`, `data.file_size`（1024）, `data.mime_type`（image/jpeg）, `data.created_at` を含む。`attachments` テーブルにレコードが挿入される | 高 |
| ATT-002 | 統合 | handler | 正常系 | ATT-F01, ATT-002 | openapi.yaml#uploadAttachment, files.md#3, db_schema.md#attachments | `TestUploadAttachment_Success_PNG` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_valid_png` | 201 Created。`data.mime_type` が `image/png` | 高 |
| ATT-003 | 統合 | handler | 正常系 | ATT-F01, ATT-002 | openapi.yaml#uploadAttachment, files.md#3, db_schema.md#attachments | `TestUploadAttachment_Success_PDF` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_valid_pdf` | 201 Created。`data.mime_type` が `application/pdf` | 高 |
| ATT-004 | 統合 | handler | 境界値 | ATT-F01, ATT-003 | openapi.yaml#uploadAttachment, files.md#3 | `TestUploadAttachment_Success_ExactlyMaxSize` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_exactly_5mb`（5,242,880 B） | 201 Created。サイズ境界値（ちょうど5MB）は許可される | 高 |
| ATT-005 | 統合 | handler | 認可 | ATT-F01, RBC-010 | openapi.yaml#uploadAttachment, authz.md#4.1 | `TestUploadAttachment_Success_Approver` | 前提: `report_draft`（Test Approver が作成したレポート）、認証: Test Approver（所有者）、ファイル: `file_valid_jpeg` | 201 Created。Approver が所有者の場合はアップロード可能 | 中 |

### 1-2. ファイルバリデーション異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-006 | 統合 | handler | 境界値 | ATT-003 | openapi.yaml#uploadAttachment, files.md#3 | `TestUploadAttachment_FileTooLarge` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_too_large`（5,242,881 B） | 413 FILE_TOO_LARGE。`attachments` テーブルにレコードが挿入されない | 高 |
| ATT-007 | 統合 | handler | 異常系 | ATT-002, ATT-013 | openapi.yaml#uploadAttachment, files.md#3 | `TestUploadAttachment_InvalidMimeType_GIF` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_invalid_type_gif`（Content-Type: image/gif） | 422 INVALID_FILE_TYPE。GIF は許可リスト外 | 高 |
| ATT-008 | 統合 | handler | セキュリティ | ATT-013 | openapi.yaml#uploadAttachment, files.md#3.2 | `TestUploadAttachment_SpoofedMimeType` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_spoofed_jpeg`（Content-Type: image/jpeg と宣言、マジックバイトはGIF） | 422 INVALID_FILE_TYPE。Content-Type とマジックバイトの不一致で拒否 | 高 |
| ATT-009 | 統合 | handler | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | `TestUploadAttachment_MissingFilePart` | 前提: `report_draft`、認証: Test Member（所有者）、リクエストボディ: multipart/form-data だが `file` パートなし | 400 INVALID_REQUEST（BAD_REQUEST） | 高 |
| ATT-010 | 統合 | handler | 異常系 | ATT-013 | openapi.yaml#uploadAttachment, files.md#3.2 | `TestUploadAttachment_NoContentType` | 前提: `report_draft`、認証: Test Member（所有者）、ファイル: `file_no_content_type`（Content-Type ヘッダーなし） | 422 INVALID_FILE_TYPE | 中 |

### 1-3. RBAC 異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-011 | 統合 | handler | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | `TestUploadAttachment_Unauthorized` | 認証: なし（Authorization ヘッダーなし）、ファイル: `file_valid_jpeg` | 401 UNAUTHORIZED | 高 |
| ATT-012 | 統合 | handler | 認可 | ATT-F01, RBC-010 | openapi.yaml#uploadAttachment, authz.md#7.2 | `TestUploadAttachment_Forbidden_NotOwner` | 前提: `report_draft`（所有者: Test Member）、認証: Test Approver（別ユーザー、所有者でない）、ファイル: `file_valid_jpeg` | 403 FORBIDDEN。同一テナント内の非所有者は拒否（authz.md §7.2） | 高 |

### 1-4. レポート状態異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-013 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | `TestUploadAttachment_ReportNotEditable_Submitted` | 前提: `report_submitted`（submitted）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE。submitted 状態のレポートへはアップロード不可（ATT-020） | 高 |
| ATT-014 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | `TestUploadAttachment_ReportNotEditable_Approved` | 前提: `report_approved`（approved）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-015 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | `TestUploadAttachment_ReportNotEditable_Rejected` | 前提: `report_rejected`（rejected）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-016 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | `TestUploadAttachment_ReportNotEditable_Paid` | 前提: `report_paid`（paid）、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 422 REPORT_NOT_EDITABLE | 高 |

### 1-5. リソース不存在

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-017 | 統合 | handler | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | `TestUploadAttachment_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member、ファイル: `file_valid_jpeg` | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-018 | 統合 | handler | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | `TestUploadAttachment_ItemNotFound` | 前提: `report_draft` は存在するが、存在しない明細ID、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-019 | 統合 | handler | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment, authz.md#7.3 | `TestUploadAttachment_ItemBelongsToDifferentReport` | 前提: `report_draft` と別の `report_draft_empty` に属する明細IDを組み合わせたURL、認証: Test Member（所有者）、ファイル: `file_valid_jpeg` | 404 RESOURCE_NOT_FOUND。明細がURLのレポートに属しない（authz.md §7.3 ステップ4） | 中 |

---

## 2. listAttachments（GET /api/reports/{id}/items/{itemId}/attachments）

### 2-1. 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-020 | 統合 | handler | 正常系 | ATT-F02 | openapi.yaml#listAttachments | `TestListAttachments_Success_Owner` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data` 配列に `att_draft_jpeg` の情報を含む。各要素に `id`, `item_id`, `file_name`, `file_size`, `mime_type`, `created_at` を含み、`url` を含まない | 高 |
| ATT-021 | 統合 | handler | 正常系 | ATT-F02 | openapi.yaml#listAttachments | `TestListAttachments_Success_NoAttachments` | 前提: `report_draft_empty`（明細も添付も0件）、認証: Test Member（所有者） | 200 OK。`data` が空配列 `[]` | 中 |
| ATT-022 | 統合 | handler | 認可 | ATT-F02, RBC-013 | openapi.yaml#listAttachments, authz.md#4.4 | `TestListAttachments_Success_Admin` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Admin（所有者でない）| 200 OK。Admin は同一テナントの全レポートの添付を閲覧可能（files.md §4.5） | 高 |
| ATT-023 | 統合 | handler | 認可 | ATT-F02 | openapi.yaml#listAttachments, authz.md#4.3 | `TestListAttachments_Success_Accounting` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Accounting（所有者でない） | 200 OK。Accounting は同一テナントの全レポートの添付を閲覧可能 | 高 |
| ATT-024 | 統合 | handler | 認可 | ATT-F02, RBC-011 | openapi.yaml#listAttachments, authz.md#4.2 | `TestListAttachments_Success_Approver_SubmittedReport` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Approver（所有者でない） | 200 OK。Approver は submitted レポートの添付を閲覧可能 | 高 |
| ATT-025 | 統合 | handler | 正常系 | ATT-F02 | openapi.yaml#listAttachments, files.md#5 | `TestListAttachments_NoUrl` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。レスポンス body の各要素に `url` フィールドが含まれない（files.md §5） | 高 |

### 2-2. RBAC 異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-026 | 統合 | handler | 異常系 | ATT-F02 | openapi.yaml#listAttachments | `TestListAttachments_Unauthorized` | 認証: なし | 401 UNAUTHORIZED | 高 |
| ATT-027 | 統合 | handler | 認可 | ATT-F02, RBC-011 | openapi.yaml#listAttachments, files.md#4.5 | `TestListAttachments_Forbidden_Member_OtherOwner` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない、かつレポートは draft なので Approver も閲覧不可） | 403 FORBIDDEN。Member 以外のロールであっても、draft レポートは所有者のみ閲覧可能 | 高 |

**注記**: Approver が draft レポートの添付を閲覧できないのは、files.md §4.5 の閲覧権限ルールで Approver の閲覧範囲が「submitted レポート + 自分が承認/却下したレポート + 自分のレポート」に限定されているため。

### 2-3. リソース不存在

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-028 | 統合 | handler | 異常系 | ATT-F02 | openapi.yaml#listAttachments | `TestListAttachments_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-029 | 統合 | handler | 異常系 | ATT-F02 | openapi.yaml#listAttachments | `TestListAttachments_ItemNotFound` | 前提: `report_draft` は存在するが、存在しない明細ID、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 中 |

---

## 3. getAttachmentDownload（GET /api/reports/{id}/items/{itemId}/attachments/{attId}/download） / getAttachmentPreview（GET .../preview）

### 3-1. 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-030 | 統合 | handler | 正常系 | ATT-F03, ATT-010 | openapi.yaml#getAttachmentDownload, files.md#署名付きURL | `TestGetAttachmentDownload_Success_Owner` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data.url`（S3署名付きURL、Content-Disposition: attachment）、`data.file_name`（元ファイル名）、`data.mime_type`（image/jpeg）、`data.file_size`、`data.expires_at` を含む。レスポンススキーマは `AttachmentAccess` | 高 |
| ATT-031 | 統合 | handler | 認可 | ATT-F03, RBC-013 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Success_Admin` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Admin（所有者でない） | 200 OK。Admin は同一テナントの全レポートの添付をダウンロード可能 | 高 |
| ATT-032 | 統合 | handler | 認可 | ATT-F03 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Success_Accounting` | 前提: `report_approved` + `att_approved_png`、認証: Test Accounting（所有者でない） | 200 OK。Accounting は同一テナントの全レポートの添付をダウンロード可能 | 高 |
| ATT-033 | 統合 | handler | 認可 | ATT-F03, RBC-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Success_Approver_SubmittedReport` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Approver（所有者でない） | 200 OK。Approver は submitted レポートの添付をダウンロード可能 | 高 |
| ATT-034 | 統合 | handler | 正常系 | ATT-012 | openapi.yaml#getAttachmentDownload, files.md#4.4 | `TestGetAttachmentDownload_ExpiresAt_15min` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data.expires_at` が現在時刻から15分後であること（files.md §4.4、ATT-012） | 高 |

### 3-2. 署名付きURL認可チェック（ATT-011）

下記テストは「URL 発行前の認可チェック」を明示的に検証する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-035 | 統合 | handler | セキュリティ | ATT-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Unauthorized_NoToken` | 認証: なし（Authorization ヘッダーなし） | 401 UNAUTHORIZED。JWT 検証失敗時は署名付きURL を発行しない（files.md §4.5 チェック1） | 高 |
| ATT-036 | 統合 | handler | 認可 | ATT-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Forbidden_Member_OtherOwner` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない、draft レポートは閲覧不可ロール） | 403 FORBIDDEN。閲覧権限がない場合は署名付きURL を発行しない（files.md §4.5 チェック3） | 高 |
| ATT-037 | 統合 | handler | 認可 | ATT-011, RBC-010 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Forbidden_Member_NotOwner` | 前提: `report_submitted`（所有者: Test Member）+ `att_submitted_pdf`、認証: 別の Member ユーザー（同一テナント・所有者でない） | 403 FORBIDDEN。Member は自分が作成したレポートの添付のみ閲覧可能（files.md §4.5） | 高 |
| ATT-038 | 統合 | handler | セキュリティ | ATT-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_AuthzCheckedBeforeUrlIssue` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない） | 403 FORBIDDEN が返り、S3 署名付きURL生成処理（`s3.PresignGetObject`相当）が呼び出されないこと。モックで検証 | 高 |

### 3-3. RBAC 異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-039 | 統合 | handler | 認可 | ATT-F03, RBC-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | `TestGetAttachmentDownload_Approver_DraftReport_Forbidden` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（別ユーザー） | 403 FORBIDDEN。Approver の閲覧範囲は submitted レポートのみ（自分のレポートは除く） | 高 |

### 3-4. リソース不存在

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-040 | 統合 | handler | 異常系 | ATT-F03 | openapi.yaml#getAttachmentDownload | `TestGetAttachmentDownload_AttachmentNotFound` | 前提: `report_draft`、存在しない attachment_id、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-041 | 統合 | handler | 異常系 | ATT-F03 | openapi.yaml#getAttachmentDownload | `TestGetAttachmentDownload_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member | 404 RESOURCE_NOT_FOUND | 中 |

### 3-5. getAttachmentPreview（プレビュー用テストケース）

下記テストケースはプレビューエンドポイント（`GET .../attachments/{attId}/preview`）の認可・正常系を検証する。認可ロジックは `getAttachmentDownload` と同一のため、4 ロール x 正常系とテナント越境ケースを検証する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-055 | 統合 | handler | 正常系 | ATT-F03, ATT-010 | openapi.yaml#getAttachmentPreview, files.md#署名付きURL | `TestGetAttachmentPreview_Success_Owner` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 200 OK。`data.url`（S3署名付きURL、Content-Disposition: inline）、`data.file_name`、`data.mime_type`（image/jpeg）、`data.file_size`、`data.expires_at` を含む。レスポンススキーマは `AttachmentAccess` | 高 |
| ATT-056 | 統合 | handler | 認可 | ATT-F03, RBC-013 | openapi.yaml#getAttachmentPreview, files.md#4.5 | `TestGetAttachmentPreview_Success_Admin` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Admin（所有者でない） | 200 OK。Admin は同一テナントの全レポートの添付をプレビュー可能 | 高 |
| ATT-057 | 統合 | handler | 認可 | ATT-F03 | openapi.yaml#getAttachmentPreview, files.md#4.5 | `TestGetAttachmentPreview_Success_Accounting` | 前提: `report_approved` + `att_approved_png`、認証: Test Accounting（所有者でない） | 200 OK。Accounting は同一テナントの全レポートの添付をプレビュー可能 | 高 |
| ATT-058 | 統合 | handler | 認可 | ATT-F03, RBC-011 | openapi.yaml#getAttachmentPreview, files.md#4.5 | `TestGetAttachmentPreview_Success_Approver_SubmittedReport` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Approver（所有者でない） | 200 OK。Approver は submitted レポートの添付をプレビュー可能 | 高 |
| ATT-059 | 統合 | handler | 認可 | ATT-011 | openapi.yaml#getAttachmentPreview, files.md#4.5 | `TestGetAttachmentPreview_Forbidden_Member_OtherOwner` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない、draft レポートは閲覧不可ロール） | 403 FORBIDDEN。閲覧権限がない場合はプレビュー用署名付き URL を発行しない | 高 |
| ATT-060 | 統合 | handler | 異常系 | ATT-F03 | openapi.yaml#getAttachmentPreview | `TestGetAttachmentPreview_AttachmentNotFound` | 前提: `report_draft`、存在しない attachment_id、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 高 |

---

## 4. deleteAttachment（DELETE /api/reports/{id}/items/{itemId}/attachments/{attId}）

### 4-1. 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-042 | 統合 | handler | 正常系 | ATT-F04, NFR-DATA-001 | openapi.yaml#deleteAttachment, files.md#6.1 | `TestDeleteAttachment_Success` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 204 No Content。`attachments` テーブルの対象レコードに `deleted_at` が設定される（論理削除）。S3 オブジェクトは即時削除されない（files.md §6.1） | 高 |
| ATT-043 | 統合 | handler | 正常系 | ATT-F04 | openapi.yaml#deleteAttachment, files.md#6.1 | `TestDeleteAttachment_Success_SoftDelete_S3NotDeleted` | 前提: `report_draft` + `att_draft_jpeg`、認証: Test Member（所有者） | 204 No Content。削除後も S3 オブジェクトが存在すること（物理削除はバッチ処理） | 高 |

### 4-2. RBAC 異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-044 | 統合 | handler | 異常系 | ATT-F04 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_Unauthorized` | 認証: なし | 401 UNAUTHORIZED | 高 |
| ATT-045 | 統合 | handler | 認可 | ATT-F04, RBC-010 | openapi.yaml#deleteAttachment, authz.md#6.5 | `TestDeleteAttachment_Forbidden_NotOwner_Approver` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Approver（所有者でない） | 403 FORBIDDEN。所有者でない場合は削除不可（authz.md §6.5、RBC-010） | 高 |
| ATT-046 | 統合 | handler | 認可 | ATT-F04, RBC-014 | openapi.yaml#deleteAttachment, authz.md#7.2 | `TestDeleteAttachment_Forbidden_NotOwner_Admin` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Admin（所有者でない） | 403 FORBIDDEN。Admin であっても他者のレポートの添付を削除不可（authz.md §7.2、RBC-014） | 高 |
| ATT-047 | 統合 | handler | 認可 | ATT-F04, RBC-010 | openapi.yaml#deleteAttachment, authz.md#6.5 | `TestDeleteAttachment_Forbidden_NotOwner_Accounting` | 前提: `report_draft`（所有者: Test Member）+ `att_draft_jpeg`、認証: Test Accounting（所有者でない） | 403 FORBIDDEN | 中 |

### 4-3. レポート状態異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-048 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_ReportNotEditable_Submitted` | 前提: `report_submitted` + `att_submitted_pdf`、認証: Test Member（所有者） | 422 REPORT_NOT_EDITABLE。submitted 状態のレポートは編集不可（ATT-020） | 高 |
| ATT-049 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_ReportNotEditable_Approved` | 前提: `report_approved` + `att_approved_png`、認証: Test Member（所有者） | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-050 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_ReportNotEditable_Paid` | 前提: `report_paid`（paid）、認証: Test Member（所有者）、存在する添付ID | 422 REPORT_NOT_EDITABLE | 高 |
| ATT-051 | 統合 | handler | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_ReportNotEditable_Rejected` | 前提: `report_rejected`（rejected）、認証: Test Member（所有者）、存在する添付ID | 422 REPORT_NOT_EDITABLE。rejected は終端状態であり編集不可（ATT-020） | 高 |

### 4-4. リソース不存在

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 | 優先度 |
|---------|------------|--------|---------|-----------|-----------|--------------|-----------------|---------|--------|
| ATT-052 | 統合 | handler | 異常系 | ATT-F04 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_NotFound` | 前提: `report_draft`、存在しない attachment_id、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND | 高 |
| ATT-053 | 統合 | handler | 異常系 | ATT-F04 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_ReportNotFound` | 前提: 存在しないレポートID、認証: Test Member | 404 RESOURCE_NOT_FOUND | 中 |
| ATT-054 | 統合 | handler | 異常系 | ATT-F04, NFR-DATA-001 | openapi.yaml#deleteAttachment | `TestDeleteAttachment_AlreadyDeleted` | 前提: `report_draft` + 論理削除済みの `att_draft_jpeg`（`deleted_at` が設定済み）、認証: Test Member（所有者） | 404 RESOURCE_NOT_FOUND。削除済みリソースは存在しないものとして扱う | 中 |

---

## 5. テストID一覧サマリー

| テストID | エンドポイント | 分類 | 優先度 |
|---------|------------|------|--------|
| ATT-001 | uploadAttachment | 正常系（JPEG） | 正常系 | ATT-F01, ATT-002 | openapi.yaml#uploadAttachment, files.md#3, db_schema.md#attachments | 高 |
| ATT-002 | uploadAttachment | 正常系（PNG） | 正常系 | ATT-F01, ATT-002 | openapi.yaml#uploadAttachment, files.md#3, db_schema.md#attachments | 高 |
| ATT-003 | uploadAttachment | 正常系（PDF） | 正常系 | ATT-F01, ATT-002 | openapi.yaml#uploadAttachment, files.md#3, db_schema.md#attachments | 高 |
| ATT-004 | uploadAttachment | 正常系（境界値5MB） | 境界値 | ATT-F01, ATT-003 | openapi.yaml#uploadAttachment, files.md#3 | 高 |
| ATT-005 | uploadAttachment | 正常系（Approver所有者） | 認可 | ATT-F01, RBC-010 | openapi.yaml#uploadAttachment, authz.md#4.1 | 中 |
| ATT-006 | uploadAttachment | ファイルサイズ超過 | 境界値 | ATT-003 | openapi.yaml#uploadAttachment, files.md#3 | 高 |
| ATT-007 | uploadAttachment | MIME不正（GIF） | 異常系 | ATT-002, ATT-013 | openapi.yaml#uploadAttachment, files.md#3 | 高 |
| ATT-008 | uploadAttachment | MIME偽装（マジックバイト不一致） | セキュリティ | ATT-013 | openapi.yaml#uploadAttachment, files.md#3.2 | 高 |
| ATT-009 | uploadAttachment | fileパートなし | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | 高 |
| ATT-010 | uploadAttachment | Content-Typeなし | 異常系 | ATT-013 | openapi.yaml#uploadAttachment, files.md#3.2 | 中 |
| ATT-011 | uploadAttachment | RBAC（未認証） | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | 高 |
| ATT-012 | uploadAttachment | RBAC（非所有者） | 認可 | ATT-F01, RBC-010 | openapi.yaml#uploadAttachment, authz.md#7.2 | 高 |
| ATT-013 | uploadAttachment | 状態（submitted） | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | 高 |
| ATT-014 | uploadAttachment | 状態（approved） | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | 高 |
| ATT-015 | uploadAttachment | 状態（rejected） | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | 高 |
| ATT-016 | uploadAttachment | 状態（paid） | 状態遷移 | ATT-020 | openapi.yaml#uploadAttachment | 高 |
| ATT-017 | uploadAttachment | レポート不存在 | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | 高 |
| ATT-018 | uploadAttachment | 明細不存在 | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment | 高 |
| ATT-019 | uploadAttachment | 明細が別レポートに属する | 異常系 | ATT-F01 | openapi.yaml#uploadAttachment, authz.md#7.3 | 中 |
| ATT-020 | listAttachments | 正常系（所有者） | 正常系 | ATT-F02 | openapi.yaml#listAttachments | 高 |
| ATT-021 | listAttachments | 正常系（添付なし） | 正常系 | ATT-F02 | openapi.yaml#listAttachments | 中 |
| ATT-022 | listAttachments | 正常系（Admin） | 認可 | ATT-F02, RBC-013 | openapi.yaml#listAttachments, authz.md#4.4 | 高 |
| ATT-023 | listAttachments | 正常系（Accounting） | 認可 | ATT-F02 | openapi.yaml#listAttachments, authz.md#4.3 | 高 |
| ATT-024 | listAttachments | 正常系（Approver, submitted） | 認可 | ATT-F02, RBC-011 | openapi.yaml#listAttachments, authz.md#4.2 | 高 |
| ATT-025 | listAttachments | 正常系（urlなし） | 正常系 | ATT-F02 | openapi.yaml#listAttachments, files.md#5 | 高 |
| ATT-026 | listAttachments | RBAC（未認証） | 異常系 | ATT-F02 | openapi.yaml#listAttachments | 高 |
| ATT-027 | listAttachments | RBAC（Approver+draft） | 認可 | ATT-F02, RBC-011 | openapi.yaml#listAttachments, files.md#4.5 | 高 |
| ATT-028 | listAttachments | レポート不存在 | 異常系 | ATT-F02 | openapi.yaml#listAttachments | 高 |
| ATT-029 | listAttachments | 明細不存在 | 異常系 | ATT-F02 | openapi.yaml#listAttachments | 中 |
| ATT-030 | getAttachmentDownload | 正常系（所有者） | 正常系 | ATT-F03, ATT-010 | openapi.yaml#getAttachmentDownload, files.md#署名付きURL | 高 |
| ATT-031 | getAttachmentDownload | 正常系（Admin） | 認可 | ATT-F03, RBC-013 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-032 | getAttachmentDownload | 正常系（Accounting） | 認可 | ATT-F03 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-033 | getAttachmentDownload | 正常系（Approver, submitted） | 認可 | ATT-F03, RBC-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-034 | getAttachmentDownload | 正常系（有効期限15分） | 正常系 | ATT-012 | openapi.yaml#getAttachmentDownload, files.md#4.4 | 高 |
| ATT-035 | getAttachmentDownload | 署名付きURL認可（未認証） | セキュリティ | ATT-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-036 | getAttachmentDownload | 署名付きURL認可（Approver+draft） | 認可 | ATT-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-037 | getAttachmentDownload | 署名付きURL認可（Member非所有者） | 認可 | ATT-011, RBC-010 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-038 | getAttachmentDownload | 署名付きURL発行前認可チェック | セキュリティ | ATT-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-039 | getAttachmentDownload | RBAC（Approver+draft） | 認可 | ATT-F03, RBC-011 | openapi.yaml#getAttachmentDownload, files.md#4.5 | 高 |
| ATT-040 | getAttachmentDownload | 添付不存在 | 異常系 | ATT-F03 | openapi.yaml#getAttachmentDownload | 高 |
| ATT-041 | getAttachmentDownload | レポート不存在 | 異常系 | ATT-F03 | openapi.yaml#getAttachmentDownload | 中 |
| ATT-042 | deleteAttachment | 正常系（論理削除） | 正常系 | ATT-F04, NFR-DATA-001 | openapi.yaml#deleteAttachment, files.md#6.1 | 高 |
| ATT-043 | deleteAttachment | 正常系（S3即時削除なし） | 正常系 | ATT-F04 | openapi.yaml#deleteAttachment, files.md#6.1 | 高 |
| ATT-044 | deleteAttachment | RBAC（未認証） | 異常系 | ATT-F04 | openapi.yaml#deleteAttachment | 高 |
| ATT-045 | deleteAttachment | RBAC（Approver非所有者） | 認可 | ATT-F04, RBC-010 | openapi.yaml#deleteAttachment, authz.md#6.5 | 高 |
| ATT-046 | deleteAttachment | RBAC（Admin非所有者） | 認可 | ATT-F04, RBC-014 | openapi.yaml#deleteAttachment, authz.md#7.2 | 高 |
| ATT-047 | deleteAttachment | RBAC（Accounting非所有者） | 認可 | ATT-F04, RBC-010 | openapi.yaml#deleteAttachment, authz.md#6.5 | 中 |
| ATT-048 | deleteAttachment | 状態（submitted） | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | 高 |
| ATT-049 | deleteAttachment | 状態（approved） | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | 高 |
| ATT-050 | deleteAttachment | 状態（paid） | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | 高 |
| ATT-051 | deleteAttachment | 状態（rejected） | 状態遷移 | ATT-020 | openapi.yaml#deleteAttachment | 高 |
| ATT-052 | deleteAttachment | 添付不存在 | 異常系 | ATT-F04 | openapi.yaml#deleteAttachment | 高 |
| ATT-053 | deleteAttachment | レポート不存在 | 異常系 | ATT-F04 | openapi.yaml#deleteAttachment | 中 |
| ATT-054 | deleteAttachment | 削除済み添付への再削除 | 異常系 | ATT-F04, NFR-DATA-001 | openapi.yaml#deleteAttachment | 中 |

**合計: 60件**（優先度「高」: 50件、優先度「中」: 10件）

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

func (m *mockS3Client) PresignGetObject(ctx context.Context, key string, disposition string) (string, error) {
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

---

## 7. FE テストケース

### 対象コンポーネント

| コンポーネント | 配置 | 設計識別子 |
|--------------|------|-----------|
| AttachmentArea | `pages/reports/AttachmentArea.tsx` | `55_ui_component/screens/report-detail.md §AttachmentArea` |
| AttachmentList | `pages/reports/AttachmentList.tsx` | `55_ui_component/screens/report-detail.md §AttachmentList` |
| AttachmentUploader | `pages/reports/AttachmentUploader.tsx` | `55_ui_component/screens/report-detail.md §AttachmentUploader` |

### 対象 Hook

| Hook 名 | 対応 API | 設計識別子 |
|---------|---------|-----------|
| `useAttachments` | `GET /api/reports/:id/items/:itemId/attachments` | `55_ui_component/state-management.md §useAttachments` |
| `useAttachmentDownloadUrl` | `GET /api/reports/:id/items/:itemId/attachments/:attId/download` | `55_ui_component/state-management.md §useAttachmentDownloadUrl` |
| `useAttachmentPreviewUrl` | `GET /api/reports/:id/items/:itemId/attachments/:attId/preview` | `55_ui_component/state-management.md §useAttachmentPreviewUrl` |
| `useUploadAttachment` | `POST /api/reports/:id/items/:itemId/attachments` | `55_ui_component/state-management.md §useUploadAttachment` |
| `useDeleteAttachment` | `DELETE /api/reports/:id/items/:itemId/attachments/:attId` | `55_ui_component/state-management.md §useDeleteAttachment` |

### 参照設計書

- `55_ui_component/screens/report-detail.md` -- AttachmentArea, AttachmentList, AttachmentUploader のコンポーネント定義・Props 型・データフロー・表示条件
- `55_ui_component/state-management.md` -- useAttachments, useAttachmentDownloadUrl, useAttachmentPreviewUrl, useUploadAttachment, useDeleteAttachment の入出力・クエリキー・キャッシュ無効化
- `55_ui_component/common-components.md` -- AppToast（成功/エラー通知）

### 7-1. AttachmentArea

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-001 | 単体 | AttachmentArea | `reportId: string` | 正常系 | ATT-F02 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `renders_attachment_area_with_list_and_uploader` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。useAttachments が添付2件を返す | AttachmentList と AttachmentUploader の両方が描画される |
| ATT-FE-002 | 単体 | AttachmentArea | `itemId: string \| null` | 正常系 | ATT-F02 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `renders_empty_when_itemId_is_null` | Props: `reportId="rpt-1"`, `itemId=null`, `canModify=true` | 明細未保存のため添付一覧を取得しない。AttachmentList は空状態、AttachmentUploader は非表示 |
| ATT-FE-003 | 単体 | AttachmentArea | `canModify: boolean` | 認可 | ATT-020 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `hides_uploader_when_canModify_is_false` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=false`。useAttachments が添付1件を返す | AttachmentList は描画される。AttachmentUploader は非表示。AttachmentList の削除ボタンも非表示 |
| ATT-FE-004 | 単体 | AttachmentArea | `canModify: boolean` | 認可 | ATT-020 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `shows_uploader_when_canModify_is_true` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true` | AttachmentUploader が表示される。AttachmentList の削除ボタンも表示される |
| ATT-FE-005 | 単体 | AttachmentArea | `useAttachments` | 正常系 | ATT-F02 | `55_ui_component/state-management.md §useAttachments` | `fetches_attachments_on_mount` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true` | マウント時に useAttachments が `{ reportId: "rpt-1", itemId: "item-1" }` で呼び出される |
| ATT-FE-006 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `sets_deletingId_during_delete` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ボタンをクリック | 削除処理中に deletingId が設定され、対象添付がグレーアウトされる |

### 7-2. AttachmentList

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-007 | 単体 | AttachmentList | `attachments: Attachment[]` | 正常系 | ATT-F02 | `55_ui_component/screens/report-detail.md §AttachmentList` | `renders_attachment_list_with_file_info` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[{ id: "att-1", file_name: "receipt.jpg", file_size: 245760, mime_type: "image/jpeg" }]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()` | ファイル名「receipt.jpg」とファイルサイズが表示される |
| ATT-FE-008 | 単体 | AttachmentList | `attachments: Attachment[]` | 正常系 | ATT-F02 | `55_ui_component/screens/report-detail.md §AttachmentList` | `renders_empty_list` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()` | 添付ファイルの行要素が描画されない |
| ATT-FE-009 | 単体 | AttachmentList | `attachments: Attachment[]` | 正常系 | ATT-F02 | `55_ui_component/screens/report-detail.md §AttachmentList` | `renders_multiple_attachments` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[att-1, att-2, att-3]`（3件）、`canDelete=true`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()` | 3件の添付ファイルがそれぞれファイル名・ファイルサイズとともに表示される |
| ATT-FE-010 | 単体 | AttachmentList（内部の AttachmentItemRow） | `useAttachmentPreviewUrl` | 正常系 | ATT-F03 | `55_ui_component/screens/report-detail.md §AttachmentList`, `50_detail_design/files.md §4.5` | `opens_preview_via_window_open_on_filename_click` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[{ id: "att-1", file_name: "receipt.jpg", ... }]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()`。`useAttachmentPreviewUrl` をモックし `refetch` が `{ data: { data: { url: "https://s3.example.com/..." } } }` を返すように設定。`window.open` をスパイし `newWindow` モックを返すよう設定。ファイル名「receipt.jpg」をクリック | `window.open('about:blank', '_blank')` がクリック同期で呼ばれ、続いて `useAttachmentPreviewUrl` の `refetch()` が 1 回呼ばれる。取得成功後に `newWindow.location.href` が取得した署名付き URL に差し替えられる |
| ATT-FE-010b | 単体 | AttachmentList（内部の AttachmentItemRow） | `useAttachmentDownloadUrl` | 正常系 | ATT-F03 | `55_ui_component/screens/report-detail.md §AttachmentList`, `50_detail_design/files.md §4.5` | `triggers_download_via_anchor_element_on_download_icon_click` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[{ id: "att-1", file_name: "receipt.jpg", ... }]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()`。`useAttachmentDownloadUrl` をモックし `refetch` が `{ data: { data: { url: "https://s3.example.com/...", file_name: "receipt.jpg" } } }` を返すように設定。`document.createElement` / `appendChild` / `removeChild` をスパイ、生成される `<a>` 要素の `click` メソッドもスパイする。↓ アイコンをクリック | `useAttachmentDownloadUrl` の `refetch()` が 1 回呼ばれる。成功後に `document.createElement('a')` で `<a>` 要素が生成され、`href` に署名付き URL、`download` 属性にファイル名 `"receipt.jpg"` が設定される。`document.body.appendChild` → `link.click()` → `document.body.removeChild` の順でスパイが呼ばれる。`window.open` は呼ばれない |
| ATT-FE-011 | 単体 | AttachmentList | `canDelete: boolean` | 認可 | ATT-020 | `55_ui_component/screens/report-detail.md §AttachmentList` | `shows_delete_button_when_canDelete_true` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[att-1]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()` | 各添付行に削除ボタンが表示される |
| ATT-FE-012 | 単体 | AttachmentList | `canDelete: boolean` | 認可 | ATT-020 | `55_ui_component/screens/report-detail.md §AttachmentList` | `hides_delete_button_when_canDelete_false` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[att-1]`, `canDelete=false`, `onDelete=jest.fn()`, `deletingId=null`, `onError=jest.fn()` | 削除ボタンが表示されない。ファイル名のダウンロードリンクは表示される |
| ATT-FE-013 | 単体 | AttachmentList | `onDelete: (attachmentId: string) => void` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentList` | `calls_onDelete_on_delete_button_click` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[{ id: "att-1", ... }]`, `canDelete=true`, `onDelete=mockFn`, `deletingId=null`, `onError=jest.fn()`。削除ボタンをクリック | `onDelete` が `"att-1"` を引数として呼び出される |
| ATT-FE-014 | 単体 | AttachmentList | `deletingId: string \| null` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentList` | `greys_out_deleting_attachment` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[{ id: "att-1", ... }, { id: "att-2", ... }]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId="att-1"`, `onError=jest.fn()` | `att-1` の行がグレーアウト表示される。`att-2` の行は通常表示 |
| ATT-FE-015 | 単体 | AttachmentList | `deletingId: string \| null` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentList` | `disables_delete_button_for_deleting_attachment` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `attachments=[{ id: "att-1", ... }]`, `canDelete=true`, `onDelete=jest.fn()`, `deletingId="att-1"`, `onError=jest.fn()` | `att-1` の削除ボタンが disabled になる |

### 7-3. AttachmentUploader

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-016 | 単体 | AttachmentUploader | `reportId: string`, `itemId: string` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `renders_upload_button` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`, `isUploading=false` | 「+ ファイルを追加」ボタンが表示される |
| ATT-FE-017 | 単体 | AttachmentUploader | `isUploading: boolean` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `shows_progress_bar_during_upload` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`, `isUploading=true` | プログレスバーが表示される。「+ ファイルを追加」ボタンが disabled になる |
| ATT-FE-018 | 単体 | AttachmentUploader | ファイル選択 | 正常系 | ATT-F01, ATT-002 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `accepts_valid_jpeg_file` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`, `isUploading=false`。ファイル入力に JPEG ファイル（1KB）を選択 | クライアントサイドバリデーションを通過し、useUploadAttachment.mutate が呼び出される |
| ATT-FE-019 | 単体 | AttachmentUploader | ファイル選択 | 正常系 | ATT-F01, ATT-002 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `accepts_valid_png_file` | Props: 同上。ファイル入力に PNG ファイル（1KB）を選択 | クライアントサイドバリデーションを通過し、useUploadAttachment.mutate が呼び出される |
| ATT-FE-020 | 単体 | AttachmentUploader | ファイル選択 | 正常系 | ATT-F01, ATT-002 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `accepts_valid_pdf_file` | Props: 同上。ファイル入力に PDF ファイル（1KB）を選択 | クライアントサイドバリデーションを通過し、useUploadAttachment.mutate が呼び出される |
| ATT-FE-021 | 単体 | AttachmentUploader | ファイル選択 | 異常系 | ATT-002, ATT-013 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `rejects_invalid_mime_type_gif` | Props: 同上。ファイル入力に GIF ファイル（`image/gif`、1KB）を選択 | クライアントサイドバリデーションでエラー。useUploadAttachment.mutate は呼び出されない。エラーメッセージ（許可されないファイル形式）が表示される |
| ATT-FE-022 | 単体 | AttachmentUploader | ファイル選択 | 異常系 | ATT-002, ATT-013 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `rejects_invalid_mime_type_txt` | Props: 同上。ファイル入力に テキストファイル（`text/plain`、1KB）を選択 | クライアントサイドバリデーションでエラー。useUploadAttachment.mutate は呼び出されない。エラーメッセージが表示される |
| ATT-FE-023 | 単体 | AttachmentUploader | ファイル選択 | 境界値 | ATT-003 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `rejects_file_exceeding_5mb` | Props: 同上。ファイル入力に JPEG ファイル（5,242,881 B = 5MB + 1B）を選択 | クライアントサイドバリデーションでエラー。useUploadAttachment.mutate は呼び出されない。エラーメッセージ（ファイルサイズ上限超過）が表示される |
| ATT-FE-024 | 単体 | AttachmentUploader | ファイル選択 | 境界値 | ATT-003 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `accepts_file_exactly_5mb` | Props: 同上。ファイル入力に JPEG ファイル（5,242,880 B = ちょうど 5MB）を選択 | クライアントサイドバリデーションを通過し、useUploadAttachment.mutate が呼び出される |
| ATT-FE-025 | 単体 | AttachmentUploader | ドラッグ&ドロップ | 正常系 | ATT-F01, ATT-002 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `accepts_valid_file_via_drag_and_drop` | Props: 同上。ドロップ領域に JPEG ファイル（1KB）をドラッグ&ドロップ | クライアントサイドバリデーションを通過し、useUploadAttachment.mutate が呼び出される |
| ATT-FE-026 | 単体 | AttachmentUploader | ドラッグ&ドロップ | 異常系 | ATT-002, ATT-013 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `rejects_invalid_file_via_drag_and_drop` | Props: 同上。ドロップ領域に GIF ファイル（1KB）をドラッグ&ドロップ | クライアントサイドバリデーションでエラー。エラーメッセージが表示される |
| ATT-FE-027 | 単体 | AttachmentUploader | `onUploadSuccess: () => void` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `calls_onUploadSuccess_after_successful_upload` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=mockFn`, `isUploading=false`。useUploadAttachment.mutate が成功を返すようモック設定。JPEG ファイルを選択 | アップロード成功後に `onUploadSuccess` コールバックが呼び出される |
| ATT-FE-028 | 単体 | AttachmentUploader | `isUploading: boolean` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `disables_upload_button_while_uploading` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`, `isUploading=true` | 「+ ファイルを追加」ボタンが disabled。ドラッグ&ドロップ領域も無効 |
| ATT-FE-051 | 単体 | AttachmentUploader | `isPending: boolean` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader`, SMK-012 | `shows_circular_progress_when_pending` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`。`isPending=true`（Hook の useUploadAttachment が pending 状態） | ボタン内に `CircularProgress`（`role="progressbar"`）が表示される（issue #100 対応） |
| ATT-FE-052 | 単体 | AttachmentUploader | DOM 構造 | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `renders_single_visually_hidden_input_and_single_button` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()` | visually-hidden な file input が DOM に 1 つだけ存在し、Button も 1 つだけ存在する（重複ボタン解消の確認、issue #100 対応） |
| ATT-FE-053 | 単体 | AttachmentUploader | ドラッグ&ドロップ | 正常系 | ATT-F01, ATT-002 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `shows_drag_over_visual_feedback` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`。ドロップゾーンに dragover イベントを発火 | dragover 時にドロップゾーンの `data-drag-over` 属性が `"true"` になり、dragleave 後に `"false"` に戻る（DnD 視覚フィードバック、issue #100 対応） |

### 7-4. Hook テスト（useAttachments）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-029 | 単体 | - | `useAttachments` | 正常系 | ATT-F02 | `55_ui_component/state-management.md §useAttachments` | `useAttachments_returns_attachment_list` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1" }`。API モック: 200 OK、`data: [{ id: "att-1", item_id: "item-1", file_name: "receipt.jpg", file_size: 245760, mime_type: "image/jpeg", created_at: "..." }]` | `data.data` が添付1件の配列。`isLoading` が false に遷移 |
| ATT-FE-030 | 単体 | - | `useAttachments` | 正常系 | ATT-F02 | `55_ui_component/state-management.md §useAttachments` | `useAttachments_returns_empty_array` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1" }`。API モック: 200 OK、`data: []` | `data.data` が空配列 |
| ATT-FE-031 | 単体 | - | `useAttachments` | 正常系 | ATT-F02 | `55_ui_component/state-management.md §useAttachments` | `useAttachments_uses_correct_query_key` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1" }` | クエリキーが `['reports', 'rpt-1', 'items', 'item-1', 'attachments']` である |
| ATT-FE-032 | 単体 | - | `useAttachments` | 異常系 | ATT-F02 | `55_ui_component/state-management.md §useAttachments` | `useAttachments_handles_api_error` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1" }`。API モック: 500 Internal Server Error | `isError` が true。`error` が `ApiClientError` |

### 7-5. Hook テスト（useAttachmentDownloadUrl / useAttachmentPreviewUrl）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-033 | 単体 | - | `useAttachmentDownloadUrl` | 正常系 | ATT-F03, ATT-010 | `55_ui_component/state-management.md §useAttachmentDownloadUrl` | `useAttachmentDownloadUrl_returns_signed_url` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }`。API モック: 200 OK、`data: { url: "https://s3.example.com/...", file_name: "receipt.jpg", mime_type: "image/jpeg", file_size: 245760, expires_at: "..." }` | `data.data.url` が署名付き URL を含む。レスポンス型は `AttachmentAccess` |
| ATT-FE-034 | 単体 | - | `useAttachmentDownloadUrl` | 正常系 | ATT-012 | `55_ui_component/state-management.md §useAttachmentDownloadUrl` | `useAttachmentDownloadUrl_staleTime_is_zero` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }` | staleTime が 0 であること（署名付き URL は毎回取得）。enabled: false であること |
| ATT-FE-035 | 単体 | - | `useAttachmentDownloadUrl` | 異常系 | ATT-F03 | `55_ui_component/state-management.md §useAttachmentDownloadUrl` | `useAttachmentDownloadUrl_handles_404` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "nonexistent" }`。API モック: 404 RESOURCE_NOT_FOUND | `isError` が true。`error.status` が 404 |

### 7-6. Hook テスト（useUploadAttachment）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-036 | 単体 | - | `useUploadAttachment` | 正常系 | ATT-F01 | `55_ui_component/state-management.md §useUploadAttachment` | `useUploadAttachment_sends_multipart_request` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", file: mockJpegFile }`。API モック: 201 Created、`data: { id: "att-new", ... }` | API リクエストが `multipart/form-data` 形式で送信される。`isPending` が true -> false に遷移。`data.data.id` が `"att-new"` |
| ATT-FE-037 | 単体 | - | `useUploadAttachment` | 正常系 | ATT-F01 | `55_ui_component/state-management.md §useUploadAttachment` | `useUploadAttachment_invalidates_report_detail_cache` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", file: mockJpegFile }`。API モック: 201 Created | 成功後に `['reports', 'detail', 'rpt-1']` クエリキーが invalidate される |
| ATT-FE-038 | 単体 | - | `useUploadAttachment` | 異常系 | ATT-013 | `55_ui_component/state-management.md §useUploadAttachment` | `useUploadAttachment_handles_422_invalid_file_type` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", file: mockGifFile }`。API モック: 422 INVALID_FILE_TYPE | `isError` が true。`error.code` が `"INVALID_FILE_TYPE"` |
| ATT-FE-039 | 単体 | - | `useUploadAttachment` | 異常系 | ATT-003 | `55_ui_component/state-management.md §useUploadAttachment` | `useUploadAttachment_handles_413_file_too_large` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", file: mockLargeFile }`。API モック: 413 FILE_TOO_LARGE | `isError` が true。`error.code` が `"FILE_TOO_LARGE"` |
| ATT-FE-040 | 単体 | - | `useUploadAttachment` | 異常系 | ATT-020 | `55_ui_component/state-management.md §useUploadAttachment` | `useUploadAttachment_handles_422_report_not_editable` | mutate 引数: `{ reportId: "rpt-submitted", itemId: "item-1", file: mockJpegFile }`。API モック: 422 REPORT_NOT_EDITABLE | `isError` が true。`error.code` が `"REPORT_NOT_EDITABLE"` |

### 7-7. Hook テスト（useDeleteAttachment）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-041 | 単体 | - | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/state-management.md §useDeleteAttachment` | `useDeleteAttachment_sends_delete_request` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }`。API モック: 204 No Content | `isSuccess` が true。`isPending` が true -> false に遷移 |
| ATT-FE-042 | 単体 | - | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/state-management.md §useDeleteAttachment` | `useDeleteAttachment_invalidates_report_detail_cache` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }`。API モック: 204 No Content | 成功後に `['reports', 'detail', 'rpt-1']` クエリキーが invalidate される |
| ATT-FE-043 | 単体 | - | `useDeleteAttachment` | 異常系 | ATT-F04 | `55_ui_component/state-management.md §useDeleteAttachment` | `useDeleteAttachment_handles_404` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "nonexistent" }`。API モック: 404 RESOURCE_NOT_FOUND | `isError` が true。`error.status` が 404 |
| ATT-FE-044 | 単体 | - | `useDeleteAttachment` | 異常系 | ATT-020 | `55_ui_component/state-management.md §useDeleteAttachment` | `useDeleteAttachment_handles_422_report_not_editable` | mutate 引数: `{ reportId: "rpt-submitted", itemId: "item-1", attId: "att-1" }`。API モック: 422 REPORT_NOT_EDITABLE | `isError` が true。`error.code` が `"REPORT_NOT_EDITABLE"` |

### 7-8. 統合テスト（AttachmentArea + Hook 連携）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-045 | 統合 | AttachmentArea | `useUploadAttachment` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `upload_success_shows_toast_and_refreshes_list` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。JPEG ファイルを選択してアップロード。API モック: 201 Created | アップロード成功後に AppToast で成功通知が表示される。添付一覧が再取得される |
| ATT-FE-046 | 統合 | AttachmentArea | `useUploadAttachment` | 異常系 | ATT-013 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `upload_failure_shows_error_toast` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。JPEG ファイルを選択してアップロード。API モック: 422 INVALID_FILE_TYPE | AppToast でエラー通知が表示される |
| ATT-FE-047 | 統合 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `delete_success_shows_toast_and_refreshes_list` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ボタンをクリック。API モック: 204 No Content | 削除成功後に AppToast で成功通知が表示される。添付一覧が再取得される |
| ATT-FE-048 | 統合 | AttachmentArea | `useDeleteAttachment` | 異常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `delete_failure_shows_error_toast` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ボタンをクリック。API モック: 500 Internal Server Error | AppToast でエラー通知が表示される |
| ATT-FE-049 | 統合 | AttachmentArea + AttachmentList（内部 AttachmentItemRow） | `useAttachmentPreviewUrl` / `useAttachmentDownloadUrl` | 正常系 | ATT-F03, ATT-010 | `55_ui_component/screens/report-detail.md §AttachmentList`, `50_detail_design/files.md §4.5` | `preview_opens_signed_url_and_download_triggers_anchor_click` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件（`file_name: "receipt.jpg"`）。`window.open` / `document.createElement` / `appendChild` / `removeChild` / `<a>.click` をスパイし、`window.open` は `newWindow` モックを返すよう設定。1. ファイル名をクリック: プレビュー API モック 200 OK、`data.url` に署名付き URL（inline）。2. ↓ アイコンをクリック: ダウンロード API モック 200 OK、`data.url` に署名付き URL（attachment）、`data.file_name = "receipt.jpg"` | 1. プレビュー: `window.open('about:blank', '_blank')` がクリック同期で呼ばれ、AttachmentItemRow 内の `useAttachmentPreviewUrl.refetch()` 成功後に `newWindow.location.href` が署名付き URL に差し替えられる。2. ダウンロード: `useAttachmentDownloadUrl.refetch()` 成功後に `<a>` 要素が生成され、`href` に署名付き URL、`download` 属性にファイル名 `"receipt.jpg"` が設定される。`document.body.appendChild` → `link.click()` → `document.body.removeChild` の順で呼ばれる。ダウンロード時は `window.open` が呼ばれないことを確認する |
| ATT-FE-050 | 統合 | AttachmentArea | `useUploadAttachment` | 異常系 | ATT-003 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `upload_too_large_shows_error_toast` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。5MB超過ファイルを選択してアップロード。API モック: 413 FILE_TOO_LARGE | AppToast でエラー通知（ファイルサイズ超過）が表示される |

---

## 8. FE テストケース サマリー

| テストID | 対象 | 分類 | 優先度 |
|---------|------|------|--------|
| ATT-FE-001 | AttachmentArea | 正常系（一覧+アップローダー描画） | 高 |
| ATT-FE-002 | AttachmentArea | 正常系（itemId=null で空状態） | 高 |
| ATT-FE-003 | AttachmentArea | 認可（canModify=false でアップローダー非表示） | 高 |
| ATT-FE-004 | AttachmentArea | 認可（canModify=true でアップローダー表示） | 高 |
| ATT-FE-005 | AttachmentArea | 正常系（マウント時の添付取得） | 中 |
| ATT-FE-006 | AttachmentArea | 正常系（削除中のグレーアウト制御） | 中 |
| ATT-FE-007 | AttachmentList | 正常系（ファイル情報表示） | 高 |
| ATT-FE-008 | AttachmentList | 正常系（空一覧） | 中 |
| ATT-FE-009 | AttachmentList | 正常系（複数件表示） | 中 |
| ATT-FE-010 | AttachmentList（AttachmentItemRow） | 正常系（プレビュー: クリック同期 window.open + refetch） | 高 |
| ATT-FE-011 | AttachmentList | 認可（canDelete=true で削除ボタン表示） | 高 |
| ATT-FE-012 | AttachmentList | 認可（canDelete=false で削除ボタン非表示） | 高 |
| ATT-FE-013 | AttachmentList | 正常系（削除コールバック） | 高 |
| ATT-FE-014 | AttachmentList | 正常系（削除中グレーアウト） | 中 |
| ATT-FE-015 | AttachmentList | 正常系（削除中ボタン disabled） | 中 |
| ATT-FE-016 | AttachmentUploader | 正常系（アップロードボタン描画） | 高 |
| ATT-FE-017 | AttachmentUploader | 正常系（アップロード中プログレスバー） | 高 |
| ATT-FE-018 | AttachmentUploader | 正常系（JPEG 受理） | 高 |
| ATT-FE-019 | AttachmentUploader | 正常系（PNG 受理） | 高 |
| ATT-FE-020 | AttachmentUploader | 正常系（PDF 受理） | 高 |
| ATT-FE-021 | AttachmentUploader | 異常系（GIF 拒否） | 高 |
| ATT-FE-022 | AttachmentUploader | 異常系（TXT 拒否） | 中 |
| ATT-FE-023 | AttachmentUploader | 境界値（5MB超過拒否） | 高 |
| ATT-FE-024 | AttachmentUploader | 境界値（5MBちょうど受理） | 高 |
| ATT-FE-025 | AttachmentUploader | 正常系（ドラッグ&ドロップ受理） | 高 |
| ATT-FE-026 | AttachmentUploader | 異常系（ドラッグ&ドロップ拒否） | 中 |
| ATT-FE-027 | AttachmentUploader | 正常系（成功コールバック呼び出し） | 高 |
| ATT-FE-028 | AttachmentUploader | 正常系（アップロード中ボタン disabled） | 中 |
| ATT-FE-051 | AttachmentUploader | 正常系（isPending 時 CircularProgress 表示、issue #100） | 高 |
| ATT-FE-052 | AttachmentUploader | 正常系（hidden input/Button 重複なし、issue #100） | 中 |
| ATT-FE-053 | AttachmentUploader | 正常系（DnD 視覚フィードバック、issue #100） | 中 |
| ATT-FE-054 | AttachmentArea | 正常系（削除確認ダイアログ表示、issue #103） | 高 |
| ATT-FE-055 | AttachmentArea | 正常系（ダイアログキャンセル時 API 未呼出、issue #103） | 高 |
| ATT-FE-056 | AttachmentArea | 正常系（ダイアログ削除確定 API 呼出、issue #103） | 高 |
| ATT-FE-029 | useAttachments | 正常系（添付一覧取得） | 高 |
| ATT-FE-030 | useAttachments | 正常系（空配列） | 中 |
| ATT-FE-031 | useAttachments | 正常系（クエリキー検証） | 中 |
| ATT-FE-032 | useAttachments | 異常系（API エラー） | 高 |
| ATT-FE-033 | useAttachmentDownloadUrl | 正常系（署名付き URL 取得） | 高 |
| ATT-FE-034 | useAttachmentDownloadUrl | 正常系（staleTime=0 検証） | 中 |
| ATT-FE-035 | useAttachmentDownloadUrl | 異常系（404 ハンドリング） | 高 |
| ATT-FE-036 | useUploadAttachment | 正常系（multipart リクエスト送信） | 高 |
| ATT-FE-037 | useUploadAttachment | 正常系（キャッシュ無効化） | 高 |
| ATT-FE-038 | useUploadAttachment | 異常系（422 INVALID_FILE_TYPE） | 高 |
| ATT-FE-039 | useUploadAttachment | 異常系（413 FILE_TOO_LARGE） | 高 |
| ATT-FE-040 | useUploadAttachment | 異常系（422 REPORT_NOT_EDITABLE） | 高 |
| ATT-FE-041 | useDeleteAttachment | 正常系（削除リクエスト送信） | 高 |
| ATT-FE-042 | useDeleteAttachment | 正常系（キャッシュ無効化） | 高 |
| ATT-FE-043 | useDeleteAttachment | 異常系（404 ハンドリング） | 高 |
| ATT-FE-044 | useDeleteAttachment | 異常系（422 REPORT_NOT_EDITABLE） | 高 |
| ATT-FE-045 | AttachmentArea 統合 | 正常系（アップロード成功トースト+一覧再取得） | 高 |
| ATT-FE-046 | AttachmentArea 統合 | 異常系（アップロード失敗エラートースト） | 高 |
| ATT-FE-047 | AttachmentArea 統合 | 正常系（削除成功トースト+一覧再取得） | 高 |
| ATT-FE-048 | AttachmentArea 統合 | 異常系（削除失敗エラートースト） | 高 |
| ATT-FE-049 | AttachmentArea + AttachmentList 統合 | 正常系（プレビュー: クリック同期 window.open / ダウンロード: 動的 `<a download>` 要素クリック） | 高 |
| ATT-FE-050 | AttachmentArea 統合 | 異常系（サイズ超過エラートースト） | 中 |

### 7-9. ID 振り直し・追記テスト（issue #103 / issue #099）

以下のテスト ID は重複解消（issue #103）またはテスト実装との整合（issue #099）のために追記する。

#### AttachmentArea -- 削除確認ダイアログ（旧 ATT-FE-007/008/009 → 振り直し）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-054 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `shows_delete_confirm_dialog` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ボタンをクリック | 削除確認ダイアログが表示される（issue #103 対応） |
| ATT-FE-055 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `cancel_dialog_does_not_call_api` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ダイアログでキャンセルをクリック | useDeleteAttachment.mutate が呼び出されない（issue #103 対応） |
| ATT-FE-056 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `confirm_dialog_calls_delete_api` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ダイアログで確定をクリック | useDeleteAttachment.mutate が呼び出される（issue #103 対応） |

**備考**: ATT-FE-054/055/056 は、設計書の旧 ATT-FE-007/008/009（AttachmentArea 側の重複 ID）を振り直したもの。元の ATT-FE-007/008/009（AttachmentList 側）はそのまま維持する。

#### AttachmentList -- ファイルサイズ表示

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-007b | 単体 | AttachmentList | `attachments: Attachment[]` | 正常系 | ATT-F02 | `55_ui_component/screens/report-detail.md §AttachmentList` | `renders_file_size_in_mb_format` | Props: `attachments=[{ id: "att-1", file_name: "receipt.jpg", file_size: 1048576, mime_type: "image/jpeg" }]`（1MB 以上のファイル） | ファイルサイズが "X.X MB" 形式で表示される |

#### useAttachmentDownloadUrl -- 403 ハンドリング

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-035b | 単体 | - | `useAttachmentDownloadUrl` | 異常系 | ATT-F03, ATT-011 | `55_ui_component/state-management.md §useAttachmentDownloadUrl` | `useAttachmentDownloadUrl_handles_403` | Hook 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }`。API モック: 403 FORBIDDEN | `isError` が true。`error.status` が 403 |

#### useDeleteAttachment -- 添付一覧キャッシュ無効化

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-042b | 単体 | - | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/state-management.md §useDeleteAttachment` | `useDeleteAttachment_invalidates_attachments_cache` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }`。API モック: 204 No Content | 成功後に `['reports', 'rpt-1', 'items', 'item-1', 'attachments']` クエリキーが invalidate される（issue #099 修正） |

---

**FE テストケース合計: 59件**（優先度「高」: 44件、優先度「中」: 15件）

### FE テスト実装ガイド

#### テストファイル配置

| テストファイル | 対象 |
|-------------|------|
| `src/pages/reports/__tests__/AttachmentArea.test.tsx` | AttachmentArea コンポーネントテスト（ATT-FE-001〜006） |
| `src/pages/reports/__tests__/AttachmentList.test.tsx` | AttachmentList コンポーネントテスト（ATT-FE-007〜015） |
| `src/pages/reports/__tests__/AttachmentUploader.test.tsx` | AttachmentUploader コンポーネントテスト（ATT-FE-016〜028, ATT-FE-051〜053） |
| `src/hooks/__tests__/useAttachments.test.ts` | Hook テスト（ATT-FE-029〜044） |
| `src/pages/reports/__tests__/AttachmentArea.integration.test.tsx` | 統合テスト（ATT-FE-045〜050） |

#### モック方針

- **API モック**: MSW（Mock Service Worker）でエンドポイントをモックする。テストケースごとにハンドラのレスポンスを切り替える
- **Hook モック**: コンポーネント単体テスト（ATT-FE-001〜028）では `useAttachments`, `useUploadAttachment`, `useDeleteAttachment`, `useAttachmentDownloadUrl`, `useAttachmentPreviewUrl` を vi.mock でモックする
- **Hook テスト（ATT-FE-029〜044）**: `@testing-library/react-hooks` の `renderHook` + MSW で実際の API 呼び出しをモックし、Hook の振る舞いを検証する
- **統合テスト（ATT-FE-045〜050）**: Hook をモックせず MSW のみで API をモックし、コンポーネント + Hook の連携を検証する

#### ファイルオブジェクトの生成

```typescript
// テスト用ファイルオブジェクト生成ヘルパー
function createMockFile(name: string, size: number, type: string): File {
  const buffer = new ArrayBuffer(size);
  return new File([buffer], name, { type });
}

// 使用例
const mockJpegFile = createMockFile('receipt.jpg', 1024, 'image/jpeg');
const mockPngFile = createMockFile('receipt.png', 1024, 'image/png');
const mockPdfFile = createMockFile('invoice.pdf', 1024, 'application/pdf');
const mockGifFile = createMockFile('image.gif', 1024, 'image/gif');
const mockTxtFile = createMockFile('notes.txt', 1024, 'text/plain');
const mockLargeFile = createMockFile('large.jpg', 5_242_881, 'image/jpeg'); // 5MB + 1B
const mockExact5mbFile = createMockFile('exact.jpg', 5_242_880, 'image/jpeg'); // ちょうど 5MB
```

#### ドラッグ&ドロップのテスト

```typescript
// fireEvent.drop でドロップイベントをシミュレートする
import { fireEvent } from '@testing-library/react';

const dropZone = screen.getByTestId('attachment-drop-zone');
const file = createMockFile('receipt.jpg', 1024, 'image/jpeg');

fireEvent.drop(dropZone, {
  dataTransfer: {
    files: [file],
    types: ['Files'],
  },
});
```

#### 注意事項

- テストデータに機密情報を含めない
- テナント ID が混在しないよう、全テストケースで単一テナントのモックデータを使用する
- AppToast の検証は `@testing-library/react` の `screen.getByRole('alert')` または `screen.getByText` で行う
- `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` の staleTime=0 検証（ATT-FE-034）は QueryClient のクエリキャッシュ設定を直接確認する

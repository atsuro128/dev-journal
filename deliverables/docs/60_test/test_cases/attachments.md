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
| ATT-FE-051 | 単体 | AttachmentUploader | `isPending: boolean` | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader`, SMK-012 | `shows_circular_progress_when_pending` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`。`isPending=true`（Hook の useUploadAttachment が pending 状態） | ボタン内に `CircularProgress`（`role="progressbar"`）が表示される |
| ATT-FE-052 | 単体 | AttachmentUploader | DOM 構造 | 正常系 | ATT-F01 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `renders_single_visually_hidden_input_and_single_button` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()` | visually-hidden な file input が DOM に 1 つだけ存在し、Button も 1 つだけ存在する（重複ボタン解消の確認） |
| ATT-FE-053 | 単体 | AttachmentUploader | ドラッグ&ドロップ | 正常系 | ATT-F01, ATT-002 | `55_ui_component/screens/report-detail.md §AttachmentUploader` | `shows_drag_over_visual_feedback` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `onUploadSuccess=jest.fn()`。ドロップゾーンに dragover イベントを発火 | dragover 時にドロップゾーンの `data-drag-over` 属性が `"true"` になり、dragleave 後に `"false"` に戻る（DnD 視覚フィードバック） |

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

### 7-9. ID 振り直し・追記テスト

以下のテスト ID は重複解消またはテスト実装との整合のために追記する。

#### AttachmentArea -- 削除確認ダイアログ（旧 ATT-FE-007/008/009 → 振り直し）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-054 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `shows_delete_confirm_dialog` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ボタンをクリック | 削除確認ダイアログが表示される |
| ATT-FE-055 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `cancel_dialog_does_not_call_api` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ダイアログでキャンセルをクリック | useDeleteAttachment.mutate が呼び出されない |
| ATT-FE-056 | 単体 | AttachmentArea | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/screens/report-detail.md §AttachmentArea` | `confirm_dialog_calls_delete_api` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。添付1件。削除ダイアログで確定をクリック | useDeleteAttachment.mutate が呼び出される |

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
| ATT-FE-042b | 単体 | - | `useDeleteAttachment` | 正常系 | ATT-F04 | `55_ui_component/state-management.md §useDeleteAttachment` | `useDeleteAttachment_invalidates_attachments_cache` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }`。API モック: 204 No Content | 成功後に `['reports', 'rpt-1', 'items', 'item-1', 'attachments']` クエリキーが invalidate される |

### 7-10. フォーム編集中の操作整合性

並行操作整合性 / 破棄確認ダイアログの 2 つの課題に対応するテストケースを定義する。設計の根拠は `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」/ §7「アップロード・削除中の並行操作制御」`。

#### 7-10-1. 並行操作整合性（AttachmentUploader / AttachmentArea / ItemSlidePanel）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-057 | 単体 | ItemSlidePanel | `isPending: boolean`, `isUploading: boolean`, `isDeleting: boolean` | 正常系 | ATT-F01, ATT-F04 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」（§7-1 / §7-3）` | `disables_save_button_when_upload_or_delete_in_progress` | Props の組み合わせでフッターの「保存する」「保存して続けて追加」ボタンの状態を検証: (a) `isPending=false, isUploading=true, isDeleting=false`、(b) `isPending=true, isUploading=false, isDeleting=false`、(c) `isPending=false, isUploading=false, isDeleting=true`（削除中も保存ボタン disabled、§7-3 と整合）、(d) `isPending=false, isUploading=false, isDeleting=false`（すべて false） | (a)(b)(c) いずれの場合も「保存する」「保存して続けて追加」ボタンが disabled になる。(d) の場合のみ enabled になる。`isPending \|\| isUploading \|\| isDeleting` の OR 合成で反映される（§7-1 保存ボタン制御と §7-3 削除処理中 disabled の整合） |
| ATT-FE-058 | 単体 | ItemSlidePanel / ItemForm | `isUploading: boolean` | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」` | `allows_close_cancel_and_field_edit_during_upload` | Props: `isPending=false`, `isUploading=true`。× ボタン・キャンセルボタン・フォームの金額/日付/カテゴリ/摘要フィールド | × ボタン・キャンセルボタンはクリック可能、各フォームフィールドも編集可能（disabled にならない） |
| ATT-FE-059 | 単体 | - | `useUploadAttachment` | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」` | `aborts_upload_mutation_on_panel_close` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", file: mockJpegFile }` で mutation 開始直後にコンポーネントを unmount（パネル閉じ相当）。API モックはレスポンスを遅延させる | `AbortController.abort()` が呼び出され、進行中の fetch に AbortSignal が伝播する。mutation が cancel 状態で終了する |
| ATT-FE-060 | 統合 | AttachmentArea + ItemSlidePanel | `useUploadAttachment` | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」` | `shows_upload_aborted_toast_on_panel_close_during_upload` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。JPEG ファイルをアップロード開始後、レスポンス受信前に × ボタン（またはキャンセル/外クリック）でパネルを閉じる。API モック: レスポンス遅延 | AppToast で「アップロードを中止しました」相当の中断通知が表示される。`onUploadSuccess` コールバックは呼ばれない |
| ATT-FE-061 | 単体 | - | `useDeleteAttachment` | 正常系 | ATT-F04 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」` | `aborts_delete_mutation_on_panel_close` | mutate 引数: `{ reportId: "rpt-1", itemId: "item-1", attId: "att-1" }` で mutation 開始直後にコンポーネントを unmount。API モックはレスポンスを遅延させる | `AbortController.abort()` が呼び出され、進行中の DELETE リクエストに AbortSignal が伝播する。mutation が cancel 状態で終了する |
| ATT-FE-062 | 統合 | AttachmentArea + ItemSlidePanel | `useDeleteAttachment` | 正常系 | ATT-F04 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」` | `shows_delete_aborted_toast_on_panel_close_during_delete` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。削除ボタンをクリック（確認ダイアログ確定）後、レスポンス受信前に × ボタンでパネルを閉じる。API モック: レスポンス遅延 | AppToast で「削除を中止しました」相当の中断通知が表示される。添付一覧の invalidate は発火しない |
| ATT-FE-063 | 統合 | AttachmentArea + ItemSlidePanel | `useUploadAttachment` / `useDeleteAttachment` | 正常系 | ATT-F01, ATT-F04 | `50_detail_design/screens/report-detail.md §7「アップロード・削除中の並行操作制御」` | `cancels_in_flight_mutation_when_switching_item` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。アップロード（または削除）を開始し、レスポンス受信前に別明細（`itemId="item-2"`）を開く → ItemSlidePanel が再マウントされる。API モック: レスポンス遅延 | 元明細向けの mutation が AbortController で cancel される。新明細の添付一覧は独立して取得される |

#### 7-10-2. 破棄確認ダイアログ（ItemForm / ItemSlidePanel）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-064 | 単体 | ItemSlidePanel + ItemForm | dirty 判定 | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」（破棄確認ダイアログの仕様）` | `shows_discard_dialog_on_close_button_when_dirty` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `canModify=true`。金額フィールドを初期値から変更（dirty）後、パネル右上の × ボタンをクリック | MUI Dialog が表示され、以下の 4 項目が設計書 §6「破棄確認ダイアログの仕様」と完全一致する。パネルはまだ閉じていない。<br/>1. タイトル文言一致: `screen.getByText('変更を破棄しますか？')` が見つかる（末尾は全角疑問符「？」）<br/>2. 本文文言一致: `screen.getByText('編集内容は保存されていません。破棄するとこれまでの変更が失われます。')` が見つかる<br/>3. 確認ボタン文言・スタイル: `getByRole('button', { name: '破棄' })` が存在し、危険スタイル（MUI `color="error"` 相当、CSS クラスで検証）である<br/>4. キャンセルボタン文言: `getByRole('button', { name: 'キャンセル' })` が存在する |
| ATT-FE-065 | 単体 | ItemSlidePanel + ItemForm | dirty 判定 | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」` | `shows_discard_dialog_on_cancel_button_when_dirty` | Props: 同上。摘要フィールドを dirty にした後、フッターの「キャンセル」ボタンをクリック | MUI Dialog が表示される。パネルはまだ閉じていない |
| ATT-FE-066 | 単体 | ItemSlidePanel + ItemForm | dirty 判定 | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」` | `shows_discard_dialog_on_overlay_click_when_dirty` | Props: 同上。カテゴリを dirty にした後、Drawer のオーバーレイ（明細外）をクリック（`onClose` 発火） | MUI Dialog が表示される。Drawer は閉じていない |
| ATT-FE-067 | 単体 | ItemSlidePanel + ItemForm | dirty 判定 | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」` | `closes_immediately_when_not_dirty` | Props: 同上。フィールドを初期値から変更していない（非 dirty）状態で、× ボタン / キャンセルボタン / オーバーレイクリックの 3 パターンで閉じ操作 | いずれも MUI Dialog は表示されず、Drawer が即座に閉じる |
| ATT-FE-068 | 単体 | ItemSlidePanel + ItemForm | 破棄確認ダイアログ | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」` | `discards_changes_on_dialog_discard_button` | Props: 同上。dirty 状態で閉じ操作 → Dialog 表示 → 「破棄」ボタンをクリック | Dialog が閉じ、Drawer も閉じる。ItemForm の編集内容は破棄される（保存 API は呼び出されない） |
| ATT-FE-069 | 単体 | ItemSlidePanel + ItemForm | 破棄確認ダイアログ | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」` | `keeps_panel_open_on_dialog_cancel_button` | Props: 同上。dirty 状態で閉じ操作 → Dialog 表示 → 「キャンセル」ボタンをクリック | Dialog のみが閉じる。Drawer は開いたままで、ItemForm のフィールド値（編集途中の内容）は保持される |
| ATT-FE-070 | 単体 | ItemSlidePanel + ItemForm | dirty 判定 | 正常系 | ATT-F01, ATT-F02, ATT-F04 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」(dirty 判定対象外) / §7「永続化タイミング」` | `attachment_changes_do_not_mark_form_dirty` | Props: 同上。フォームフィールドは初期値のまま、添付ファイルの追加（アップロード成功）または削除のみを実行した後、× ボタンをクリック | MUI Dialog は表示されず、Drawer が即座に閉じる。添付操作は即時保存方式のため Form の dirty とは独立（§7 冒頭参照） |
| ATT-FE-071 | 単体 | ItemSlidePanel + ItemForm | `beforeunload` | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」（閉じパターン #4）` | `prevents_default_on_beforeunload_when_dirty` | Props: 同上。dirty 状態で `window` に `beforeunload` イベントを発火（F5 リロード / タブ閉じ / ブラウザ閉じ相当）。`event.preventDefault` をスパイ | dirty 時のみ `beforeunload` ハンドラが登録されており、`event.preventDefault()` が呼び出される（ブラウザ標準の離脱確認ダイアログが表示される）。非 dirty 時は `preventDefault` が呼び出されない |

**備考**: 
- ATT-FE-057 / 058 は既存 ATT-FE-028（`isUploading` 単独の disabled）と区別し、`isPending || isUploading || isDeleting` の OR 合成と閉じる/編集系の操作が有効なままであることを検証する「並行操作」観点のテスト。
- ATT-FE-071 の `beforeunload` は仕様上カスタム文言表示不可 / MUI Dialog 表示不可のため、`event.preventDefault()` の発火有無のみを検証する（`50_detail_design/screens/report-detail.md §6` の技術制約と整合）。

### 7-11. 新規明細での添付（ローカル保持 + 保存時順次アップロード）

ローカル保持方式に対応するテストケースを定義する。追加モードでは AttachmentArea が表示され、ファイル選択時には即時アップロードせずフロントのローカル state に保留し、明細フォームの「保存する」押下時に明細作成 → 保留添付の順次アップロードを実行する。編集モードの挙動（即時アップロード）は変更しない。設計の根拠は `50_detail_design/screens/report-detail.md §6「モード別の保存処理（追加モード時の順次アップロード）」/ §7「モード別の添付挙動（追加モードはローカル保持方式）」`、および `50_detail_design/files.md §3.1`。

#### 7-11-1. AttachmentArea -- 追加モード表示 / ローカル保持 / 削除

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-072 | 単体 | AttachmentArea | `itemId: string \| null`, `mode: 'add' \| 'edit' \| 'view'` | 正常系 | ATT-F01, ATT-F02 | `50_detail_design/screens/report-detail.md §7「モード別の添付挙動」（モード別サマリ）` | `renders_attachment_area_in_add_mode_with_itemId_null` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。useAttachments は呼び出されないことを期待 | AttachmentArea が表示される（ATT-FE-002 の空状態とは異なり、追加モードでは `itemId=null` でも描画される）。AttachmentUploader が表示され、AttachmentList は空状態から開始する。useAttachments は `itemId=null` のため API 呼び出しをスキップする |
| ATT-FE-073 | 単体 | AttachmentArea / AttachmentUploader | ファイル選択（追加モード） | 正常系 | ATT-F01, ATT-F02 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（ファイル選択 / 一覧表示）` | `buffers_selected_file_in_local_state_in_add_mode` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。有効な JPEG（1KB）を選択 | `useUploadAttachment.mutate` は呼ばれない。保留中ファイルとしてローカル state に保持され、AttachmentList 相当の一覧にファイル名・サイズが**編集モードと同一の UI 構造**（`<ul><li>` + `<Button variant="text">` ファイル名 + `<span>` サイズ + `<Button variant="text" color="error">削除</Button>`）で表示される。「保留中」「保存後にアップロード予定」等の実装詳細を示すラベル / バッジは表示されない |
| ATT-FE-074 | 単体 | AttachmentArea / AttachmentUploader | クライアント側バリデーション（追加モード） | 異常系 | ATT-002, ATT-003, ATT-013 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（クライアント側バリデーション）` | `rejects_invalid_mime_or_oversize_file_in_add_mode_without_buffering` | Props: 同上。3 ケース検証: (a) GIF（`image/gif`, 1KB）を選択、(b) JPEG 5,242,881 B（5MB + 1B）を選択、(c) 5,242,880 B（ちょうど 5MB）を選択 | (a)(b) はローカル state に保留されず、MIME / サイズ違反のエラーメッセージが表示される。(c) は保留される。いずれのケースでも `useUploadAttachment.mutate` は呼ばれない（§7「アップロードのバリデーション」V1/V2 と同一ルール） |
| ATT-FE-075 | 単体 | AttachmentArea / AttachmentList | 保留中添付の削除 | 正常系 | ATT-F04 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（保留中添付の削除）` | `removes_pending_attachment_from_local_state_without_api_call` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。JPEG 2 件を順に選択して保留 → 1 件目の `<Button variant="text" color="error">削除</Button>`（編集モードと同一構造のテキスト削除ボタン）をクリック | ローカル state から対象ファイルのみが除去され、一覧には残り 1 件が表示される。`useDeleteAttachment.mutate` は呼ばれない。削除確認ダイアログ（ATT-FE-054 系）は表示されない |
| ATT-FE-077 | 単体 | AttachmentArea / AttachmentList | 保留中添付のダウンロード/プレビュー | 正常系 | ATT-F02, ATT-F03 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（ダウンロード / プレビュー）` | `hides_download_icon_for_pending_attachments_but_keeps_preview` | Props: 同上（追加モード + 保留中 1 件） | 保留中の行には**ダウンロードアイコン（編集モードの `<IconButton>↓</IconButton>`）のみが非表示**となり、ファイル名のプレビューリンク（`<Button variant="text">`）・ファイルサイズ（`<span>`）・削除ボタン（`<Button variant="text" color="error">削除</Button>`）は編集モードと同一構造で表示される。`useAttachmentDownloadUrl` は呼ばれない。ファイル名クリック時は `URL.createObjectURL` の Blob URL を新タブで開く（`useAttachmentPreviewUrl` も呼ばれない）。「保留中」「保存後にアップロード予定」等のラベルは表示されない |

##### add / edit 両モードでのトースト発火・UI 構造一致テスト

UI 一貫性優先（内部実装の差異をユーザーに見せない）の方針に対応する。トースト文言・リスト UI 構造を編集モードと完全同一にし、ダウンロードアイコンのみ add モードで非表示とする方針を検証する。

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-076 | 単体 | AttachmentArea / AttachmentUploader | 追加モードのアップロード成功トースト | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（アップロード成功時のトースト）`, `55_ui_component/state-management.md §6.5.5「添付ファイル成功トーストの発火条件」` | `shows_upload_success_toast_in_add_mode_with_same_wording_as_edit_mode` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。有効な JPEG（1KB）を選択（バリデーション通過 → ローカル state への保留が完了）。`AppToast` 表示用 state（または `SnackbarContext.showSnackbar`）をスパイ | `useUploadAttachment.mutate` は呼ばれない。AppToast に `severity="success"` で**編集モードと完全同一の文言「ファイルをアップロードしました」**が発火する（add モード固有の文言「保留しました」「保存後にアップロード予定」等は使われない）。文言定数は `ATTACHMENT_UPLOAD_SUCCESS` 等の共通定数を参照する |
| ATT-FE-084 | 単体 | AttachmentArea / AttachmentList | 追加モードの pending 削除トースト | 正常系 | ATT-F04 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（保留中添付の削除）`, `55_ui_component/state-management.md §6.5.5「添付ファイル成功トーストの発火条件」` | `shows_delete_success_toast_in_add_mode_with_same_wording_as_edit_mode` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。有効な JPEG を 1 件保留した状態で「削除」ボタンをクリック。`AppToast` 表示用 state をスパイ | `useDeleteAttachment.mutate` は呼ばれず、削除確認ダイアログも表示されない（add モードはローカル state からの除去のみ）。AppToast に `severity="success"` で**編集モードと完全同一の文言「添付ファイルを削除しました」**が発火する。文言定数は `ATTACHMENT_DELETE_SUCCESS` 等の共通定数を参照する |
| ATT-FE-085 | 単体 | AttachmentArea / AttachmentList | 追加モードのリスト UI 構造 | 正常系 | ATT-F02 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（一覧表示）`, `50_detail_design/screens/report-detail.md §7「保留中ファイル」` | `renders_pending_list_with_same_dom_structure_as_edit_mode` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。有効な JPEG（`receipt.jpg`, 245,760 B）を 1 件保留した状態 | DOM 構造を以下の 4 点で検証する: (1) 保留中ファイル一覧のルートが `<ul>` 要素で、各項目が `<li>` 要素である。(2) ファイル名は `<Button variant="text">`（MUI Button、CSS クラスで検証）でレンダリングされている。(3) ファイルサイズは `<span>` 要素で `XX KB` / `X.X MB` 形式（例: `240.0 KB`）で表示される（`<Typography variant="caption">` ではない）。(4) 削除ボタンは `<Button variant="text" color="error">削除</Button>`（テキストボタン、ラベル文言「削除」、`<IconButton>×</IconButton>` ではない）。編集モード（ATT-FE-007 系）の DOM 構造と一致する |
| ATT-FE-086 | 単体 | AttachmentArea / AttachmentList | 追加モードのダウンロードアイコン非表示 | 正常系 | ATT-F03 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」（ダウンロード）`, `50_detail_design/screens/report-detail.md §7「保留中ファイル」` | `hides_download_icon_only_in_add_mode_and_keeps_other_elements` | Props: `reportId="rpt-1"`, `itemId=null`, `mode="add"`, `canModify=true`。有効な JPEG を 1 件保留した状態 | 行内のダウンロードアイコン（編集モードで使われる `<IconButton aria-label="download">↓</IconButton>` 相当）が **DOM 上に存在しない**。一方、ファイル名（`<Button variant="text">`）・サイズ（`<span>`）・削除ボタン（`<Button variant="text" color="error">削除</Button>`）は描画されている。差分が「ダウンロードアイコンの有無のみ」であることを検証する。`useAttachmentDownloadUrl` Hook は呼ばれない |
| ATT-FE-087 | 単体 | AttachmentArea / AttachmentList | add / edit リスト UI 構造の同型性 | 正常系 | ATT-F02 | `50_detail_design/screens/report-detail.md §7「追加モードでの添付 UI 挙動」`, `50_detail_design/screens/report-detail.md §7「保留中ファイル」` | `add_and_edit_mode_lists_share_same_markup_except_download_icon` | (a) Props: `mode="edit"`, `itemId="item-1"`, `canModify=true`、`useAttachments` モックで添付 1 件（`receipt.jpg`, 245,760 B）を返す。(b) Props: `mode="add"`, `itemId=null`, `canModify=true`、有効な JPEG（同名同サイズ）を 1 件保留。両ケースで AttachmentList のスナップショット相当の DOM 構造を取得して比較 | (a) と (b) の DOM 構造を比較した結果、`<ul><li>` 構造・`<Button variant="text">`（ファイル名）・`<span>`（サイズ）・`<Button variant="text" color="error">削除</Button>`（削除ボタン）の存在は完全一致する。差分は (a) にダウンロードアイコン（`<IconButton>↓</IconButton>`）が存在し、(b) には存在しない、という 1 点のみ。クラス名・要素ツリー・テキスト内容（ファイル名・サイズ表記）はその他すべて一致する（UI 一貫性の最終ゲート） |

#### 7-11-2. 編集モードの挙動不変

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-078 | 単体 | AttachmentArea / AttachmentUploader | ファイル選択（編集モード） | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §7「永続化タイミング（即時保存方式）」/ §7「モード別の添付挙動」（モード別サマリ）` | `keeps_edit_mode_behavior_unchanged_immediate_upload` | Props: `reportId="rpt-1"`, `itemId="item-1"`, `mode="edit"`, `canModify=true`。有効な JPEG（1KB）を選択 | ファイル選択時点で `useUploadAttachment.mutate` が `{ reportId: "rpt-1", itemId: "item-1", file }` で呼び出される（即時アップロード）。ローカル state に保留されない（追加モードとの分岐が編集モードに波及しないことを確認） |

#### 7-11-3. 保存時の順次アップロード（ItemSlidePanel / ItemForm）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-079 | 統合 | ItemSlidePanel + ItemForm + AttachmentArea | 保存時順次アップロード（全成功） | 正常系 | ATT-F01, ITM-F01 | `50_detail_design/screens/report-detail.md §6「モード別の保存処理」（追加モードの「保存する」押下時シーケンス）` | `saves_item_then_sequentially_uploads_pending_attachments_on_save` | Props: `mode="add"`, `reportId="rpt-1"`, `canModify=true`。有効なフォーム値を入力し、保留添付 2 件（JPEG + PDF）をセット。API モック: (1) POST `/api/reports/rpt-1/items` → 201 Created、レスポンス `data.id = "item-new"`、(2) POST `/api/reports/rpt-1/items/item-new/attachments` × 2 件 → いずれも 201 Created。保存ボタンをクリック | 呼び出し順序が以下の順で行われる: (1) 明細作成 POST が先行し、レスポンスから `itemId="item-new"` を取得。(2) 取得した `itemId` で添付アップロード POST を 1 件ずつ**順次**実行する（並列ではない。2 件目の POST は 1 件目のレスポンス受信後に発火する）。全成功後にパネルがクローズし、明細一覧が再読み込みされる（`['reports', 'detail', 'rpt-1']` の invalidate）。トーストで「明細を追加しました」が表示される（files.md §9 レート制限との整合） |
| ATT-FE-080 | 統合 | ItemSlidePanel + ItemForm + AttachmentArea | 保存時順次アップロード（部分失敗） | 異常系 | ATT-F01, ATT-013 | `50_detail_design/screens/report-detail.md §6「モード別の保存処理」（部分失敗時のロールバック方針）` | `keeps_item_created_and_shows_warning_toast_on_partial_attachment_failure` | Props: 同上。保留添付 2 件をセット。API モック: (1) POST `/api/reports/rpt-1/items` → 201（`item_id="item-new"`）、(2) 1 件目の添付 POST → 201、(3) 2 件目の添付 POST → 500 Internal Server Error。保存ボタンをクリック | 明細は作成済みのまま維持される（明細の DELETE API は**呼ばれない**）。AppToast に「N 件の添付ファイルがアップロードに失敗しました。再試行してください」相当の警告通知が表示される。トースト表示直後にパネルがクローズし、明細一覧が再読み込みされる（トーストは画面右上の Snackbar 領域に表示継続）。呼び出し順序は「警告トースト表示 → パネルクローズ → 明細一覧再読み込み」の順であることを検証する |
| ATT-FE-081 | 単体 | ItemSlidePanel + ItemForm | 順次アップロード中の UI | 正常系 | ATT-F01, ITM-F01 | `50_detail_design/screens/report-detail.md §6「順次アップロード中の UI 表示」` | `disables_save_button_and_sets_readonly_form_during_sequential_upload` | Props: `mode="add"`, `canModify=true`。保留添付 3 件をセット。API モック: POST `items` は即時 201、POST `attachments` はレスポンスを遅延させ、1 件目の完了後・2 件目進行中の状態で検証 | 「保存する」「保存して続けて追加」ボタンが disabled + スピナー付きになり、ラベルが「アップロード中... (1/3 件完了)」の形式で表示される。ItemForm の日付・金額・カテゴリ・摘要の各フィールドが readonly になる（明細作成後の順次アップロード中の二重送信・整合性崩れ防止） |
| ATT-FE-082 | 統合 | ItemSlidePanel + ItemForm + AttachmentArea | 順次アップロード中のパネル閉じ（AbortController） | 正常系 | ATT-F01 | `50_detail_design/screens/report-detail.md §6「順次アップロード中の UI 表示」（閉じる操作） / §7「アップロード・削除中の並行操作制御」` | `aborts_sequential_upload_and_shows_warning_toast_on_panel_close_during_upload` | Props: 同上。保留添付 3 件をセット。POST `items` → 201 完了、1 件目の添付 POST 進行中にパネル閉じ操作を 3 パターン検証: (a) × ボタン、(b) キャンセルボタン、(c) 明細外クリック（Drawer onClose） | いずれのパターンでも AbortController で進行中のアップロード POST が中断される（AbortSignal が fetch に伝播）。2 件目・3 件目の POST は発火しない。AppToast に「アップロードを中止しました」相当の警告通知が表示される。作成済み明細はロールバック**しない**（DELETE API は呼ばれない）。未アップロード分は編集モードで再アップロード可能な状態（§7「アップロード・削除中の並行操作制御」と同じ方針を準用） |

#### 7-11-4. 追加モードの破棄確認ダイアログ（保留中添付による dirty 判定）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ATT-FE-083 | 単体 | ItemSlidePanel + ItemForm + AttachmentArea | dirty 判定（追加モード） | 正常系 | ATT-F01, ITM-F01 | `50_detail_design/screens/report-detail.md §6「編集中の破棄確認ダイアログ」（dirty 判定対象 -- 追加モードのみ）` | `treats_pending_attachments_as_dirty_in_add_mode` | Props: `mode="add"`, `canModify=true`。フォームフィールドは初期値のまま変更せず、有効な JPEG を 1 件だけ保留 state に追加後、パネル閉じ操作（× ボタン）を実行 | MUI Dialog（「変更を破棄しますか？」）が表示される。Drawer はまだ閉じていない。「破棄」ボタンを押下するとローカル state の保留ファイルが破棄され Drawer がクローズする。「キャンセル」ボタンを押下すると Dialog のみ閉じて保留ファイルは保持される（追加モードは「フィールド変更 1 件以上」OR「保留中添付 1 件以上」で dirty 成立。編集モードの ATT-FE-070「添付操作は dirty 対象外」とは分岐） |

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
| ATT-FE-051 | AttachmentUploader | 正常系（isPending 時 CircularProgress 表示） | 高 |
| ATT-FE-052 | AttachmentUploader | 正常系（hidden input/Button 重複なし） | 中 |
| ATT-FE-053 | AttachmentUploader | 正常系（DnD 視覚フィードバック） | 中 |
| ATT-FE-054 | AttachmentArea | 正常系（削除確認ダイアログ表示） | 高 |
| ATT-FE-055 | AttachmentArea | 正常系（ダイアログキャンセル時 API 未呼出） | 高 |
| ATT-FE-056 | AttachmentArea | 正常系（ダイアログ削除確定 API 呼出） | 高 |
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
| ATT-FE-057 | ItemSlidePanel | 正常系（`isPending \|\| isUploading \|\| isDeleting` で保存ボタン disable） | 高 |
| ATT-FE-058 | ItemSlidePanel / ItemForm | 正常系（アップロード中も閉じる/キャンセル/フィールド編集は有効） | 高 |
| ATT-FE-059 | useUploadAttachment | 正常系（unmount 時 AbortController で upload cancel） | 高 |
| ATT-FE-060 | AttachmentArea + ItemSlidePanel 統合 | 正常系（アップロード中断トースト表示） | 高 |
| ATT-FE-061 | useDeleteAttachment | 正常系（unmount 時 AbortController で delete cancel） | 高 |
| ATT-FE-062 | AttachmentArea + ItemSlidePanel 統合 | 正常系（削除中断トースト表示） | 高 |
| ATT-FE-063 | AttachmentArea + ItemSlidePanel 統合 | 正常系（別明細切替時の進行中 mutation cancel） | 中 |
| ATT-FE-064 | ItemSlidePanel + ItemForm | 正常系（dirty × ボタン → 破棄確認 Dialog 表示） | 高 |
| ATT-FE-065 | ItemSlidePanel + ItemForm | 正常系（dirty キャンセルボタン → Dialog 表示） | 高 |
| ATT-FE-066 | ItemSlidePanel + ItemForm | 正常系（dirty 外クリック → Dialog 表示） | 高 |
| ATT-FE-067 | ItemSlidePanel + ItemForm | 正常系（非 dirty 時は Dialog 非表示・即閉じ） | 高 |
| ATT-FE-068 | ItemSlidePanel + ItemForm | 正常系（Dialog 破棄確定でパネル閉じ・内容破棄） | 高 |
| ATT-FE-069 | ItemSlidePanel + ItemForm | 正常系（Dialog キャンセルでパネル保持・内容保持） | 高 |
| ATT-FE-070 | ItemSlidePanel + ItemForm | 正常系（添付操作のみは dirty とみなさない） | 中 |
| ATT-FE-071 | ItemSlidePanel + ItemForm | 正常系（dirty 時 beforeunload で preventDefault） | 中 |
| ATT-FE-072 | AttachmentArea | 正常系（追加モード + itemId=null で AttachmentArea 表示） | 高 |
| ATT-FE-073 | AttachmentArea / AttachmentUploader | 正常系（追加モードでファイル選択をローカル state に保留） | 高 |
| ATT-FE-074 | AttachmentArea / AttachmentUploader | 異常系（追加モードの MIME / サイズ クライアント側バリデーション） | 高 |
| ATT-FE-075 | AttachmentArea / AttachmentList | 正常系（保留中添付の削除は API 未呼出・確認ダイアログなし） | 高 |
| ATT-FE-076 | AttachmentArea / AttachmentUploader | 正常系（追加モードのアップロード成功トーストが編集モードと同一文言で発火） | 高 |
| ATT-FE-077 | AttachmentArea / AttachmentList | 正常系（保留中添付はダウンロードアイコンのみ非表示・プレビュー/サイズ/削除は編集モードと同一構造） | 高 |
| ATT-FE-078 | AttachmentArea / AttachmentUploader | 正常系（編集モードは即時アップロード挙動不変） | 高 |
| ATT-FE-079 | ItemSlidePanel + ItemForm + AttachmentArea 統合 | 正常系（保存時: 明細作成 → 保留添付を順次アップロード全成功） | 高 |
| ATT-FE-080 | ItemSlidePanel + ItemForm + AttachmentArea 統合 | 異常系（部分失敗時に明細は残し警告トースト → パネルクローズ → 一覧再読み込み） | 高 |
| ATT-FE-081 | ItemSlidePanel + ItemForm | 正常系（順次アップロード中は保存ボタン disabled + 「N/M 件完了」表示 + フォーム readonly） | 高 |
| ATT-FE-082 | ItemSlidePanel + ItemForm + AttachmentArea 統合 | 正常系（順次アップロード中のパネル閉じで AbortController 中断 + 警告トースト） | 高 |
| ATT-FE-083 | ItemSlidePanel + ItemForm + AttachmentArea | 正常系（追加モードは保留中添付を dirty と見なし破棄確認ダイアログ表示） | 高 |
| ATT-FE-084 | AttachmentArea / AttachmentList | 正常系（追加モードの pending 削除トーストが編集モードと同一文言で発火） | 高 |
| ATT-FE-085 | AttachmentArea / AttachmentList | 正常系（追加モードのリスト UI 構造: `<ul><li>` + text 削除ボタン + `<span>` サイズ） | 高 |
| ATT-FE-086 | AttachmentArea / AttachmentList | 正常系（追加モードはダウンロードアイコンのみ非表示・他の要素は編集モードと同一） | 高 |
| ATT-FE-087 | AttachmentArea / AttachmentList | 正常系（add / edit リスト UI 構造の同型性検証、ダウンロードアイコン以外完全一致） | 高 |

**FE テストケース合計: 87件**（優先度「高」: 69件、優先度「中」: 18件）

### FE テスト実装ガイド

#### テストファイル配置

| テストファイル | 対象 |
|-------------|------|
| `src/pages/reports/__tests__/AttachmentArea.test.tsx` | AttachmentArea コンポーネントテスト（ATT-FE-001〜006, ATT-FE-072, 075, 076, 077, 084, 085, 086, 087） |
| `src/pages/reports/__tests__/AttachmentList.test.tsx` | AttachmentList コンポーネントテスト（ATT-FE-007〜015） |
| `src/pages/reports/__tests__/AttachmentUploader.test.tsx` | AttachmentUploader コンポーネントテスト（ATT-FE-016〜028, ATT-FE-051〜053, ATT-FE-073, 074, 078） |
| `src/hooks/__tests__/useAttachments.test.ts` | Hook テスト（ATT-FE-029〜044） |
| `src/pages/reports/__tests__/AttachmentArea.integration.test.tsx` | 統合テスト（ATT-FE-045〜050） |
| `src/pages/reports/__tests__/ItemSlidePanel.test.tsx` | ItemSlidePanel 並行操作・破棄確認テスト（ATT-FE-057, 058, 064〜071, ATT-FE-081, 083） |
| `src/hooks/__tests__/useUploadAttachment.test.ts` | useUploadAttachment 中断テスト（ATT-FE-059） |
| `src/hooks/__tests__/useDeleteAttachment.test.ts` | useDeleteAttachment 中断テスト（ATT-FE-061） |
| `src/pages/reports/__tests__/ItemSlidePanel.integration.test.tsx` | 並行操作中断トースト・明細切替統合テスト（ATT-FE-060, 062, 063）、追加モード保存時の順次アップロード統合テスト（ATT-FE-079, 080, 082） |

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

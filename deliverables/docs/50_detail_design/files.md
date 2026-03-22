# 添付ファイル設計

## 1. 概要

本書は、経費精算SaaS MVP における添付ファイル（領収書）の保存・取得・削除に関する設計を定義する。
ファイル実体は Amazon S3 に保存し、メタデータは PostgreSQL の `attachments` テーブルで管理する。

### 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `10_requirements/requirements.md` | 機能要件 ATT-F01〜F04、ビジネスルール ATT-002〜020 |
| `20_domain/domain_model.md` | Attachment エンティティ定義（3.6） |
| `20_domain/state_machine.md` | 状態別の添付操作可否（5.3） |
| `30_arch/architecture.md` | S3 インフラ構成（2）、添付エンドポイント（5.1） |
| `30_arch/adr/0004-infra.md` | S3 設定・署名付き URL |
| `.claude/rules/security-policy.md` | ファイルセキュリティポリシー（3, 5） |
| `references/glossary.md` | 用語集 |

---

## 2. S3 バケット構成

### 2.1 バケット設計

| 項目 | 値 |
|------|-----|
| バケット数 | 1（全環境共通の命名パターン） |
| バケット名パターン | `expense-saas-attachments-{env}`（例: `expense-saas-attachments-prod`） |
| リージョン | ap-northeast-1（東京） |
| バージョニング | 無効（MVP） |
| 暗号化 | SSE-S3（AES-256、デフォルト暗号化） |
| パブリックアクセス | 全てブロック（Block Public Access 有効） |
| ライフサイクルルール | 2.4 節で定義 |

### 2.2 オブジェクトキーフォーマット

```
{tenant_id}/{report_id}/{attachment_id}
```

| セグメント | 値 | 説明 |
|-----------|-----|------|
| `tenant_id` | UUID | テナント識別子。テナント単位のプレフィックスによりバケットポリシーでの追加制御が将来可能 |
| `report_id` | UUID | 経費レポート識別子。レポート単位での一括操作（削除等）に利用 |
| `attachment_id` | UUID | 添付ファイル識別子。サーバー側で生成する UUID v4 |

**設計判断**:

- `item_id`（明細 ID）はキーに含めない。`attachment_id` は UUID でグローバルにユニークであり、階層を深くする必要がない
- クライアント提供のファイル名は S3 キーに使用しない（パストラバーサル防止、security-policy.md 5 禁止事項）。元ファイル名は `attachments` テーブルの `file_name` カラムに保持する
- ファイル拡張子も S3 キーに含めない。MIME タイプは DB に保持し、ダウンロード時の `Content-Type` は署名付き URL のレスポンスヘッダーオーバーライドで設定する

**キー例**:

```
550e8400-e29b-41d4-a716-446655440000/6ba7b810-9dad-11d1-80b4-00c04fd430c8/f47ac10b-58cc-4372-a567-0e02b2c3d479
```

### 2.3 バケットポリシー

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::expense-saas-attachments-{env}",
        "arn:aws:s3:::expense-saas-attachments-{env}/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**ECS タスクロールの IAM ポリシー**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::expense-saas-attachments-{env}/*"
    }
  ]
}
```

- `s3:ListBucket` は付与しない（オブジェクト一覧はアプリケーション側で DB から取得する）
- S3 操作は全て ECS タスクロール経由。IAM ユーザーのアクセスキーは使用しない

### 2.4 ライフサイクルルール

| ルール | 対象 | アクション | 備考 |
|--------|------|----------|------|
| 論理削除済みファイルの物理削除 | アプリケーションから明示的に削除 | DeleteObject | 6 節で詳述 |

MVP ではライフサイクルポリシーによる自動削除は設定しない。物理削除はアプリケーション層で制御する。

### 2.5 ローカル開発環境

| 項目 | 値 |
|------|-----|
| S3 互換サービス | MinIO（Docker Compose） |
| エンドポイント | `http://localhost:9000` |
| バケット名 | `expense-saas-attachments-local` |
| アクセスキー | 環境変数で設定（`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`） |

Go の S3 クライアント初期化時にエンドポイント URL を環境変数で切り替え可能にする。

---

## 3. アップロードフロー

### 3.1 方式

**API 経由プロキシ方式**を採用する。クライアントはファイルを API サーバーに `multipart/form-data` で送信し、サーバーがバリデーション後に S3 へアップロードする。

| 方式 | 採用 | 理由 |
|------|------|------|
| API 経由プロキシ | **採用** | サーバー側でバリデーション（MIME タイプ、マジックバイト、サイズ）を完全に制御可能。MVP のシンプルさを優先 |
| 署名付きアップロード URL | 不採用 | クライアントが S3 に直接アップロードするため、サーバー側でのファイル内容検証が困難。将来的な大容量対応時に検討 |

### 3.2 シーケンス

```
クライアント                     API サーバー                        S3                DB
    |                              |                              |                  |
    |-- POST /api/reports/:id/     |                              |                  |
    |   items/:itemId/attachments  |                              |                  |
    |   [multipart/form-data]      |                              |                  |
    |   Authorization: Bearer xxx  |                              |                  |
    |                              |                              |                  |
    |                              |-- [1] JWT 検証 + ロール検証    |                  |
    |                              |-- [2] テナントコンテキスト設定   |                  |
    |                              |                              |                  |
    |                              |-- [3] レポート取得 --------------------------->|
    |                              |<-- report (status, tenant_id, user_id) --------|
    |                              |                              |                  |
    |                              |-- [4] 認可チェック             |                  |
    |                              |   - 同一テナントか            |                  |
    |                              |   - 所有者か                  |                  |
    |                              |   - draft 状態か              |                  |
    |                              |                              |                  |
    |                              |-- [5] 明細の存在確認 ----------------------->|
    |                              |<-- item (report_id 一致確認) ------------------|
    |                              |                              |                  |
    |                              |-- [6] ファイルバリデーション    |                  |
    |                              |   - サイズ <= 5MB             |                  |
    |                              |   - MIME タイプ検証            |                  |
    |                              |   - マジックバイトチェック      |                  |
    |                              |                              |                  |
    |                              |-- [7] UUID 生成              |                  |
    |                              |   attachment_id = uuid.New() |                  |
    |                              |                              |                  |
    |                              |-- [8] S3 アップロード ------->|                  |
    |                              |   Key: {tenant_id}/          |                  |
    |                              |        {report_id}/          |                  |
    |                              |        {attachment_id}        |                  |
    |                              |   ContentType: {mime_type}    |                  |
    |                              |<-- 成功 -----------------------|                  |
    |                              |                              |                  |
    |                              |-- [9] メタデータ保存 --------------------------->|
    |                              |   INSERT attachments          |                  |
    |                              |<-- attachment record ----------------------------|
    |                              |                              |                  |
    |<-- 201 Created --------------|                              |                  |
    |   {data: {attachment}}       |                              |                  |
```

### 3.3 エンドポイント

```
POST /api/reports/:id/items/:itemId/attachments
```

**リクエスト**:

| 項目 | 値 |
|------|-----|
| Content-Type | `multipart/form-data` |
| パート名 | `file` |
| 認証 | 必須（Bearer トークン） |
| 必要ロール | Member, Approver, Admin, Accounting（所有者のみ） |

**レスポンス（201 Created）**:

```json
{
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "item_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "file_name": "receipt_20260315.jpg",
    "file_size": 245760,
    "mime_type": "image/jpeg",
    "created_at": "2026-03-22T10:30:00Z"
  }
}
```

**エラーレスポンス**:

| HTTP ステータス | エラーコード | 条件 |
|----------------|-------------|------|
| 400 | INVALID_REQUEST | ファイルパートが存在しない |
| 404 | RESOURCE_NOT_FOUND | レポートまたは明細が見つからない（テナント境界越え含む） |
| 413 | FILE_TOO_LARGE | ファイルサイズが 5MB を超過（ATT-003） |
| 422 | INVALID_FILE_TYPE | 許可されていない MIME タイプ（ATT-002） |
| 422 | REPORT_NOT_EDITABLE | レポートが draft 状態でない（ATT-020） |
| 403 | PERMISSION_DENIED | 所有者でない |
| 429 | RATE_LIMIT_EXCEEDED | ファイルアップロードレート制限超過（10 req/min/user） |

### 3.4 リクエストサイズ制限

| レイヤー | 制限 | 設定箇所 |
|---------|------|---------|
| ALB | 受信ボディ上限をデフォルト（1MB 以上可）で運用 | ALB 設定 |
| Go HTTP サーバー | `http.MaxBytesReader` で **6MB** に制限 | ミドルウェアまたはハンドラ |
| アプリケーション | ファイルサイズ **5MB** チェック | ハンドラ層 |

Go サーバーの制限を 6MB とするのは、`multipart/form-data` のヘッダー・境界文字列のオーバーヘッドを考慮するため。実際のファイルサイズバリデーション（5MB 以下）はアプリケーション層で行う。

### 3.5 メモリ管理

ファイルデータをメモリに全て読み込むのではなく、`io.Reader` をストリーミングで S3 に転送する。

```
[multipart.Part] --io.Reader--> [バリデーション（先頭バイト読み取り）] --io.Reader--> [S3 PutObject]
```

- マジックバイトチェックのために先頭 512 バイトを読み取り、`io.MultiReader` で再結合して S3 に送信する
- 5MB のファイルを同時に複数処理してもメモリ使用量を抑制できる

---

## 4. ダウンロードフロー

### 4.1 方式

署名付き GET URL を発行し、クライアントが S3 から直接ダウンロードする。API サーバーはプロキシしない。

### 4.2 シーケンス

```
クライアント                     API サーバー                        S3                DB
    |                              |                              |                  |
    |-- GET /api/reports/:id/      |                              |                  |
    |   items/:itemId/             |                              |                  |
    |   attachments/:attId         |                              |                  |
    |   Authorization: Bearer xxx  |                              |                  |
    |                              |                              |                  |
    |                              |-- [1] JWT 検証 + ロール検証    |                  |
    |                              |-- [2] テナントコンテキスト設定   |                  |
    |                              |                              |                  |
    |                              |-- [3] 添付メタデータ取得 --------------------->|
    |                              |<-- attachment (s3_key, tenant_id 等) ----------|
    |                              |                              |                  |
    |                              |-- [4] 認可チェック             |                  |
    |                              |   - 同一テナントか            |                  |
    |                              |   - 閲覧権限があるか          |                  |
    |                              |     (所有者 or 権限ロール)    |                  |
    |                              |                              |                  |
    |                              |-- [5] 署名付き URL 生成       |                  |
    |                              |   有効期限: 15分              |                  |
    |                              |   ResponseContentDisposition  |                  |
    |                              |   ResponseContentType         |                  |
    |                              |                              |                  |
    |<-- 200 OK ------------------|                              |                  |
    |   {data: {download_url,      |                              |                  |
    |    file_name, expires_at}}   |                              |                  |
    |                              |                              |                  |
    |-- GET {download_url} --------|----------------------------->|                  |
    |<-- ファイルデータ ------------|-------------------------------|                  |
```

### 4.3 エンドポイント

```
GET /api/reports/:id/items/:itemId/attachments/:attId
```

**レスポンス（200 OK）**:

```json
{
  "data": {
    "download_url": "https://expense-saas-attachments-prod.s3.ap-northeast-1.amazonaws.com/...",
    "file_name": "receipt_20260315.jpg",
    "mime_type": "image/jpeg",
    "file_size": 245760,
    "expires_at": "2026-03-22T10:45:00Z"
  }
}
```

**エラーレスポンス**:

| HTTP ステータス | エラーコード | 条件 |
|----------------|-------------|------|
| 404 | RESOURCE_NOT_FOUND | 添付ファイルが見つからない（テナント境界越え含む） |
| 403 | PERMISSION_DENIED | 閲覧権限がない |

### 4.4 署名付き URL の生成パラメータ

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| 有効期限 | 15分（ATT-012） | `time.Duration(15 * time.Minute)` |
| ResponseContentDisposition | `attachment; filename="{file_name}"` | ブラウザでのダウンロード時に元ファイル名を使用 |
| ResponseContentType | `{mime_type}` | DB に保存された MIME タイプ |

### 4.5 認可チェック（ATT-011）

署名付き URL 発行前に以下を検証する。

| # | チェック | 失敗時 | 責務 |
|---|---------|--------|------|
| 1 | JWT が有効 | 401 Unauthorized | Auth ミドルウェア |
| 2 | 添付が同一テナント | 404 Not Found（TNT-006） | リポジトリ層（RLS + WHERE tenant_id） |
| 3 | 閲覧権限 | 403 Forbidden | ハンドラ層 |

**閲覧権限のルール**:

| ロール | 閲覧可能範囲 |
|--------|------------|
| Member | 自分が作成したレポートの添付のみ |
| Approver | 自分のレポート + 同一テナントの submitted レポートの添付 + 自分が承認/却下したレポートの添付 |
| Accounting | 自分のレポート + 同一テナントの全レポートの添付 |
| Admin | 同一テナントの全レポートの添付 |

ダウンロードの認可ルールはレポート詳細閲覧の認可ルールと同一とする。レポートを閲覧できるユーザーは、そのレポートに紐づく全ての添付をダウンロードできる。

---

## 5. 添付ファイル一覧取得

### 5.1 エンドポイント

```
GET /api/reports/:id/items/:itemId/attachments
```

**レスポンス（200 OK）**:

```json
{
  "data": [
    {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "item_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
      "file_name": "receipt_20260315.jpg",
      "file_size": 245760,
      "mime_type": "image/jpeg",
      "created_at": "2026-03-22T10:30:00Z"
    }
  ]
}
```

一覧取得では署名付き URL を含めない。各ファイルのダウンロード URL は個別に取得する（署名付き URL の不要な大量生成を避ける）。

---

## 6. 削除フロー

### 6.1 論理削除（DB）

```
DELETE /api/reports/:id/items/:itemId/attachments/:attId
```

**事前条件**:

| # | 条件 | 失敗時 | ルールID |
|---|------|--------|---------|
| 1 | レポートが draft 状態 | 422 REPORT_NOT_EDITABLE | ATT-020 |
| 2 | 所有者である | 403 PERMISSION_DENIED | RBC-010 |
| 3 | 同一テナント | 404 RESOURCE_NOT_FOUND | TNT-006 |

**処理**:

1. `attachments` テーブルの `deleted_at` に現在日時を設定（論理削除、DAT-002）
2. S3 オブジェクトは即時削除しない（6.2 節参照）

**レスポンス（204 No Content）**: ボディなし。

### 6.2 S3 物理削除戦略

論理削除されたファイルの S3 物理削除は以下の方針で行う。

| 方式 | 採用 | 説明 |
|------|------|------|
| 即時削除 | 不採用 | DB トランザクション失敗時にファイルだけ削除される不整合リスクがある |
| バッチ削除 | **採用（MVP）** | 定期バッチで `deleted_at IS NOT NULL` かつ一定期間経過した添付の S3 オブジェクトを削除 |
| イベント駆動削除 | 不採用（Phase 3 検討） | SQS/Lambda による非同期削除。MVP では過剰 |

**バッチ削除の仕様**:

| 項目 | 値 |
|------|-----|
| 実行間隔 | 日次（cron or ECS Scheduled Task） |
| 猶予期間 | 7日（`deleted_at` から7日経過後に物理削除） |
| 処理内容 | (1) DB から対象レコード取得 → (2) S3 DeleteObject → (3) DB からレコード物理削除 |
| エラー処理 | S3 削除失敗時はスキップし次回バッチで再試行。ログに記録 |

**猶予期間の理由**: 誤操作からの復旧猶予。論理削除後7日間は S3 にファイルが残るため、必要に応じてDBレコードの `deleted_at` を NULL に戻すことで復元可能。

### 6.3 レポート削除時の連動削除

レポートが削除（論理削除）された場合、関連する全ての添付ファイルも連動して論理削除される（state_machine.md T5 事後処理）。

```
[レポート削除 (draft のみ)]
  |
  +--> expense_reports.deleted_at = now()
  +--> 関連 expense_items.deleted_at = now()
  +--> 関連 attachments.deleted_at = now()
  |
  [7日後: バッチ削除]
  |
  +--> S3 オブジェクト物理削除
  +--> attachments レコード物理削除
```

---

## 7. MIME タイプ検証

### 7.1 許可する MIME タイプ

| MIME タイプ | 拡張子 | マジックバイト（先頭） |
|------------|--------|---------------------|
| `image/jpeg` | .jpg, .jpeg | `FF D8 FF` |
| `image/png` | .png | `89 50 4E 47 0D 0A 1A 0A` |
| `application/pdf` | .pdf | `25 50 44 46`（`%PDF`） |

### 7.2 検証手順

アップロード時に以下の順序で検証する。全て合格した場合のみ S3 にアップロードする。

```
[1] Content-Type ヘッダーの確認
    multipart パートの Content-Type が許可リストに含まれるか
    |
    +--> 不一致: 422 INVALID_FILE_TYPE
    |
[2] マジックバイトチェック
    ファイル先頭 512 バイトを読み取り、実際のファイル形式を判定
    Go 標準ライブラリ http.DetectContentType() を使用
    |
    +--> 不一致: 422 INVALID_FILE_TYPE
    |
[3] Content-Type とマジックバイトの整合性確認
    [1] と [2] の結果が一致するか
    |
    +--> 不一致: 422 INVALID_FILE_TYPE
    |
[4] 合格 → S3 アップロードへ
```

**設計判断**: 拡張子のみの判定は行わない。ファイル拡張子はクライアントが自由に変更できるため、マジックバイトチェックを必須とする（security-policy.md 5）。

### 7.3 ファイルサイズ検証

| チェックポイント | 値 | 説明 |
|----------------|-----|------|
| HTTP リクエストボディ上限 | 6MB | `http.MaxBytesReader` で制限。multipart ヘッダーのオーバーヘッドを考慮 |
| ファイルサイズ上限 | 5MB（5,242,880 バイト） | アプリケーション層でパート読み取り時に検証（ATT-003） |

サイズ超過時は 413 FILE_TOO_LARGE を返却する。

---

## 8. attachments テーブルとの連携

### 8.1 テーブル定義（概要）

`attachments` テーブルの詳細は `db_schema.md`（T1-6）で定義する。本設計書では S3 連携に関連するカラムを記載する。

| カラム | 型 | 説明 |
|--------|-----|------|
| attachment_id | UUID | PK。S3 キーの一部として使用 |
| item_id | UUID | FK → expense_items。所属明細 |
| report_id | UUID | FK → expense_reports。所属レポート（冗長保持） |
| tenant_id | UUID | FK → tenants。テナント（冗長保持: RLS + S3 キー） |
| file_name | String | クライアント提供の元ファイル名。S3 キーには使用しない |
| file_size | Integer | ファイルサイズ（バイト）。5MB 以下 |
| mime_type | String | 検証済みの MIME タイプ |
| s3_key | String | S3 オブジェクトキー（`{tenant_id}/{report_id}/{attachment_id}`） |
| created_at | Timestamp | 作成日時 |
| deleted_at | Timestamp | 論理削除日時（NULL: 有効、非NULL: 削除済み） |

**注記**: `domain_model.md` では `s3_path` となっているが、実装上は `s3_key` を使用する（S3 のオブジェクトキーを格納するカラムであることを明示するため）。DB スキーマ設計（T1-6）で最終的なカラム名を確定する。

### 8.2 DB と S3 の整合性

アップロード時の処理順序:

```
[1] ファイルバリデーション（メモリ上）
[2] S3 PutObject（ファイル実体の保存）
[3] DB INSERT（メタデータの保存）
```

**S3 アップロード成功 → DB INSERT 失敗の場合**: S3 に孤立ファイルが残る。この不整合は日次バッチで検出・削除する。

**孤立ファイル検出バッチ**:

| 項目 | 値 |
|------|-----|
| 実行間隔 | 日次（物理削除バッチと同時実行） |
| 検出方法 | S3 オブジェクトキーから `attachment_id` を抽出し、DB に該当レコードが存在しないものを検出 |
| 対象 | 作成から 24 時間以上経過した孤立ファイル |
| 処理 | S3 DeleteObject + ログ記録 |

---

## 9. レート制限

ファイルアップロードには専用のレート制限を適用する（security-policy.md 4）。

| 対象 | 制限値 | 単位 | キー |
|------|--------|------|------|
| ファイルアップロード | 10 req/min | ユーザー単位 | user_id |

レート制限超過時は 429 Too Many Requests を返却し、`Retry-After` ヘッダーで再試行可能時刻を通知する。

---

## 10. セキュリティ考慮事項

### 10.1 パストラバーサル防止

- S3 キーにクライアント提供のファイル名を使用しない
- S3 キーは全てサーバー側で生成した UUID で構成する
- 元ファイル名は DB に保存し、ダウンロード時の `Content-Disposition` ヘッダーでのみ使用する

### 10.2 テナント分離

- S3 キーの先頭セグメントに `tenant_id` を含める
- DB クエリは RLS + WHERE tenant_id で二重保証
- 他テナントの添付へのアクセス試行には 404 Not Found を返却（TNT-006: 存在漏洩防止）

### 10.3 署名付き URL のセキュリティ

- 有効期限は 15 分（ATT-012）。長期間のキャッシュや共有を防止
- 発行前に必ず認可チェックを実施（ATT-011）
- 署名付き URL にはダウンロード専用の権限のみ付与（GetObject のみ）

### 10.4 アップロードコンテンツの安全性

- マジックバイトチェックで MIME タイプ偽装を検出
- ウイルスチェックは MVP スコープ外（requirements.md 5 やらないこと）
- アップロードされたファイルはサーバー側で実行しない（S3 に保存するのみ）

### 10.5 バケットのパブリックアクセス防止

- S3 Block Public Access を全て有効化
- バケットポリシーで HTTPS 以外のアクセスを拒否
- 署名付き URL 経由でのみファイルにアクセス可能

---

## 11. エラー一覧

添付ファイル操作で発生するエラーの一覧。

| HTTP ステータス | エラーコード | 発生条件 | ルールID |
|----------------|-------------|---------|---------|
| 400 | INVALID_REQUEST | ファイルパートが存在しない / リクエスト形式不正 | - |
| 401 | UNAUTHORIZED | JWT が無効または期限切れ | SEC-003 |
| 403 | PERMISSION_DENIED | 所有権不足または閲覧権限なし | RBC-003, RBC-010 |
| 404 | RESOURCE_NOT_FOUND | リソースが見つからない（テナント境界越え含む） | TNT-006 |
| 413 | FILE_TOO_LARGE | ファイルサイズが 5MB を超過 | ATT-003 |
| 422 | INVALID_FILE_TYPE | 許可されていない MIME タイプ | ATT-002, ATT-013 |
| 422 | REPORT_NOT_EDITABLE | レポートが draft 状態でない | ATT-020 |
| 429 | RATE_LIMIT_EXCEEDED | レート制限超過 | SEC-012 |

---

## 12. 他タスクとの接点

| タスク | 接点 | 本設計書での扱い |
|--------|------|----------------|
| T1-6（db_schema.md） | `attachments` テーブルのスキーマ定義 | 8 節で S3 連携に関連するカラムの概要を記載。DDL は db_schema.md で確定 |
| T1-8（security.md） | バケットポリシー、ファイルアップロードのセキュリティ | 10 節でファイル固有のセキュリティ考慮事項を記載。全体のセキュリティ設計は security.md で統括 |
| T1-3（screens/report-detail.md） | レポート詳細画面の添付 UI（アップロード・ダウンロード・削除） | 本設計書は API・バックエンド・インフラの設計。UI 仕様は screens/report-detail.md で定義 |

---

## 13. 上流成果物との差分記録

| 項目 | 上流の記載 | 本設計書での対応 | 理由 |
|------|-----------|----------------|------|
| S3 キーの `item_id` 有無 | requirements.md ATT-014: `{tenant_id}/{report_id}/{attachment_id}` | `item_id` を含めない（`{tenant_id}/{report_id}/{attachment_id}`）| attachment_id は UUID でユニーク。item_id 階層は不要 |
| DB カラム名 | domain_model.md: `s3_path` | 本設計書では `s3_key` を推奨 | S3 のオブジェクトキーを格納することを明示。最終確定は db_schema.md |

---

## 14. 品質チェック

- [x] S3 バケット構成が定義されているか（単一バケット、キーフォーマット、暗号化、パブリックアクセスブロック）
- [x] アップロードフローが API プロキシ方式で設計されているか（判断ポイント適用）
- [x] ダウンロードフローが署名付き GET URL 方式で設計されているか
- [x] 署名付き URL の有効期限が 15 分であること（ATT-012）
- [x] MIME タイプ検証にマジックバイトチェックが含まれているか（ATT-013）
- [x] ファイルサイズ上限が 5MB であること（ATT-003）
- [x] 署名付き URL 発行前の認可チェックが定義されているか（ATT-011）
- [x] 論理削除と S3 物理削除の戦略が定義されているか
- [x] クライアント提供のファイル名を S3 キーに使用していないこと
- [x] S3 キーが `{tenant_id}/{report_id}/{attachment_id}` であること（判断ポイント + issue 036）
- [x] テナント分離が S3 キー + DB RLS で二重保証されているか
- [x] 上流成果物との差分が記録されているか
- [x] 用語が glossary.md と一致しているか
- [x] MVP スコープ内に収まっているか

# 署名付きダウンロード URL が Docker 内部ホスト名 `minio:9000` で生成されホストブラウザから到達不能

## 発見日
2026-04-14

## カテゴリ
implementation / backend / infrastructure / local-dev

## 影響度
**高（SMK-037 ブロッカー、ローカル開発で添付ダウンロード一切不可）**

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 + SMK-030 実施中、アップロードされた添付ファイル名をクリックしてダウンロードを試みたところ、ブラウザ（Edge）が `http://minio:9000/...` の署名付き URL を解決できず `ERR_NAME_NOT_RESOLVED` で失敗することを確認

## 関連ステップ
Step 7（運用設計）/ Step 8-11（ローカル開発環境統合）/ Step 10-G（添付ファイル）/ Step 11-A（ローカル動作確認）

## ブロッカー
**あり** — SMK-037（ダウンロード起動）が実施不可能。ローカルデモでも添付ファイルのダウンロードが機能しない致命的問題

## 概要

API バックエンドが生成する S3 署名付きダウンロード URL が Docker 内部ホスト名 `minio` を参照している。ホストブラウザは `minio` という DNS 名を解決できないため、ダウンロードリンクをクリックすると `ERR_NAME_NOT_RESOLVED` エラーが発生する。

観測された URL 例:
```
http://minio:9000/expense-saas-receipts-dev/aaaaaaaa-0001-0001-0001-000000000001/cccccccc-0001-0001-0001-000000000001/ffffffff-0002-0002-0002-000000000002?X-Amz-Algorithm=AWS4-HMAC-SHA256&...
```

## 事実

### docker-compose.yml

`expense-saas/docker-compose.yml` L120, L153:
```yaml
S3_ENDPOINT: ${S3_ENDPOINT:-http://minio:9000}
```

API サービスと seed サービスが同じ内部ホスト名 `minio:9000` を使用。

### S3 クライアント実装

`expense-saas/internal/pkg/s3/client.go` L55-75:

```go
var s3Opts []func(*s3.Options)

endpoint := os.Getenv("S3_ENDPOINT")
if endpoint != "" {
    s3Opts = append(s3Opts, func(o *s3.Options) {
        o.BaseEndpoint = aws.String(endpoint)
        o.UsePathStyle = true
    })
}

client := s3.NewFromConfig(cfg, s3Opts...)
presigner := s3.NewPresignClient(client)  // ← 同じ client から生成
```

**PresignClient が Upload と同じ BaseEndpoint を使う**ため、署名付き URL も `http://minio:9000/...` で生成される。

### 生成挙動

- Upload: API コンテナ内から `http://minio:9000` に PutObject → Docker ネットワーク内の名前解決が効くので成功
- PresignGetObject: 同じ BaseEndpoint を使って URL 生成 → `http://minio:9000/...` を返す
- ホストブラウザでダウンロードクリック: `minio` の名前解決不可 → `ERR_NAME_NOT_RESOLVED`

### 本番環境への影響

本番（ECS + AWS S3）では `S3_ENDPOINT` を未設定（もしくは AWS 公式ホスト名）にするため、本問題は発生しない。**ローカル開発環境専用のバグ**。

## 修正方針

AWS SDK Go v2 の PresignClient に独立したエンドポイント設定を渡す。

### 案 A: `S3_PUBLIC_ENDPOINT` 環境変数を追加（推奨）

**コード変更** (`internal/pkg/s3/client.go`):

```go
// 通常クライアント用オプション（内部通信）
var s3Opts []func(*s3.Options)
endpoint := os.Getenv("S3_ENDPOINT")
if endpoint != "" {
    s3Opts = append(s3Opts, func(o *s3.Options) {
        o.BaseEndpoint = aws.String(endpoint)
        o.UsePathStyle = true
    })
}
client := s3.NewFromConfig(cfg, s3Opts...)

// PresignClient 用オプション（公開アクセス用）
var presignOpts []func(*s3.Options)
publicEndpoint := os.Getenv("S3_PUBLIC_ENDPOINT")
if publicEndpoint == "" {
    publicEndpoint = endpoint  // 未設定時は S3_ENDPOINT にフォールバック
}
if publicEndpoint != "" {
    presignOpts = append(presignOpts, func(o *s3.Options) {
        o.BaseEndpoint = aws.String(publicEndpoint)
        o.UsePathStyle = true
    })
}
presignClient := s3.NewPresignClient(
    s3.NewFromConfig(cfg, presignOpts...),
)
```

**docker-compose.yml 変更** (api service の environment セクション):

```yaml
S3_ENDPOINT: ${S3_ENDPOINT:-http://minio:9000}
S3_PUBLIC_ENDPOINT: ${S3_PUBLIC_ENDPOINT:-http://localhost:9000}
```

**env_config.md 更新**: `S3_PUBLIC_ENDPOINT` を新規追加、本番では未設定（AWS S3 のデフォルトホスト使用）の旨を記載。

### 案 B: extra_hosts で `minio` をホスト IP にマッピング

- ブラウザから Docker サービス名を解決できるようにする方法だが、host 側 DNS や hosts ファイル操作が必要で fragile
- 非推奨

### 案 C: nginx リバースプロキシ経由

- 過剰設計。ローカル dev スコープに対して重すぎる
- 非推奨

### 推奨

**案 A**。理由:
- AWS SDK v2 の標準機能を使うクリーンな実装
- 本番環境では `S3_PUBLIC_ENDPOINT` 未設定で従来通り動作（フォールバックあり）
- `docker-compose.yml` に 1 行追加するだけで localhost 経由で解決

## 修正対象ファイル

- `expense-saas/internal/pkg/s3/client.go` — PresignClient に独立 endpoint 設定を追加
- `expense-saas/internal/pkg/s3/client_test.go`（もしあれば）
- `expense-saas/docker-compose.yml` — api service に `S3_PUBLIC_ENDPOINT` 追加
- `dev-journal/deliverables/docs/70_operations/env_config.md` — 新規環境変数の説明追加
- `expense-saas/.env.example`（もしあれば）

## 完了条件

- ローカル環境で添付ファイル名をクリックするとホストブラウザでダウンロードが正常開始される
- 署名付き URL が `http://localhost:9000/...` 形式で生成される
- 本番環境（AWS S3）では `S3_PUBLIC_ENDPOINT` 未設定で従来通り動作する
- `env_config.md` に `S3_PUBLIC_ENDPOINT` の説明が追加されている
- SMK-037 が PASS 判定できる

## 関連 issue

- 077: S3_PRESIGNED_URL_EXPIRY 実装未参照（PR #44 レビューで顕在化）
- 078: S3 関連変数名の設計書・実装乖離（S3_REGION vs AWS_REGION、PR #44 レビューで顕在化）
- 079: env_config.md §4.x 全変数の棚卸し

本 issue は上記 S3 系 issue と合わせて対応するとコンフィグ整理が一度で済む可能性がある。

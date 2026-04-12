# 設計書で定義された S3_PRESIGNED_URL_EXPIRY が実装で環境変数参照されていない

## 発見日
2026-04-12

## カテゴリ
implementation

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 5（詳細設計）、Step 8（基盤構築）、Step 10-G（添付ファイル機能）

## ブロッカー
なし

## 問題

`70_operations/env_config.md` §4.4 L117 で環境変数 `S3_PRESIGNED_URL_EXPIRY=15m`（dev/stg/prod 共通）が定義されているが、実装コード側でこの環境変数を参照している箇所が存在しない。

- 実装: `internal/service/attachment_service.go:190` で `s.storage.PresignGetObject(ctx, att.S3Key, att.FileName, string(att.MimeType), attachmentDownloadExpiry)` と呼び出している
- `attachmentDownloadExpiry` は Go 定数（ハードコード値）であり、環境変数から設定できない
- `internal/pkg/s3/client.go` の `PresignGetObject` は `expiry time.Duration` を引数で受けるだけで、`os.Getenv("S3_PRESIGNED_URL_EXPIRY")` を読んでいない

結果として、設計書定義された環境変数が実装で無視されており、運用環境ごとに署名付き URL の有効期限を変更できない。

## 影響

- dev/stg/prod で URL 有効期限を環境変数で切り替えられない（現状は全環境で同じ定数）
- 設計書（env_config.md）と実装の乖離が残り、設計書を信頼できなくなる
- 将来、セキュリティ要件で有効期限を短縮する場合、コード変更＋再デプロイが必要

本問題は Step 11-A ローカル動作確認のブロッカーではない（既存の定数ベースで動作する）。Step 8-11 対応 PR #44 のレビュー時に発覚。

## 提案

1. `internal/pkg/config`（または適切な設定モジュール）で `S3_PRESIGNED_URL_EXPIRY` 環境変数を読み込む
2. `attachment_service.go` の `attachmentDownloadExpiry` 定数を設定値に置き換える
3. `.env.example` と `docker-compose.yml` に `S3_PRESIGNED_URL_EXPIRY=15m` を追記
4. 単体テストで環境変数設定時の挙動を検証
5. デフォルト値（環境変数未設定時）は現行の Go 定数値と一致させる（破壊的変更を避ける）

本 issue は Step 10-G（添付ファイル機能）の遡及改修として扱うのが自然。ただし影響度は低いため、優先度は Step 11 完了後でよい。

# S3 関連環境変数名が設計書と実装コードで乖離している

## 発見日
2026-04-12

## カテゴリ
implementation

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 7（運用設計）、Step 8-1（開発環境構築）、Step 10-G（添付ファイル機能）

## ブロッカー
なし

## 問題

`70_operations/env_config.md` §4.4 で定義された S3 関連環境変数名と、実装コード `internal/pkg/s3/client.go` で参照される変数名が乖離している。

| 役割 | env_config.md §4.4 の定義 | `client.go` の参照 |
|------|---------------------------|---------------------|
| リージョン | `S3_REGION` (L115) | `AWS_REGION` (L33) |
| バケット名 | `S3_BUCKET` (L114) | `S3_BUCKET` (L73) ✅ 一致 |
| エンドポイント | `S3_ENDPOINT` (L116) | `S3_ENDPOINT` (L58) ✅ 一致 |
| アクセスキー | `AWS_ACCESS_KEY_ID` (L118) | `AWS_ACCESS_KEY_ID` (L42) ✅ 一致 |
| シークレットキー | `AWS_SECRET_ACCESS_KEY` (L119) | `AWS_SECRET_ACCESS_KEY` (L43) ✅ 一致 |

乖離しているのは **リージョン変数名** のみ（`S3_REGION` vs `AWS_REGION`）。

## 背景

この乖離は Step 8 以前から存在する。AWS SDK v2 は `AWS_REGION` を標準的に参照するため、実装側では SDK の慣例に従い `AWS_REGION` を採用している。一方、env_config.md は `S3_` プレフィックスで統一性を優先している。

本問題は Step 8-11 対応 PR #44 のレビュー時に顕在化した。PR #44 では実装に合わせて `AWS_REGION=ap-northeast-1` を設定したが、設計書との文字通りの乖離は残っている。

## 影響

- 設計書を見て環境変数を設定しようとする開発者が `S3_REGION` を設定しても無視される（実装が `AWS_REGION` を読むため）
- 運用ドキュメント（release.md 等）で `S3_REGION` を参照している箇所があれば連鎖的に不整合

本問題は Step 11-A ローカル動作確認のブロッカーではない（AWS_REGION を docker-compose で設定済み）。

## 提案

以下のいずれかを採用する。判断はユーザーに委ねる。

### 方針 A: 実装を設計書に寄せる（`AWS_REGION` → `S3_REGION`）

- `internal/pkg/s3/client.go` を `os.Getenv("S3_REGION")` に変更
- `.env.example`、`docker-compose.yml`、および全運用ドキュメントで `AWS_REGION` を `S3_REGION` に置換
- メリット: 設計書の統一性を維持
- デメリット: AWS SDK の慣例から外れる。`AWS_REGION` を環境変数にセットするデフォルト挙動と二重管理になるおそれ

### 方針 B: 設計書を実装に寄せる（`S3_REGION` → `AWS_REGION`）

- `env_config.md` §4.4 L115 を `AWS_REGION` に変更
- 他の運用ドキュメントで `S3_REGION` を参照している箇所も併せて修正
- メリット: AWS SDK v2 の慣例に従う自然な実装
- デメリット: 設計書の「`S3_` プレフィックス統一」という暗黙ルールが崩れる

### 推奨: 方針 B

AWS SDK v2 の標準挙動と合致するため、方針 B を推奨する。ただし影響範囲は軽微で、ユーザー判断で A を選ぶことも妥当。

## 関連

- PR #44（Step 8-11 ローカル開発環境統合）: 本 issue が顕在化したきっかけ。PR #44 では `AWS_REGION=ap-northeast-1` を設定しており、動作上は問題なし
- 本 issue と並行して起票: issue 077（`S3_PRESIGNED_URL_EXPIRY` が実装で環境変数参照されていない）

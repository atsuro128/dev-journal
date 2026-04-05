# JWT 鍵ファイルが未設定でもサーバーが起動する

## 発見日
2026-04-02

## カテゴリ
security

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 8（基盤構築）

## 問題

`cmd/server/main.go` のステップ 5（JWT Verifier 初期化）で、鍵ファイルが未設定または読み込み失敗の場合に Warn ログを出して起動を続行する設計になっている。

開発用の鍵ファイル（`keys/private.pem`, `keys/public.pem`）は既にリポジトリに含まれているため、省略可能にする理由がない。本番で鍵ファイルの配置を忘れた場合、サーバーは正常起動しヘルスチェックも通るが、認証エンドポイントが全て 401 を返す。実際にリクエストを送るまで問題に気づけない。

## 影響

- 本番デプロイ時の設定漏れに気づくのが遅れる
- サーバーは起動しているのに認証が機能しない状態が発生しうる

## 提案

`JWT_PUBLIC_KEY_PATH`（および `JWT_PRIVATE_KEY_PATH`）を必須の環境変数に変更し、未設定または読み込み失敗時は DB 接続失敗と同様に `os.Exit(1)` で即終了させる。

修正箇所:
- `cmd/server/main.go` — Warn → Error + os.Exit(1)
- `internal/config/config.go` — バリデーション追加

---

## 解決内容

`JWT_PRIVATE_KEY_PATH` と `JWT_PUBLIC_KEY_PATH` を必須環境変数に変更。

- `internal/config/config.go`: 未設定時にエラーを返すバリデーションを追加
- `cmd/server/main.go`: JWT Verifier 初期化失敗時に Warn → Error + `os.Exit(1)` に変更。nil チェック分岐を削除

## 解決日
2026-04-05

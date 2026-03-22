# 044: auth-login のフローチャートが API 契約にない失敗分岐を追加している

## 指摘概要
`screens/auth-login.md` に追加された認証フローチャートが、同一ファイルの API 仕様・`openapi.yaml`・既存シーケンス図に存在しない失敗分岐を新たに定義している。具体的には `422 バリデーションエラー` と `TenantMembership 取得失敗 -> 401 認証失敗` が図に追加されているが、ログイン API の契約は 200/401/429 のみで、テナントメンバーシップ取得失敗時の扱いも他文書で定義されていない。詳細設計で図だけが追加の失敗契約を持つ状態になっており、実装・テスト・OpenAPI の解釈が分岐する。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/screens/auth-login.md:288-296`
  - `B -->|No| E1[422 バリデーションエラー]`
  - `F -->|No| E3[401 認証失敗]`
- `dev-journal/deliverables/docs/50_detail_design/screens/auth-login.md:220-226`
  - API リクエスト/レスポンス表では `200` / `401` / `429` しか定義していない
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:159-181`
  - `POST /api/auth/login` のレスポンス定義は `200` / `401` / `429` のみ
- `dev-journal/deliverables/docs/50_detail_design/screens/auth-login.md:245-273`
  - 既存シーケンス図には入力バリデーションはあるが `422` のレスポンス分岐はなく、`tenant_memberships` 取得失敗時の分岐も定義されていない

## 判定
重大度: 中
分類: 内部整合性不備 / API 契約の不整合

## 修正方針案
以下のどちらかに揃える。

1. 図を既存契約に合わせる。
   `422` と `TenantMembership 取得失敗` の失敗分岐を削除し、SEC-011 の同一エラー合流だけを表す図に限定する。
2. 追加した分岐を正式仕様にする。
   その場合は `screens/auth-login.md` の API 表、`openapi.yaml`、必要なら `architecture.md` まで同じ失敗契約を追記し、`TenantMembership` 不整合時に `401` を返す妥当性も明文化する。

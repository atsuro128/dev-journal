# auth-login フローチャートが API 契約にない失敗分岐を定義している

## 発見日
2026-03-22

## カテゴリ
detail-design

## 影響度
中

## 発見経緯
review

## 関連ステップ
Step 5（詳細設計）

## 問題

`screens/auth-login.md` の認証フローチャート（§11）が、同ファイルの API 仕様・`openapi.yaml`・シーケンス図に存在しない失敗分岐を定義している。

- フローチャート: `422 バリデーションエラー`（メール形式不正時）と `401 認証失敗`（TenantMembership 取得失敗時）を定義
- API 契約（openapi.yaml）: `POST /api/auth/login` のレスポンスは `200 / 401 / 429` のみ
- シーケンス図（§10）: 入力バリデーションの 422 分岐なし、TenantMembership 取得失敗の分岐なし

## 影響

実装者がフローチャートを正として実装すると、API 契約（openapi.yaml）と食い違う。テスト設計時にもエラーケースの網羅範囲が不明確になる。

## 提案

以下のどちらかに統一する:

1. **フローチャートを API 契約に合わせる**（推奨）: 422 分岐を削除し、TenantMembership 失敗も SEC-011 の同一 401 エラーに合流させる
2. **API 契約をフローチャートに合わせる**: openapi.yaml に 422 レスポンスを追加し、TenantMembership 不整合時の振る舞いを明文化する

---

## 解決内容
フローチャートを API 契約（200/400/401/429）に合わせる方針で修正。

1. **auth-login.md §11**: E1(422 バリデーションエラー) と E3(独立 401) を削除し、全認証関連エラーを E2(401 INVALID_CREDENTIALS) に合流。フローのスコープを「リクエストボディのパース・必須項目チェック成功後」に限定し、注記を更新。
2. **openapi.yaml**: login request schema から `format: email` / `minLength: 8` を削除（SEC-011 によりサーバー側で形式別エラーを返さないため）。`400` レスポンスを追加。description に SEC-011 によるスキーマ制約省略の意図を明記。
3. **security.md §8.4**: login エンドポイントが認証関連失敗を `401 INVALID_CREDENTIALS` に統一し、`400 VALIDATION_ERROR` を返さない旨の注記を追加。

関連: review-finding 046（format: email / minLength: 8 削除の妥当性を記録）

## 解決日
2026-03-23

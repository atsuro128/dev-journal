# 認証トークン保管方式を sessionStorage → httpOnly Cookie へ移行（post-MVP）

## 発見日
2026-04-13

## カテゴリ
security / post-mvp

## 影響度
中（post-MVP）

## 発見経緯
issue 083（F5 リロードでログアウトする問題）の対応方針検討中に、本来の最終形（httpOnly Cookie + CSRF token）を MVP スコープ外として切り出した

## 関連ステップ
post-MVP（運用フェーズ）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外のセキュリティ強化項目を記録するものであり、MVP リリース前には対応しない。管理方式は **`ops-080`** で検討中。決定後、本 issue は適切な管理先へ移動または分割される。

## 概要

issue 083 で、認証トークンの永続化方式を **sessionStorage** に決定した。これは「F5 でログアウトしない」UX 要件と「MVP の実装コスト最小化」を両立する妥協案であり、本来の最終形は **httpOnly Cookie + CSRF token + SameSite=Lax/Strict** 構成である。

sessionStorage を採用したことで以下のリスクが残存する:

1. **XSS 経由のトークン窃取** — XSS が成立した場合、攻撃者の JavaScript が `sessionStorage.getItem('auth.access_token')` で直接トークンを読み取り、外部サーバに送信できる。React の自動エスケープと CSP で XSS 自体を起こりにくくしているが、**多層防御の観点では「読み取れない場所に保管する」が望ましい**
2. **CSP 設定漏れの単一障害点化** — XSS が成立した瞬間に認証情報全部が露出する。httpOnly Cookie であれば JS から触れないため、XSS が起きてもトークン窃取は別経路（例: API 呼び出しによるなりすまし送信）に限定される

本番環境（実運用）で実ユーザーのデータを扱う場合は、httpOnly Cookie 方式に移行することを推奨する。

## 移行内容（提案）

### バックエンド（Go）
1. ログイン成功時に `Set-Cookie: access_token=...; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=900` を返す
2. リフレッシュトークン用の別 Cookie を `/api/auth/refresh` のみ Path で発行（`Path=/api/auth/refresh`）
3. 認証ミドルウェアを `Authorization: Bearer ...` ヘッダ取得から Cookie 取得に切り替え（または併用）
4. CSRF token 発行用エンドポイント新設、または double-submit cookie パターン採用
5. CORS 設定で `Access-Control-Allow-Credentials: true` と明示オリジン指定

### フロントエンド（React）
1. `stores/auth.ts` を削除（または「ログイン状態フラグだけ保持」に縮退）
2. `apiClient` の `Authorization` ヘッダ付与ロジックを削除
3. `fetch` の `credentials: 'include'` を全リクエストに付与
4. CSRF token をリクエストヘッダ（`X-CSRF-Token`）に付与する処理を追加
5. `useAuth()` の判定を「`/api/auth/me` を呼んで成功するか」に変更（クライアントから直接トークン有無を見れないため）
6. `PrivateRoute` の判定方式を再設計

### セキュリティ設計書
- `dev-journal/deliverables/docs/50_detail_design/security.md` のトークン管理セクションに httpOnly Cookie 方式を明記
- CSRF 保護方式の追記
- 既存の sessionStorage 方式を「MVP 版・暫定」と注記

### テスト
- BE: 認証ミドルウェアの Cookie 取得テスト、CSRF token 検証テスト
- FE: `apiClient` の credentials include 動作テスト、CSRF token 付与テスト
- E2E: ログイン → 認証必須 API → ログアウトの一連フロー

## 影響範囲

- 認証関連の全エンドポイント（`/api/auth/*` + 全認証必須エンドポイント）
- フロントエンドの API クライアント全体
- ミドルウェア構成（CSRF 検証ミドルウェア追加）
- インフラ設定（Cookie の Secure 属性のため HTTPS 必須）

## 対応タイミング

- MVP リリース後の運用フェーズ
- 実運用前のセキュリティ強化フェーズ（特にユーザーが実データを扱い始める前）
- 関連 ADR（または既存 ADR-0006 の更新）を起票してから着手

## 関連

- 起票契機: issue 083（F5 リロードでログアウトする問題、sessionStorage 方式採用）
- 関連 ADR: ADR-0006（認証方式 — 既存）
- 関連設計: `dev-journal/deliverables/docs/50_detail_design/security.md` SEC-003, SEC-004
- 管理方式: ops-080 で post-MVP issue の管理方法を決定後、本 issue を適切な管理先へ移動

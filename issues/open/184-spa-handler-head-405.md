# SPA fallback ハンドラが HEAD リクエストに 405 を返す（GET のみ登録）

## 発見日
2026-05-20

## カテゴリ
implementation

## 影響度
低

## 発見経緯
proactive

## 関連ステップ
Step 11-E Phase 6（ヘルスチェック）/ issue #183（SPA embed 実装）

## ブロッカー
なし

## 問題

PR #148（issue #183 対応）で実装した SPA fallback ハンドラが、chi router に `r.Get("/*", ...)` / `r.NotFound(...)` で登録されているため、**HEAD リクエストに対して `405 Method Not Allowed`（`Allow: GET`）を返す**。

Step 11-E Phase 6 の検証中、`curl -I`（HEAD メソッド）で静的アセットと SPA ルートを叩いた際に検出した:

```
$ curl -I http://localhost:8080/assets/index-B1Q7IuJ_.js
HTTP/1.1 405 Method Not Allowed
Allow: GET

$ curl -I http://<alb-dns>/
HTTP/1.1 405 Method Not Allowed
Allow: GET
```

`/` および `/assets/*`（SPA fallback 経路）が HEAD を受け付けない。`/health` 等の個別 GET エンドポイントも同様だが、本 issue の主対象は SPA fallback 経路。

## 影響

- **SPA 表示への実害はない**: ブラウザは HTML・JS・CSS を GET で取得するため、ログイン画面表示・SPA 動作は正常（Phase 6 でブラウザ表示確認済み）。
- HTTP 標準（RFC 9110 §9.3.2）では「GET をサポートするリソースは HEAD もサポートすべき」。HEAD を使うツール（外形監視・ヘルスチェッカー、CDN プリフェッチ、リンクチェッカー、一部のロードバランサ）が 405 を受ける。
- 現状 ALB のヘルスチェックは `/health` を GET で叩く設定のため Phase 6 では問題化していないが、HEAD ベースの監視を将来導入すると影響する。

## 提案

SPA fallback ハンドラを GET だけでなく HEAD でも登録する。`http.FileServer` 自体は HEAD を正しく処理できる（ボディなしでヘッダのみ返す）ため、chi のルーティング登録を GET + HEAD に広げれば足りる。

- `r.Get("/*", h)` に加えて `r.Head("/*", h)` を登録、または `r.Method` / `r.Handle` で GET+HEAD をまとめて扱う
- `r.NotFound` 経路も同様に HEAD を考慮

post-MVP 対応で可（影響度低、SPA 表示に実害なし）。対応時は `internal/spa/handler_test.go` に HEAD リクエストのテストケースを追加すること。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

# cloudfront.tf: default_root_object="index.html" が SPA fallback と組み合わさり / で無限リダイレクトループを引き起こす

## 発見日
2026-05-26

## カテゴリ
infrastructure

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
Step 11-E（11-F 着手直前、UAT 開始前の動作確認時）

## ブロッカー
**11-F UAT 着手ブロッカー**（ルート URL でブラウザがアクセス不能のため UAT 実施不可能）

## 問題

`https://djhmwtrr79jdq.cloudfront.net/` にブラウザでアクセスすると「**このページは現在機能していません**」（Chrome `ERR_TOO_MANY_REDIRECTS`）が表示される。

### 再現

```bash
$ curl -sI https://djhmwtrr79jdq.cloudfront.net/
HTTP/1.1 301 Moved Permanently
Location: ./
...
```

`/` リクエストに対して **CloudFront が 301 + `Location: ./` を返している**。ブラウザは `./` を `/` に解釈し、同じリクエストを繰り返して無限ループに陥る。

### 真因

`expense-saas/infra/terraform/cloudfront.tf` L30 で `default_root_object = "index.html"` を設定している。これは S3 オリジン向けの設定で、CloudFront が `/` リクエストを `/index.html` に内部書き換えしてオリジン取得する機能。

カスタムオリジン（ALB）＋ Go 製 SPA fallback ハンドラの組み合わせでは以下のループが発生する:

```
ブラウザ: GET /
  ↓
CloudFront: default_root_object="index.html" のため /index.html をオリジンに転送
  ↓
Go SPA handler (internal/spa/handler.go):
  - リクエストパス /index.html は拡張子あり (.html) と判定
  - http.FileServer.ServeHTTP に委譲
  ↓
http.FileServer (Go std lib):
  - /index.html リクエストには Location: ./ で 301 (canonical 化、
    index.html を直接参照しないようリダイレクト)
  ↓
CloudFront: 301 を素通しでブラウザに返す
  ↓
ブラウザ: ./ → / に解釈 → GET / → 以下無限ループ
```

### 動く経路（参考）

- `/login` `/dashboard` 等 SPA ルート → 200 OK（拡張子なし → serveIndexHTML 直接呼び出し）
- `/health` → 200 OK（chi router で個別登録）
- `/api/*` → 正常動作
- EC2 localhost で `curl http://localhost:8080/` → 200 OK（Go アプリ自体は `/` を正しく処理）

つまり **CloudFront default_root_object 経由でのみ regression**。

### 影響範囲

- ブラウザでのルート URL アクセス = 100% 失敗
- ポートフォリオデモ / UAT で致命的
- SPA リンク経由（`/login` 等を直接踏む）でログイン後は SPA 内ルーティングで動作

### regression のタイミング

- PR #148（issue #183 SPA embed 実装）の handler.go 実装時から `http.FileServer` の canonical 化挙動を内包していた
- PR #151（issue #185 T3 CloudFront 化）で `default_root_object = "index.html"` を追加した時点で顕在化
- 過去 5/20 セッションは ALB 直接 HTTP の状態で「SPA 配信実地確認済み」とした → CloudFront 経由 `/` での動作は実機確認していなかった見落とし

## 影響

- **11-F UAT 着手ブロッカー**
- ポートフォリオ URL として共有不可
- 副次的に「SPA fallback handler のテストが /index.html 経由を検証していない」品質課題も露呈

## 提案

### 修正案 A（採用、最小修正）

`cloudfront.tf` L30 `default_root_object = "index.html"` を削除する。

理由:
- SPA handler はすでに `/` リクエストを `serveIndexHTML` 経由で正しく処理する設計（`/login` 等が動いている証拠）
- `default_root_object` は S3 オリジン向け機能で、SPA fallback ハンドラがあるカスタムオリジン構成では不要
- 削除後の挙動: CloudFront は `/` をそのまま ALB に転送 → Go SPA handler の `/` パターン（拡張子なし）→ serveIndexHTML → 200 OK + index.html

修正規模: 1 行削除 + terraform apply（CloudFront in-place 変更、10-15 分伝播）

### 修正案 B（不採用）

`internal/spa/handler.go` で `/index.html` リクエストも `serveIndexHTML` に分岐させる。

不採用理由:
- アプリ側修正＋テスト追加＋docker build＋S3 upload＋EC2 redeploy が必要で工数大
- CloudFront 側の不要設定が残る

### 再発防止策

- **CloudFront cache behavior の実機ブラウザ確認をリリース前に必須化**: curl だけでは無限リダイレクトは検出しづらい（`curl -L` で 50 リダイレクト後に検出）。ブラウザでの実機確認が必須
- **SPA handler の handler_test.go に `/index.html` ケース追加**: 現状 `/` と `/login` のみ。`/index.html` を直接叩いた時の挙動を unit test で検証
- 関連 issue: #189（SG description の見落としと同根の「設定セット間の整合性チェック不足」、品質追跡）

### 着手タイミング

**即時**（11-F UAT 着手ブロッカー）。本 issue を解決するまで UAT 開始不可。

---

## 解決内容

PR #155（expense-saas、squash commit `51f93fb`）でマージ完了。

### 実装変更

`infra/terraform/cloudfront.tf` L30 `default_root_object = "index.html"` を 1 行削除（コメントで理由・経緯を明記）。

修正後の挙動:
- CloudFront は `/` をリライトせずそのまま ALB に転送
- ALB → EC2 → Go chi router `r.NotFound(spaHandler)` → SPA handler の serveIndexHTML 経由で index.html 配信
- `/index.html` への canonical 化 301 ループが発生しない

### terraform apply

- 2026-05-26: in-place modify、Modifications complete 33s
- CloudFront Status: `Deployed`（伝播完了確認）
- 検証:
  ```
  $ curl -sI https://djhmwtrr79jdq.cloudfront.net/
  HTTP/1.1 200 OK
  Content-Type: text/html; charset=utf-8
  Content-Length: 327
  ```
  ```
  $ curl -s https://djhmwtrr79jdq.cloudfront.net/ | head -c 50
  <!doctype html><html lang="ja"><head>...<title>経費精算SaaS</title>
  ```

### レビューサイクル

- reviewer エージェント: PASS（blocker 0、副作用なし確認）
- codex レビュー: PASS（blocker 0、`default_root_object` は S3 オリジン向け機能でカスタムオリジン + SPA 構成では不要・有害の判断を支持）

### 派生タスク（post-MVP）

issue #190 §再発防止策 で提案した以下は post-MVP 検討:
- ブラウザ実機確認のリリース前必須化（チェックリスト化）
- `internal/spa/handler_test.go` に `/index.html` 直接アクセスケース追加

これらは別 issue として今回は切り出さない（issue #189 の post-MVP lint 検討と一括で品質追跡 issue として整理する想定）。

## 解決日
2026-05-26

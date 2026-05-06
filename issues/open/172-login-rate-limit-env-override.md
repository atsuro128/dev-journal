# ログインエンドポイントのレート制限値を環境変数で上書き可能にする

## 発見日
2026-05-06

## カテゴリ
backend / configuration / e2e-test-blocker

## 影響度
中（11-C E2E テストのブロッカー。本番動作には影響なし、開発・テスト環境の運用改善）

## 発見経緯

11-C E2E テスト（PR #138）の Windows 側実行で、複数ロールを連続でログインするテスト群が `loginLimit`（5 req/min/IP）に到達し、429 で 4 件 FAIL。

具体的な失敗パターン（`test-results/results.json` より）:

| テスト | 結果 | 累積ログイン回数 |
|---|---|---|
| CRS-055（フロー1: Member→Approver→Accounting）| FAIL | 3 |
| CRS-056（Member 単独）| PASS | 4 |
| CRS-060（Approver 単独）| PASS | 5 |
| CRS-063（Accounting 単独）| **FAIL（429）** | 6 |
| CRS-066（フロー2）| FAIL | 7 |
| CRS-073/074（Admin 単独）| PASS（1 分経過後）| - |
| CRS-075（Admin テナント情報）| **FAIL（429）** | - |
| CRS-075-補足（Member）| PASS | - |

エラーメッセージは「しばらく待ってから再試行してください」で、ログイン画面にレート制限アラートが表示。

## 関連ステップ

Step 11-C（E2E テスト実装）の動作確認ブロッカー。本 issue 解消後に PR #138 のテスト再実行が可能になる。

## ブロッカー

- PR #138（11-C E2E テスト）のローカル実行
- 将来の手動 E2E 検証（複数ロール連続操作）

## 問題

### 現状の実装

`expense-saas/cmd/server/main.go:174` でログインのレート制限値がハードコード:

```go
pub.With(middleware.RateLimitByIP(bgCtx, 5, time.Minute)).Post("/api/auth/login", authHandler.Login)
```

- 値: 5 req/min/IP
- 環境変数による上書き不可
- `internal/config/` も該当 env を読み取る実装なし

### なぜ問題か

- **本番**: 5 req/min/IP は妥当（ブルートフォース対策、`security.md` §4.4 準拠）
- **開発・テスト**: E2E テストは複数ロールを短時間に連続でログインする想定があり、5/min では実用に耐えない
- E2E テストのために本番値を緩めるのは安全性を損なう

## 提案

### 環境変数による上書き機構を追加

`internal/config/config.go` で env 読み取り、`main.go` で適用:

```go
// internal/config/config.go
LoginRateLimitPerMinute  int  // env: LOGIN_RATE_LIMIT_PER_MINUTE（未設定時は 5）
UnauthRateLimitPerMinute int  // env: UNAUTH_RATE_LIMIT_PER_MINUTE（未設定時は 20）

// main.go (該当箇所)
r.Use(middleware.RateLimitByIP(bgCtx, cfg.UnauthRateLimitPerMinute, time.Minute))
pub.With(middleware.RateLimitByIP(bgCtx, cfg.LoginRateLimitPerMinute, time.Minute))
```

### 開発・テスト環境での適用

`docker-compose.yml` の api サービスに env を追加:

```yaml
environment:
  LOGIN_RATE_LIMIT_PER_MINUTE: "100"     # 開発・E2E 用に緩和
  UNAUTH_RATE_LIMIT_PER_MINUTE: "200"    # 同上
```

本番（AWS デプロイ環境）では env 未設定 → デフォルト値（5 / 20）が使われる。

### 設計書反映

- `security.md` §4.4 に「本番値は 5/20、開発・E2E 環境は env で上書き可能」を追記
- `env_config.md`（運用設計）に新規 env 変数を追加

## 影響範囲（事前評価）

- 既存テスト（11-B `cross_cutting_test.go` / `rate_limit_test.go`）は **テスト内で限界値を直接渡している**ため影響なし
- 既存の手動スモークテスト（11-A）は影響なし
- 本番デプロイは env 未設定でデフォルト値が使われるため挙動不変

## 対応優先度

11-C のブロッカーなので **MVP 内で対応必須**。PR #138 マージ前に解消する。

## 対応方針

別 PR（小規模）で先に対応し、master にマージ後、11-C ブランチで master を取り込んで再テスト実行。

## 関連 issue / PR

- 関連 PR: #138（11-C E2E テスト、本 issue のブロック対象）
- 設計参照: `security.md` §4.4（レート制限）、`env_config.md`（環境変数一覧）

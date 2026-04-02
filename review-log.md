# レビューログ

## レビュー: 2026-04-02 15:18

### 対象
- `expense-saas/cmd/server/main.go`（Step 8 成果物）

### 確認済み
- ステップ 1: 設定読み込み（config.LoadConfig、環境変数）
- ステップ 2: ログ設定（slog、JSON 形式、LOG_LEVEL 切替）
- ステップ 3: DB コネクションプール作成（pgxpool.New、プールの仕組み、上限4本）
- ステップ 4: DB 疎通確認（Ping、失敗時即終了）
- ステップ 5: JWT Verifier 初期化（RS256、非 fatal 設計 → issue 052 で必須化予定）
- ステップ 6: バックグラウンド Context（レートリミッター掃除用、トークンバケット方式の詳細）
- ステップ 7: DI（Repository → Authorizer → Service → Handler の手動組み立て）— 途中まで
  - Repository: テーブル単位で9個、全て同じ pool を共有
  - Service: Repository を受け取って業務ロジック、全7個
  - Handler: Service を受け取って HTTP 処理、全8個
  - Authorizer: ロール + 所有権の認可判定
- ステップ 8〜12: 説明は受けたが詳細レビューは未実施

### 未確認
- main.go ステップ 8〜12 の詳細レビュー（ルーター、ルート定義、サーバー起動、グレースフルシャットダウン）
- `internal/` の個別ファイル（handler, service, domain, repository, middleware, config, pkg）
- `frontend/`
- `Makefile`
- `db/`

### 発見した issue
- 050: JWT 署名アルゴリズムの耐量子計算機方式への移行（RS256 → ML-DSA）
- 051: アーキテクチャ設計書のスケーラビリティトレードオフ未記載
- 052: JWT 鍵ファイル未設定でもサーバーが起動する問題
- 053: 非機能要件のテストカバレッジ不足

### 疑問・メモ
- ステップ 7 の DI は確認途中。次回はステップ 8（ルーター）の詳細から再開
- middleware は全ファイル読み込み済みだが、ユーザーへの説明は一部のみ（コンテキスト節約のため中断）
- 非機能要件の同時接続100ユーザーテストが MVP スコープ外 — Step 11 で対応予定

### 学んだこと
- **Go 構文**: ポインタ（`&`, `*`）、struct、レシーバ（`(r *userRepo)`）、暗黙的インターフェース、defer、context、public/private（大文字/小文字）
- **JWT の仕組み**: アクセストークン 15分 / リフレッシュトークン 7日、RS256 署名、トークン更新フロー
- **セキュリティ**: RS256 は 2030 年以降 NIST 非推奨、EdDSA も量子耐性なし、ML-DSA（FIPS 204）が候補
- **レートリミッター**: トークンバケット方式、IP 別（20/min）+ ユーザー別（100/min）の2層、掃除は5分ごと
- **スケーラビリティ**: RLS 方式の限界（コネクションプール上限、RDS Proxy 相性、リードレプリカ非対応）
- **localStorage vs Cookie**: JWT は localStorage に保存、自動送信されないので CSRF に強い
- **Java との違い**: `new` → `&struct{}`、`class` → `struct`、`try/catch` → `if err != nil`、`implements` → 暗黙的

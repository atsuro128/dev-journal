# レビューログ

## レビュー: 2026-04-07 21:01

### 対象
- `expense-saas/cmd/server/main.go`（Step 8 成果物）— 前回の続き

### 確認済み
- ステップ 7: DI — Service 数の修正（7個→8個）、全体の確認完了
- ステップ 8: ルーター生成＋共通ミドルウェアチェーン（CORS → SecurityHeaders → RequestID → Logger → RateLimit(IP)）、順序の設計意図
- ステップ 9: 認証不要ルート（auth 系 7 エンドポイント）、logout/refresh が認証不要側にある理由
- ステップ 10: 認証必須グループ（Auth → TenantContext → RateLimitByUser 3層）、ロール別 4 サブグループの RBAC 設計
- ステップ 11: HTTP サーバ起動（0.0.0.0 バインド、http.Server を変数に持つ理由）
- ステップ 12: グレースフルシャットダウン（SIGINT/SIGTERM 待ち、bgCancel、srv.Shutdown 10秒タイムアウト）

### 未確認
- `internal/` の個別ファイル（handler, service, domain, repository, middleware, config, pkg）
- `frontend/`
- `Makefile`
- `db/`

### 発見した issue
- なし

### 疑問・メモ
- API パスのハードコーディングについて質問あり → 現状は main.go のみで使用するため問題なし。肥大化時は routes_*.go へのファイル分割で対応する方針
- ReadTimeout / WriteTimeout 未設定 — MVP では ALB 側タイムアウトに委ねる設計

### 学んだこと
- **ポートと 0.0.0.0**: ポートはサービスの窓口番号、0.0.0.0 は全インターフェース待ち受け。Fargate（awsvpc モード）では ENI 経由の通信を受けるため 0.0.0.0 が実質必須
- **chi の Group と With**: Group はミドルウェアスコープを作る（URL プレフィックスは付かない）、With はサブグループに追加ミドルウェアを適用
- **グレースフルシャットダウン**: <-quit でシグナル待ち → bgCancel で BG 処理停止 → srv.Shutdown で処理中リクエスト完了を待って終了。ECS の SIGTERM 猶予 30 秒に対し 10 秒タイムアウト
- **Go のチャネル**: `<-quit` はチャネルからデータが届くまでブロックする構文。シグナル受信まで main() はここで待機し続ける

---

## レビュー: 2026-04-02 15:18（前回）

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
- ステップ 7 の DI は確認途中。次回はステップ 7 の続きから再開
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

# 監視・ログ設計

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 何を計測し、どう検知し、どんなログを残すかを定義する |
| 正本情報 | 監視対象、閾値、ログフィールド、通知フロー |
| 扱わない内容 | 障害対応手順（runbook.md）、リリース手順（release.md）、復旧 Runbook |
| 主な参照元 | `30_arch/adr/0005-monitoring-logging.md`, `30_arch/architecture.md`, `10_requirements/requirements.md` |
| 主な参照先 | `70_operations/runbook.md`, `70_operations/release.md` |

## 1. 概要

本書は経費精算SaaS の監視・ログ設計を詳細に定義する。ADR-0005（監視・ログ戦略）で決定した方針を具体的な実装仕様に落とし込む。

### 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `30_arch/adr/0005-monitoring-logging.md` | 監視・ログ戦略の決定（主要参照） |
| `30_arch/architecture.md` §7 | 監視・ログ方針のサマリー |
| `30_arch/adr/0004-infra.md` | インフラ構成（ECS Fargate / RDS / CloudWatch） |
| `10_requirements/requirements.md` §4.1, §4.3 | 非機能要件（p95 500ms、稼働率 99.5%） |
| `.claude/rules/security-policy.md` §9 | セキュリティログ要件 |

### 設計方針サマリー

| 項目 | 選定 |
|------|------|
| ログフレームワーク | log/slog（Go 標準ライブラリ） |
| ログ出力形式 | JSON（構造化ログ） |
| ログ出力先 | stdout → CloudWatch Logs（ECS 自動転送） |
| メトリクス基盤 | Amazon CloudWatch |
| ダッシュボード | CloudWatch Logs Insights（アドホッククエリ） |
| アラーム | CloudWatch メトリクスフィルタ → CloudWatch Alarms |
| 通知 | SNS → メール（MVP） |

---

## 2. 構造化ログ設計

### 2.1 ログフォーマット

slog の `JSONHandler` を使用し、全ログを JSON 形式で stdout に出力する。ECS Fargate の awslogs ログドライバーにより CloudWatch Logs に自動転送される。

```json
{
  "time": "2026-03-22T10:30:00.123Z",
  "level": "INFO",
  "msg": "request completed",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "tenant_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "user_id": "f0e1d2c3-b4a5-6789-0abc-def123456789",
  "method": "GET",
  "path": "/api/reports",
  "status": 200,
  "duration_ms": 45,
  "user_agent": "Mozilla/5.0 ..."
}
```

### 2.2 標準フィールド定義

全ログに共通して含めるフィールドを定義する。

#### アクセスログフィールド（HTTP リクエスト/レスポンス）

Logger ミドルウェアが自動的に付与する。

| フィールド | 型 | 説明 | 例 | 必須 |
|-----------|-----|------|-----|------|
| `time` | string | ISO 8601 (UTC) タイムスタンプ | `"2026-03-22T10:30:00.123Z"` | 必須 |
| `level` | string | ログレベル | `"INFO"` | 必須 |
| `msg` | string | ログメッセージ | `"request completed"` | 必須 |
| `request_id` | string | リクエスト固有 ID（UUID v4） | `"550e8400-..."` | 必須 |
| `tenant_id` | string | テナント ID（認証済みリクエストのみ） | `"a1b2c3d4-..."` | 条件付き |
| `user_id` | string | ユーザー ID（認証済みリクエストのみ） | `"f0e1d2c3-..."` | 条件付き |
| `method` | string | HTTP メソッド | `"GET"` | 必須 |
| `path` | string | リクエストパス（クエリパラメータ除外） | `"/api/reports"` | 必須 |
| `status` | int | HTTP ステータスコード | `200` | 必須 |
| `duration_ms` | int | レスポンスタイム（ミリ秒） | `45` | 必須 |
| `user_agent` | string | クライアントの User-Agent | `"Mozilla/5.0 ..."` | 必須 |
| `remote_ip` | string | クライアント IP アドレス | `"203.0.113.1"` | 必須 |
| `error` | string | エラー詳細（エラー時のみ） | `"database connection timeout"` | 条件付き |

#### アプリケーションログフィールド（業務イベント）

ハンドラ層・サービス層で明示的に出力する。

| フィールド | 型 | 説明 | 例 | 必須 |
|-----------|-----|------|-----|------|
| `time` | string | ISO 8601 (UTC) タイムスタンプ | `"2026-03-22T10:30:00.123Z"` | 必須 |
| `level` | string | ログレベル | `"INFO"` | 必須 |
| `msg` | string | ログメッセージ | `"report submitted"` | 必須 |
| `request_id` | string | リクエスト固有 ID（context から取得） | `"550e8400-..."` | 必須 |
| `tenant_id` | string | テナント ID | `"a1b2c3d4-..."` | 条件付き |
| `user_id` | string | ユーザー ID | `"f0e1d2c3-..."` | 条件付き |
| `component` | string | 出力元コンポーネント | `"service.report"` | 推奨 |
| `action` | string | 実行されたアクション | `"submit"` | 推奨 |
| `resource_type` | string | 対象リソースの種類 | `"expense_report"` | 推奨 |
| `resource_id` | string | 対象リソースの ID | `"report-uuid"` | 推奨 |

### 2.3 slog 実装方針

#### ロガーの初期化

```go
// cmd/server/main.go での初期化イメージ
var level slog.Level
switch os.Getenv("LOG_LEVEL") {
case "debug":
    level = slog.LevelDebug
case "warn":
    level = slog.LevelWarn
case "error":
    level = slog.LevelError
default:
    level = slog.LevelInfo
}

handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: level,
})
logger := slog.New(handler)
slog.SetDefault(logger)
```

#### ミドルウェアでの context 伝播

RequestID ミドルウェアで生成した `request_id` を context に格納し、Logger ミドルウェアおよび後続の全層で利用する。

```go
// Logger ミドルウェアの出力イメージ
slog.InfoContext(ctx, "request completed",
    "request_id", middleware.GetRequestID(ctx),
    "tenant_id", middleware.GetTenantID(ctx),
    "user_id", middleware.GetUserID(ctx),
    "method", r.Method,
    "path", r.URL.Path,
    "status", ww.Status(),
    "duration_ms", elapsed.Milliseconds(),
    "user_agent", r.UserAgent(),
    "remote_ip", r.RemoteAddr,
)
```

#### サービス層でのログ出力

```go
// service/report.go での出力イメージ
slog.InfoContext(ctx, "report submitted",
    "component", "service.report",
    "action", "submit",
    "resource_type", "expense_report",
    "resource_id", report.ID.String(),
)
```

---

## 3. ログレベルポリシー

### 3.1 ログレベル定義

環境変数 `LOG_LEVEL` で制御する。本番環境は `INFO`、開発環境は `DEBUG` を設定する。

| レベル | 用途 | 本番出力 | 開発出力 |
|--------|------|----------|----------|
| DEBUG | 開発時の詳細情報 | 出力しない | 出力する |
| INFO | 通常の業務イベント | 出力する | 出力する |
| WARN | 異常だが処理続行可能 | 出力する | 出力する |
| ERROR | 処理失敗 | 出力する | 出力する |

### 3.2 ログレベル運用基準（出力例付き）

#### ERROR: 処理失敗（即時対応が必要な可能性あり）

オペレータが確認すべきイベント。アラート対象になりうる。

| イベント | メッセージ例 |
|---------|------------|
| DB 接続エラー | `"database connection failed"` |
| DB クエリ実行エラー | `"query execution failed"` |
| S3 操作エラー | `"s3 upload failed"` |
| JWT 署名鍵の読み込み失敗 | `"failed to load signing key"` |
| パニックリカバリ | `"panic recovered"` |

```json
{
  "time": "2026-03-22T10:30:00.123Z",
  "level": "ERROR",
  "msg": "database connection failed",
  "request_id": "550e8400-...",
  "component": "repository.postgres",
  "error": "dial tcp 10.0.1.100:5432: connect: connection refused"
}
```

#### WARN: 異常だが処理続行可能（傾向監視が必要）

単発では問題ないが、頻発する場合は調査が必要なイベント。

| イベント | メッセージ例 |
|---------|------------|
| レート制限接近（閾値の 80% 到達） | `"rate limit threshold approaching"` |
| DB コネクションプールの待ち発生 | `"connection pool wait occurred"` |
| レスポンスタイム閾値超過（p95 基準） | `"slow request detected"` |
| 無効な JWT トークン（改ざん等） | `"invalid token rejected"` |
| 楽観的ロックの競合（409 Conflict） | `"optimistic lock conflict"` |

```json
{
  "time": "2026-03-22T10:30:00.123Z",
  "level": "WARN",
  "msg": "slow request detected",
  "request_id": "550e8400-...",
  "method": "GET",
  "path": "/api/reports",
  "duration_ms": 980
}
```

#### INFO: 通常の業務イベント（運用状況の把握）

正常な業務フローのマイルストーン。

| イベント | メッセージ例 |
|---------|------------|
| HTTP リクエスト完了 | `"request completed"` |
| 状態遷移の実行 | `"report submitted"`, `"report approved"`, `"report rejected"`, `"report marked as paid"` |
| ログイン成功 | `"login succeeded"` |
| ログイン失敗 | `"login failed"` |
| ログアウト | `"logout completed"` |
| ファイルアップロード完了 | `"file uploaded"` |
| ファイル削除完了 | `"file deleted"` |
| レート制限による拒否（429） | `"rate limit exceeded"` |
| 認可拒否（403） | `"authorization denied"` |
| テナント境界越えアクセス（404） | `"cross-tenant access blocked"` |
| サーバー起動 | `"server started"` |
| サーバーシャットダウン | `"server shutting down"` |

```json
{
  "time": "2026-03-22T10:30:00.123Z",
  "level": "INFO",
  "msg": "report approved",
  "request_id": "550e8400-...",
  "tenant_id": "a1b2c3d4-...",
  "user_id": "f0e1d2c3-...",
  "component": "service.workflow",
  "action": "approve",
  "resource_type": "expense_report",
  "resource_id": "report-uuid"
}
```

#### DEBUG: 開発時の詳細情報（本番では出力しない）

開発・デバッグ時にのみ必要な情報。

| イベント | メッセージ例 |
|---------|------------|
| SQL クエリの実行内容 | `"query executed"` |
| リクエストボディ（マスク済み） | `"request body received"` |
| JWT クレームの検証詳細 | `"token claims validated"` |
| RLS 設定の適用 | `"rls context set"` |
| ミドルウェアチェーンの通過 | `"middleware passed"` |

```json
{
  "time": "2026-03-22T10:30:00.123Z",
  "level": "DEBUG",
  "msg": "query executed",
  "component": "repository.postgres",
  "query_name": "ListReportsByTenant",
  "duration_ms": 12
}
```

---

## 4. セキュリティログ要件

### 4.1 ログに出力してはならない情報

security-policy.md §9 および ADR-0005 に基づく。

| 禁止対象 | 理由 |
|---------|------|
| パスワード（平文・ハッシュ含む） | 認証情報の漏洩防止 |
| JWT トークン（アクセス・リフレッシュ） | セッションハイジャック防止 |
| パスワードリセットトークン | 認証情報の漏洩防止 |
| リクエストボディの個人情報 | DEBUG レベルでもマスク処理を行う |

### 4.2 マスク処理ルール

DEBUG レベルでリクエストボディをログに含める場合、以下のフィールドをマスクする。

| フィールド | マスク方法 | 例 |
|-----------|----------|-----|
| `password` | 完全マスク | `"***"` |
| `email` | ローカル部分を部分マスク | `"t***@example.com"` |
| `token` (全種別) | 完全マスク | `"***"` |

### 4.3 セキュリティイベントのログ出力

security-policy.md §9 で定められたセキュリティイベントは INFO レベルで必ずログに記録する。

| イベント | ログメッセージ | 追加フィールド |
|---------|-------------|--------------|
| ログイン成功 | `"login succeeded"` | `remote_ip`, `user_id`, `tenant_id` |
| ログイン失敗 | `"login failed"` | `remote_ip`, `email_hash`（メールアドレスの SHA-256 先頭 8 文字） |
| ログアウト | `"logout completed"` | `user_id`, `tenant_id` |
| 認可拒否（403） | `"authorization denied"` | `user_id`, `tenant_id`, `required_role`, `path` |
| テナント境界越え（404） | `"cross-tenant access blocked"` | `user_id`, `tenant_id`, `path` |
| レート制限超過（429） | `"rate limit exceeded"` | `remote_ip`, `user_id`（認証済みの場合）, `limit_type` |
| 無効な JWT | `"invalid token rejected"` | `remote_ip`, `reason`（expired / invalid_signature / malformed） |

ログイン失敗時の `email_hash` は、ブルートフォース攻撃のパターン分析に使用する。メールアドレスそのものはログに出力しない。

---

## 5. メトリクス設計

### 5.1 メトリクス収集方式

| 用途 | 方式 | 説明 |
|------|------|------|
| インフラメトリクス | ECS Container Insights / RDS 標準メトリクス | 自動収集。追加設定不要 |
| アプリケーションメトリクス（ダッシュボード） | CloudWatch Logs Insights | アドホッククエリで柔軟に集計 |
| アプリケーションメトリクス（アラート） | CloudWatch メトリクスフィルタ | 構造化ログからパターンマッチで時系列メトリクスを生成。Alarm のソースとして使用 |

カスタムメトリクス SDK（PutMetricData API、EMF 等）は使用しない。構造化ログに含まれる `status`、`duration_ms` 等のフィールドから、メトリクスフィルタで時系列メトリクスを生成する。

### 5.2 CloudWatch メトリクスフィルタ定義

構造化ログのフィールドをパターンマッチし、CloudWatch メトリクスを自動生成する。生成されたメトリクスは CloudWatch Alarms のソースとして使用する。

| メトリクス名 | 名前空間 | フィルタパターン | メトリクス値 | 統計 |
|------------|----------|----------------|------------|------|
| `RequestCount` | `ExpenseSaaS/API` | `{ $.status >= 200 }` | 1（カウント） | Sum |
| `5xxCount` | `ExpenseSaaS/API` | `{ $.status >= 500 }` | 1（カウント） | Sum |
| `4xxCount` | `ExpenseSaaS/API` | `{ $.status >= 400 && $.status < 500 }` | 1（カウント） | Sum |
| `Duration` | `ExpenseSaaS/API` | `{ $.duration_ms >= 0 }` | `$.duration_ms` | p95, p99, Average |
| `LoginFailureCount` | `ExpenseSaaS/Security` | `{ $.msg = "login failed" }` | 1（カウント） | Sum |
| `AuthzDeniedCount` | `ExpenseSaaS/Security` | `{ $.msg = "authorization denied" }` | 1（カウント） | Sum |
| `RateLimitCount` | `ExpenseSaaS/Security` | `{ $.msg = "rate limit exceeded" }` | 1（カウント） | Sum |

### 5.3 自動収集メトリクス（インフラ）

ECS Container Insights および RDS 標準メトリクスにより自動収集されるメトリクス。

#### ECS Fargate（Container Insights）

| メトリクス | 説明 | 名前空間 |
|-----------|------|----------|
| `CpuUtilized` / `CpuReserved` | タスク CPU 使用率 | `ECS/ContainerInsights` |
| `MemoryUtilized` / `MemoryReserved` | タスク メモリ使用率 | `ECS/ContainerInsights` |
| `RunningTaskCount` | 実行中タスク数 | `ECS/ContainerInsights` |
| `DesiredTaskCount` | 希望タスク数 | `ECS/ContainerInsights` |

#### RDS for PostgreSQL

| メトリクス | 説明 | 名前空間 |
|-----------|------|----------|
| `CPUUtilization` | DB インスタンス CPU 使用率 | `AWS/RDS` |
| `FreeStorageSpace` | 空きストレージ容量 | `AWS/RDS` |
| `DatabaseConnections` | 現在のDB接続数 | `AWS/RDS` |
| `FreeableMemory` | 空きメモリ | `AWS/RDS` |
| `ReadLatency` / `WriteLatency` | ディスク I/O レイテンシ | `AWS/RDS` |

#### ALB

| メトリクス | 説明 | 名前空間 |
|-----------|------|----------|
| `HealthyHostCount` | 正常なターゲット数 | `AWS/ApplicationELB` |
| `UnHealthyHostCount` | 異常なターゲット数 | `AWS/ApplicationELB` |
| `TargetResponseTime` | ターゲットレスポンスタイム | `AWS/ApplicationELB` |
| `HTTPCode_Target_5XX_Count` | 5xx レスポンス数 | `AWS/ApplicationELB` |

### 5.4 MVP で計測すべき指標一覧

| カテゴリ | 指標 | 取得方法 | アラート閾値 | 重要度 |
|----------|------|----------|-------------|--------|
| パフォーマンス | API レスポンスタイム p95 | メトリクスフィルタ `Duration` の p95 統計 | > 500ms | Warning |
| パフォーマンス | API レスポンスタイム p99 | メトリクスフィルタ `Duration` の p99 統計 | > 1000ms | 参考値（アラート対象外） |
| エラー | 5xx エラーレート | Math: `5xxCount / RequestCount * 100` | > 1% | Warning |
| エラー | 5xx エラーレート（重大） | Math: `5xxCount / RequestCount * 100` | > 5% | Critical |
| 可用性 | ヘルスチェック失敗 | ALB ヘルスチェック `UnHealthyHostCount` | > 0（連続 2 回） | Critical |
| リソース | ECS タスク CPU 使用率 | Container Insights `CpuUtilized/CpuReserved` | > 80% | Warning |
| リソース | ECS タスク メモリ使用率 | Container Insights `MemoryUtilized/MemoryReserved` | > 80% | Warning |
| リソース | RDS CPU 使用率 | RDS `CPUUtilization` | > 80% | Warning |
| リソース | RDS 空きストレージ | RDS `FreeStorageSpace` | < 20% | Warning |
| リソース | RDS 接続数 | RDS `DatabaseConnections` | > 80% of max_connections | Warning |
| セキュリティ | ログイン失敗率 | メトリクスフィルタ `LoginFailureCount` | > 50 回/5 分 | Warning |

---

## 6. アラート設計

### 6.1 アラート重要度と通知先

| 重要度 | 通知先 | 対応目標時間 | 対象 |
|--------|--------|------------|------|
| Critical | SNS → メール | 即時確認 | ヘルスチェック連続失敗、5xx エラーレート > 5% |
| Warning | SNS → メール | 営業時間内に確認 | p95 > 500ms、CPU > 80%、ストレージ < 20%、5xx > 1% |

MVP ではメール通知のみ。将来的に Slack / PagerDuty 連携を検討する。

### 6.2 CloudWatch Alarms 定義

#### Critical アラーム

| アラーム名 | メトリクス | 条件 | 評価期間 | データポイント |
|-----------|-----------|------|---------|--------------|
| `HealthCheck-Failure` | ALB `UnHealthyHostCount` | > 0 | 1 分 | 2/2（連続 2 回） |
| `High-5xx-Rate-Critical` | Math: `5xxCount/RequestCount*100` | > 5% | 1 分 | 3/5（5 分中 3 回） |

#### Warning アラーム

| アラーム名 | メトリクス | 条件 | 評価期間 | データポイント |
|-----------|-----------|------|---------|--------------|
| `High-5xx-Rate-Warning` | Math: `5xxCount/RequestCount*100` | > 1% | 5 分 | 3/3（連続 3 回） |
| `High-p95-Latency` | `Duration` p95 | > 500ms | 5 分 | 3/3（連続 3 回） |
| `High-ECS-CPU` | ECS `CpuUtilized/CpuReserved*100` | > 80% | 5 分 | 3/3 |
| `High-ECS-Memory` | ECS `MemoryUtilized/MemoryReserved*100` | > 80% | 5 分 | 3/3 |
| `High-RDS-CPU` | RDS `CPUUtilization` | > 80% | 5 分 | 3/3 |
| `Low-RDS-Storage` | RDS `FreeStorageSpace` | < 20% | 5 分 | 3/3 |
| `High-RDS-Connections` | RDS `DatabaseConnections` | > 80% of max | 5 分 | 3/3 |
| `High-Login-Failures` | `LoginFailureCount` | > 50/5min | 5 分 | 1/1 |

### 6.3 SNS トピック構成

| トピック名 | 用途 | サブスクリプション |
|-----------|------|----------------|
| `expense-saas-critical` | Critical アラーム通知 | 運用担当者メールアドレス |
| `expense-saas-warning` | Warning アラーム通知 | 運用担当者メールアドレス |

---

## 7. ヘルスチェックエンドポイント

### 7.1 仕様

```
GET /health
```

認証不要。ミドルウェアチェーンの Auth / TenantContext / RBAC をスキップする。RateLimit ミドルウェアも適用しない（ALB からの定期チェックのため）。

### 7.2 レスポンス

#### 正常時（200 OK）

```json
{
  "status": "ok",
  "checks": {
    "database": "ok"
  }
}
```

#### 異常時（503 Service Unavailable）

```json
{
  "status": "degraded",
  "checks": {
    "database": "error"
  }
}
```

### 7.3 チェック内容

| チェック項目 | 方法 | タイムアウト |
|------------|------|------------|
| DB 接続 | `SELECT 1` の実行 | 5 秒 |

DB 接続チェックは、RLS 設定を行わないオーナーロール接続（またはプールからの直接取得）で実行する。ヘルスチェック自体がテナントコンテキストを持たないため、RLS バイパスが妥当である。

### 7.4 ALB ヘルスチェック設定

| 設定項目 | 値 |
|---------|-----|
| パス | `/health` |
| プロトコル | HTTP |
| 間隔 | 30 秒 |
| タイムアウト | 5 秒 |
| 正常判定回数 | 2 回連続 |
| 異常判定回数 | 2 回連続 |
| 成功コード | 200 |

### 7.5 ログ出力方針

ヘルスチェックのリクエストは通常のアクセスログには出力しない（ノイズ低減のため）。ヘルスチェック失敗時のみ WARN レベルで出力する。

```json
{
  "time": "2026-03-22T10:30:00.123Z",
  "level": "WARN",
  "msg": "health check failed",
  "component": "handler.health",
  "check": "database",
  "error": "dial tcp 10.0.1.100:5432: connect: connection refused"
}
```

---

## 8. ダッシュボード設計

### 8.1 概要

CloudWatch Logs Insights のアドホッククエリでダッシュボードを構成する。CloudWatch ダッシュボードに保存済みクエリとして登録する。ダッシュボードの構築は Step 7 Phase 2 で実施する（ADR-0005 記載）。

### 8.2 ダッシュボードパネル構成

| パネル名 | 種類 | 説明 |
|---------|------|------|
| リクエスト概況 | 数値 | 直近 1 時間のリクエスト数・エラー数・平均レスポンスタイム |
| レスポンスタイム推移 | 折れ線グラフ | p50 / p95 / p99 の時系列推移 |
| ステータスコード分布 | 棒グラフ | 2xx / 4xx / 5xx の件数推移 |
| エンドポイント別レスポンスタイム | テーブル | path 別の平均・p95・リクエスト数 |
| エラーログ一覧 | テーブル | 直近の ERROR / WARN ログ |
| セキュリティイベント | テーブル | ログイン失敗・認可拒否・レート制限の一覧 |

### 8.3 Logs Insights クエリ定義

#### リクエスト概況（直近 1 時間）

```
fields @timestamp, @message
| filter ispresent(status)
| stats count() as total_requests,
        sum(status >= 500) as error_5xx,
        sum(status >= 400 and status < 500) as error_4xx,
        avg(duration_ms) as avg_duration,
        pct(duration_ms, 95) as p95_duration
  by bin(5m)
| sort @timestamp desc
```

#### レスポンスタイム推移

```
fields @timestamp, duration_ms
| filter ispresent(duration_ms)
| stats pct(duration_ms, 50) as p50,
        pct(duration_ms, 95) as p95,
        pct(duration_ms, 99) as p99
  by bin(5m)
| sort @timestamp asc
```

#### エンドポイント別レスポンスタイム（上位 20）

```
fields path, duration_ms
| filter ispresent(duration_ms)
| stats count() as requests,
        avg(duration_ms) as avg_ms,
        pct(duration_ms, 95) as p95_ms,
        max(duration_ms) as max_ms
  by path
| sort requests desc
| limit 20
```

#### エラーログ一覧（直近 100 件）

```
fields @timestamp, level, msg, request_id, path, status, error
| filter level = "ERROR" or level = "WARN"
| sort @timestamp desc
| limit 100
```

#### セキュリティイベント一覧

```
fields @timestamp, msg, remote_ip, user_id, tenant_id, path
| filter msg = "login failed"
      or msg = "authorization denied"
      or msg = "cross-tenant access blocked"
      or msg = "rate limit exceeded"
      or msg = "invalid token rejected"
| sort @timestamp desc
| limit 100
```

#### テナント別リクエスト数（上位 10）

```
fields tenant_id
| filter ispresent(tenant_id)
| stats count() as requests,
        avg(duration_ms) as avg_ms
  by tenant_id
| sort requests desc
| limit 10
```

---

## 9. ログ保持ポリシー

### 9.1 CloudWatch Logs 保持期間

| ロググループ | 保持期間 | 理由 |
|------------|---------|------|
| `/ecs/expense-saas/api` | 30 日 | ADR-0005 で決定。MVP フェーズの運用コスト最適化 |

### 9.2 将来的な拡張

| フェーズ | 対応 |
|---------|------|
| テナント数増加時 | S3 へのログエクスポート（長期保存、コスト最適化） |
| コンプライアンス要件発生時 | 保持期間の延長（90 日 / 1 年等） |
| 監査ログ導入時（Phase 3） | 監査ログ専用のロググループ・保持ポリシーを別途定義 |

---

## 10. DB 接続監視

T1-6（db_schema.md）との接点として、DB 接続プールの監視方針を定義する。

### 10.1 コネクションプール監視

Go の `database/sql` パッケージの `DBStats` を利用して、コネクションプールの状態を定期的にログ出力する。

| 項目 | フィールド名 | 説明 |
|------|-----------|------|
| オープン接続数 | `db_open_connections` | 現在のオープン接続数 |
| 使用中接続数 | `db_in_use` | アクティブに使用中の接続数 |
| アイドル接続数 | `db_idle` | アイドル状態の接続数 |
| 待ち回数 | `db_wait_count` | 接続待ちが発生した累計回数 |
| 待ち時間 | `db_wait_duration_ms` | 接続待ちの累計時間（ミリ秒） |

### 10.2 出力方法

60 秒間隔のバックグラウンドゴルーチンで INFO レベルのログを出力する。

```json
{
  "time": "2026-03-22T10:30:00.000Z",
  "level": "INFO",
  "msg": "db pool stats",
  "component": "infra.db",
  "db_open_connections": 8,
  "db_in_use": 3,
  "db_idle": 5,
  "db_wait_count": 0,
  "db_wait_duration_ms": 0
}
```

`db_wait_count` が増加傾向にある場合、コネクションプールの枯渇が疑われるため WARN レベルに昇格する。

### 10.3 RDS メトリクスとの併用

DB 接続監視は以下の 2 層で行う。

| 層 | メトリクス | 取得方法 |
|----|-----------|---------|
| アプリケーション層 | コネクションプール使用状況 | Go `DBStats` → 構造化ログ |
| インフラ層 | RDS 接続数 | RDS 標準メトリクス `DatabaseConnections` |

アプリケーション層の待ち発生とインフラ層の接続数上限を併せて確認することで、ボトルネックの特定を容易にする。

---

## 11. request_id の伝播

### 11.1 生成と伝播フロー

```
[クライアント] ────→ [ALB] ────→ [RequestID ミドルウェア] ────→ [全ハンドラ/サービス]
                                        │
                                        ├── context に格納
                                        └── レスポンスヘッダー X-Request-ID に設定
```

1. RequestID ミドルウェアが UUID v4 を生成する
2. context に `request_id` として格納する
3. レスポンスヘッダー `X-Request-ID` に設定する（クライアントからの問い合わせ時に追跡可能）
4. Logger ミドルウェア以降、全てのログ出力に `request_id` が含まれる
5. サービス層・リポジトリ層でもログ出力時に `slog.InfoContext(ctx, ...)` を使用して `request_id` を自動伝播する

### 11.2 クライアント連携

フロントエンドでエラーが発生した場合、レスポンスヘッダーの `X-Request-ID` をエラー画面に表示する。ユーザーがサポートに問い合わせる際に `request_id` を伝えることで、CloudWatch Logs Insights で該当リクエストのログを即座に特定できる。

```
fields @timestamp, level, msg, method, path, status, duration_ms, error
| filter request_id = "550e8400-e29b-41d4-a716-446655440000"
| sort @timestamp asc
```

---

## 12. 上流成果物との差分

| 項目 | 上流の記載 | 本書の対応 | 差分の種類 |
|------|----------|-----------|----------|
| ヘルスチェック応答タイムアウト | ADR-0005: 5 秒 | §7.4 ALB 設定: タイムアウト 5 秒 | 整合（差分なし） |
| ログ保持期間 | ADR-0005: 30 日 | §9.1: 30 日 | 整合（差分なし） |
| メトリクスフィルタのメトリクス名 | ADR-0005: `API/5xxCount` 等 | §5.2: 名前空間を `ExpenseSaaS/API` に正式化 | 詳細化（名前空間を明示） |
| 4xx エラーレート | ADR-0005: 参考値 > 10% | §5.4: 参考値としてアラート対象外に分類 | 整合（差分なし） |
| p99 レスポンスタイム | ADR-0005: 参考値 > 1000ms | §5.4: 参考値としてアラート対象外に分類 | 整合（差分なし） |

上流成果物との不整合は検出されなかった。

# 運用 Runbook

## 1. この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 監視アラート検知後の一次対応手順と定常運用を定義する |
| 正本情報 | 障害一次対応手順、切り分け観点、定常確認項目、エスカレーション基準 |
| 扱わない内容 | 監視閾値・アラーム定義の正本（monitoring.md が正本）、リリース手順（release.md に委譲）、バックアップ・リストア手順（backup_restore.md に委譲） |
| 主な参照元 | `../50_detail_design/monitoring.md`, `../30_arch/architecture.md` |
| 主な参照先 | `./release.md`, `./backup_restore.md` |

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `50_detail_design/monitoring.md` | アラート種別・閾値・メトリクス定義（正本） |
| `30_arch/architecture.md` | システム構成・ミドルウェアチェーン |
| `30_arch/adr/0004-infra.md` | インフラ構成（ECS Fargate / RDS / S3） |
| `30_arch/adr/0005-monitoring-logging.md` | 監視・ログ戦略 |
| `70_operations/release.md` | ロールバック手順 |
| `70_operations/backup_restore.md` | データ復旧手順 |

---

## 2. 定常確認項目

### 2.1 日次確認

毎営業日の始業時に以下を確認する。確認者は運用担当者とする。

| # | 確認項目 | 確認方法 | 正常基準 |
|---|---------|---------|---------|
| 1 | 未対応アラートの有無 | CloudWatch Alarms コンソールで `ALARM` 状態のアラームを確認 | `ALARM` 状態のアラームが 0 件 |
| 2 | ECS タスク稼働状態 | ECS コンソールでサービスの Running Task Count を確認 | Running = Desired（2 タスク） |
| 3 | 5xx エラーの発生傾向 | CloudWatch Logs Insights で直近 24 時間の 5xx 件数を集計 | 5xx エラーレート < 1% |
| 4 | RDS 空きストレージ | RDS コンソールの FreeStorageSpace メトリクスを確認 | 残容量 > 20% |
| 5 | ヘルスチェック状態 | ALB ターゲットグループの Healthy/Unhealthy Host Count を確認 | Unhealthy = 0 |

**日次確認用 Logs Insights クエリ（5xx 件数）**:

```
fields @timestamp, status, path, error
| filter status >= 500
| stats count() as error_count by bin(1h)
| sort @timestamp desc
```

### 2.2 週次確認

毎週月曜日に以下を確認する。確認者は運用担当者とする。

| # | 確認項目 | 確認方法 | 正常基準 |
|---|---------|---------|---------|
| 1 | API レスポンスタイム p95 推移 | CloudWatch メトリクス `ExpenseSaaS/API/Duration` の週次推移を確認 | p95 < 500ms で安定 |
| 2 | RDS CPU 使用率推移 | RDS メトリクス `CPUUtilization` の週次推移を確認 | ピーク時 < 80% |
| 3 | RDS 接続数推移 | RDS メトリクス `DatabaseConnections` の週次推移を確認 | ピーク時 < max_connections の 80% |
| 4 | ECS CPU/メモリ推移 | Container Insights の週次推移を確認 | ピーク時 < 80% |
| 5 | ログイン失敗の傾向 | Logs Insights で `msg = "login failed"` の件数推移を確認 | 急激な増加傾向がないこと |
| 6 | ログ保持状況 | CloudWatch Logs のロググループ設定を確認 | 保持期間が 30 日に設定されていること |

**週次確認用 Logs Insights クエリ（ログイン失敗推移）**:

```
fields @timestamp, msg, remote_ip
| filter msg = "login failed"
| stats count() as failure_count by bin(1d)
| sort @timestamp desc
```

---

## 3. アラート別一次対応手順

monitoring.md SS6.2 で定義されたアラームに対する一次対応手順を定義する。閾値・評価条件の正本は monitoring.md を参照すること。

### 3.1 Critical アラート

#### ALERT-C1: HealthCheck-Failure（ヘルスチェック連続失敗）

**概要**: ALB のヘルスチェック（`GET /health`）が連続 2 回失敗した。ECS タスクまたは DB に問題がある可能性がある。

**一次対応手順**:

```
[1] ECS タスクの状態を確認する
    $ aws ecs describe-services \
        --cluster expense-saas-prod \
        --services expense-saas-api \
        --query 'services[0].{running:runningCount,desired:desiredCount,events:events[:3]}'

    確認観点:
    - runningCount < desiredCount であれば、タスクが起動に失敗している
    - events にエラーが含まれていないか

[2] ECS タスクのログを確認する
    CloudWatch Logs Insights（ロググループ: /ecs/expense-saas/api）:

    fields @timestamp, level, msg, error, component
    | filter level = "ERROR"
    | sort @timestamp desc
    | limit 20

    確認観点:
    - "health check failed" のログがあるか
    - "database connection failed" のエラーがあるか

[3] RDS の接続性を確認する
    $ aws rds describe-db-instances \
        --db-instance-identifier expense-saas-prod \
        --query 'DBInstances[0].{status:DBInstanceStatus,endpoint:Endpoint}'

    確認観点:
    - DBInstanceStatus が "available" であること
    - エンドポイントが正しいこと

[4] 切り分け判断
    - タスクが起動失敗 → [インフラ障害] ECS タスク定義・コンテナイメージを確認
    - タスクは起動しているが DB 接続エラー → [DB 障害] RDS の状態を確認
    - タスクもDBも正常だがヘルスチェック失敗 → [アプリ障害] アプリケーションログを詳細確認
```

**エスカレーション**: 一次対応開始から 15 分以内に復旧しない場合、エスカレーション（SS5 参照）。

---

#### ALERT-C2: High-5xx-Rate-Critical（5xx エラーレート > 5%）

**概要**: 5xx エラーレートが 5% を超過した。アプリケーションに重大な障害が発生している可能性がある。

**一次対応手順**:

```
[1] 5xx エラーの詳細を確認する
    CloudWatch Logs Insights:

    fields @timestamp, path, status, error, request_id
    | filter status >= 500
    | sort @timestamp desc
    | limit 50

    確認観点:
    - 特定のエンドポイントに集中しているか
    - エラーメッセージにパターンがあるか

[2] エラーの傾向を確認する
    fields @timestamp
    | filter status >= 500
    | stats count() as error_count by bin(1m)
    | sort @timestamp desc

    確認観点:
    - 特定の時刻から急増しているか（デプロイ直後の可能性）
    - 継続的に発生しているか、間欠的か

[3] DB の状態を確認する
    fields @timestamp, msg, error, component
    | filter component = "repository.postgres" and level = "ERROR"
    | sort @timestamp desc
    | limit 20

    確認観点:
    - DB 接続エラーが発生していないか
    - 特定のクエリでエラーが集中していないか

[4] 直近のデプロイとの関連を確認する
    - 直近にデプロイが実施されたか確認する
    - デプロイ直後から 5xx が増加している場合 → ロールバックを検討
      （ロールバック手順は release.md SS4 を参照）

[5] 切り分け判断
    - 特定エンドポイントに集中 → [アプリ障害] 該当ハンドラ・サービス層のコードを確認
    - DB 関連エラーが大量 → [DB 障害] RDS の状態・接続数を確認
    - デプロイ直後 → [リリース障害] ロールバックを実施
    - 全体的に発生 → [インフラ障害] ECS タスク・ネットワーク設定を確認
```

**エスカレーション**: 一次対応開始から 10 分以内に原因特定できない場合、または 15 分以内に復旧しない場合、エスカレーション（SS5 参照）。

---

### 3.2 Warning アラート

#### ALERT-W1: High-5xx-Rate-Warning（5xx エラーレート > 1%）

**概要**: 5xx エラーレートが 1% を超過した。軽度の障害またはスパイク。

**一次対応手順**:

```
[1] ALERT-C2 の [1]〜[3] と同じ手順でエラー詳細を確認する

[2] 一過性のスパイクか継続的かを判断する
    - 5 分間の観察で収束傾向にある場合 → 経過観察
    - 継続または増加傾向にある場合 → ALERT-C2 の対応に昇格

[3] 対応結果を記録する
```

---

#### ALERT-W2: High-p95-Latency（p95 レスポンスタイム > 500ms）

**概要**: API レスポンスタイムの p95 が 500ms を超過した。

**一次対応手順**:

```
[1] 遅延しているエンドポイントを特定する
    CloudWatch Logs Insights:

    fields path, duration_ms
    | filter ispresent(duration_ms)
    | stats count() as requests,
            avg(duration_ms) as avg_ms,
            pct(duration_ms, 95) as p95_ms,
            max(duration_ms) as max_ms
      by path
    | sort p95_ms desc
    | limit 10

[2] DB のパフォーマンスを確認する
    - RDS CPUUtilization を確認
    - RDS ReadLatency / WriteLatency を確認
    - アプリケーションログで "slow request detected"（WARN レベル）を検索する

    fields @timestamp, path, duration_ms, request_id
    | filter msg = "slow request detected"
    | sort duration_ms desc
    | limit 20

[3] コネクションプールの状態を確認する
    fields @timestamp, db_open_connections, db_in_use, db_wait_count
    | filter msg = "db pool stats"
    | sort @timestamp desc
    | limit 10

    確認観点:
    - db_wait_count が増加傾向にないか
    - db_in_use が上限に近づいていないか

[4] 切り分け判断
    - 特定エンドポイントが遅延 → [アプリ障害] 該当クエリの最適化を検討
    - DB 全体が遅延 → [DB 障害] RDS のリソース状況を確認
    - コネクションプール枯渇 → [アプリ障害] プール設定の見直し
    - ECS の CPU/メモリが高負荷 → [インフラ障害] スケーリングを検討
```

---

#### ALERT-W3: High-ECS-CPU / High-ECS-Memory（ECS タスク CPU/メモリ使用率 > 80%）

**概要**: ECS タスクのリソース使用率が閾値を超過した。

**一次対応手順**:

```
[1] Container Insights で使用率の推移を確認する
    - 急激な上昇か、緩やかな上昇かを判断する
    - 特定タスクに偏りがないかを確認する

[2] リクエスト数の増加を確認する
    fields @timestamp
    | filter ispresent(status)
    | stats count() as requests by bin(5m)
    | sort @timestamp desc

    確認観点:
    - 通常時と比較してリクエスト数が異常に多くないか

[3] 切り分け判断
    - リクエスト数増加に比例 → 正常な負荷増加。タスク数の増加を検討
    - リクエスト数は通常だがリソース消費が高い → メモリリーク等のアプリ障害を疑う
      → ECS タスクの再起動（ローリング更新）で一時対処

[4] 一時対処: ECS タスクのスケールアウト
    $ aws ecs update-service \
        --cluster expense-saas-prod \
        --service expense-saas-api \
        --desired-count 3
```

---

#### ALERT-W4: High-RDS-CPU（RDS CPU 使用率 > 80%）

**概要**: RDS の CPU 使用率が閾値を超過した。

**一次対応手順**:

```
[1] RDS Performance Insights でクエリ負荷を確認する
    - Top SQL を確認し、負荷が高いクエリを特定する

[2] アプリケーションログでスロークエリを確認する
    fields @timestamp, path, duration_ms
    | filter ispresent(duration_ms) and duration_ms > 500
    | sort duration_ms desc
    | limit 20

[3] DB 接続数を確認する
    - RDS メトリクス DatabaseConnections を確認する
    - 接続数が max_connections の 80% を超えている場合は ALERT-W6 の対応も実施する

[4] 切り分け判断
    - 特定クエリが原因 → クエリ最適化・インデックス追加を検討
    - 全体的に高負荷 → RDS インスタンスのスケールアップを検討（db.t3.micro → db.t3.small）
```

---

#### ALERT-W5: Low-RDS-Storage（RDS 空きストレージ < 20%）

**概要**: RDS の空きストレージ容量が 20% を下回った。

**一次対応手順**:

```
[1] 現在のストレージ使用量を確認する
    $ aws rds describe-db-instances \
        --db-instance-identifier expense-saas-prod \
        --query 'DBInstances[0].AllocatedStorage'

    $ aws cloudwatch get-metric-statistics \
        --namespace AWS/RDS \
        --metric-name FreeStorageSpace \
        --dimensions Name=DBInstanceIdentifier,Value=expense-saas-prod \
        --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
        --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
        --period 300 \
        --statistics Average

[2] ストレージ消費の傾向を確認する
    - 急激な消費か、緩やかな消費かを判断する
    - 急激な場合: 不正なデータ投入や異常なバッチ処理を疑う
    - 緩やかな場合: 自然なデータ増加。計画的なストレージ拡張を実施

[3] 不要データの確認
    - 期限切れのリフレッシュトークン（refresh_tokens.expires_at < now()）
    - 論理削除済みの古いレコード（deleted_at が一定期間以上前）

[4] 対処: ストレージの拡張
    - RDS コンソールからストレージサイズを変更する（ダウンタイムなし）
    - ストレージは拡張のみ可能で縮小はできないことに注意
```

---

#### ALERT-W6: High-RDS-Connections（RDS 接続数 > max_connections の 80%）

**概要**: RDS の接続数が上限の 80% を超過した。

**一次対応手順**:

```
[1] 現在の接続数を確認する
    $ aws cloudwatch get-metric-statistics \
        --namespace AWS/RDS \
        --metric-name DatabaseConnections \
        --dimensions Name=DBInstanceIdentifier,Value=expense-saas-prod \
        --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
        --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
        --period 60 \
        --statistics Maximum

[2] アプリケーション側のコネクションプール状態を確認する
    CloudWatch Logs Insights:

    fields @timestamp, db_open_connections, db_in_use, db_idle, db_wait_count
    | filter msg = "db pool stats"
    | sort @timestamp desc
    | limit 10

[3] 切り分け判断
    - アプリ側の接続数が多い → コネクションプールの MaxOpenConns 設定を見直す
    - アプリ側の接続数は正常だが RDS 接続数が多い → 他クライアント（マイグレーションツール等）からの接続を疑う

[4] 一時対処
    - 不要なアイドル接続の切断
    - コネクションプールの MaxOpenConns / MaxIdleConns の調整
```

---

#### ALERT-W7: High-Login-Failures（ログイン失敗 > 50 回/5 分）

**概要**: 短時間に大量のログイン失敗が発生した。ブルートフォース攻撃の可能性がある。

**一次対応手順**:

```
[1] ログイン失敗の詳細を確認する
    CloudWatch Logs Insights:

    fields @timestamp, remote_ip, email_hash
    | filter msg = "login failed"
    | sort @timestamp desc
    | limit 100

[2] 攻撃パターンを分析する
    fields remote_ip
    | filter msg = "login failed"
    | stats count() as failures by remote_ip
    | sort failures desc
    | limit 20

    確認観点:
    - 特定 IP アドレスに集中しているか（ブルートフォース攻撃の疑い）
    - 特定 email_hash に集中しているか（アカウント狙い撃ち）
    - 広範な IP から分散しているか（ボットネット攻撃の疑い）

[3] レート制限の動作を確認する
    - ログイン試行のレート制限（5 req/min/IP）が正常に機能しているか
    - "rate limit exceeded" のログを確認する

    fields @timestamp, remote_ip
    | filter msg = "rate limit exceeded"
    | sort @timestamp desc
    | limit 20

[4] 対処
    - 特定 IP に集中 → WAF（将来導入時）でのブロックを検討
    - MVP ではレート制限（5 req/min/IP）が防御層として機能する
    - 攻撃が大規模で、レート制限では不十分な場合 → エスカレーション
```

---

## 4. よくある障害パターンと切り分け

### 4.1 障害切り分けフローチャート

```
[アラート受信]
    |
    v
[ヘルスチェック失敗?] -- Yes --> [ECS タスク起動状態確認]
    |                                   |
    | No                          [タスク起動失敗?] -- Yes --> [インフラ障害]
    v                                   |                      コンテナイメージ・設定を確認
[5xx エラー発生?]                  | No
    |                          [DB 接続エラー?] -- Yes --> [DB 障害]
    | Yes                              |
    v                              | No
[全エンドポイントで発生?]         [アプリ障害]
    |                              アプリログ詳細確認
    | Yes --> [DB/インフラ障害を疑う]
    |
    | No（特定エンドポイント）
    v
[デプロイ直後?] -- Yes --> [リリース障害]
    |                      ロールバック検討（release.md SS4）
    | No
    v
[アプリ障害]
    該当コードの確認
```

### 4.2 障害カテゴリ別の確認コマンド

#### アプリケーション障害

```bash
# ERROR レベルのログを直近から確認
# CloudWatch Logs Insights で以下のクエリを実行:
fields @timestamp, level, msg, error, path, request_id
| filter level = "ERROR"
| sort @timestamp desc
| limit 50

# 特定の request_id でリクエストを追跡
fields @timestamp, level, msg, method, path, status, duration_ms, error, component
| filter request_id = "<request_id>"
| sort @timestamp asc
```

#### DB 障害

```bash
# RDS インスタンスの状態確認
$ aws rds describe-db-instances \
    --db-instance-identifier expense-saas-prod \
    --query 'DBInstances[0].{status:DBInstanceStatus,cpu:PerformanceInsightsEnabled,storage:AllocatedStorage}'

# DB 関連のエラーログを確認
# CloudWatch Logs Insights:
fields @timestamp, msg, error, component
| filter component = "repository.postgres" and level = "ERROR"
| sort @timestamp desc
| limit 20

# コネクションプール状態の確認
fields @timestamp, db_open_connections, db_in_use, db_idle, db_wait_count, db_wait_duration_ms
| filter msg = "db pool stats"
| sort @timestamp desc
| limit 10
```

#### インフラ障害

```bash
# ECS サービスの状態確認
$ aws ecs describe-services \
    --cluster expense-saas-prod \
    --services expense-saas-api \
    --query 'services[0].{running:runningCount,desired:desiredCount,status:status}'

# ECS タスクの停止理由確認（直近の停止タスク）
$ aws ecs list-tasks \
    --cluster expense-saas-prod \
    --service-name expense-saas-api \
    --desired-status STOPPED \
    --query 'taskArns[:5]'

$ aws ecs describe-tasks \
    --cluster expense-saas-prod \
    --tasks <task_arn> \
    --query 'tasks[0].{stoppedReason:stoppedReason,stopCode:stopCode}'

# ALB ターゲットグループのヘルス状態
$ aws elbv2 describe-target-health \
    --target-group-arn <target_group_arn>
```

---

## 5. エスカレーション基準

### 5.1 エスカレーションの判断基準

| 条件 | エスカレーション先 | 方法 |
|------|------------------|------|
| Critical アラートが 15 分以内に復旧しない | 開発チームリード | メール + チャット |
| Warning アラートが 1 時間以内に復旧しない | 開発チームリード | メール |
| 一次対応で原因を特定できない | 開発チームリード | メール + チャット |
| データ損失の可能性がある | 開発チームリード + プロダクトオーナー | メール + 電話 |
| セキュリティインシデントの疑い（大規模なブルートフォース攻撃等） | 開発チームリード + セキュリティ担当 | メール + 電話 |

### 5.2 エスカレーション時の報告テンプレート

```
件名: [Critical/Warning] <アラーム名> - <簡潔な状況>

1. 発生時刻: YYYY-MM-DD HH:MM (UTC)
2. アラーム名: <monitoring.md のアラーム名>
3. 影響範囲:
   - 影響を受けるサービス/機能:
   - 影響を受けるユーザー数（推定）:
4. 現在の状況:
   - 一次対応の実施内容:
   - 現時点の切り分け結果:
5. 必要な対応:
   - 依頼事項:
6. 参照情報:
   - CloudWatch Logs の request_id:
   - 関連する Logs Insights クエリの時間範囲:
```

### 5.3 インシデント対応のタイムライン

| 経過時間 | Critical | Warning |
|---------|----------|---------|
| 0 分 | アラート受信、一次対応開始 | アラート受信、一次対応開始 |
| 5 分 | 切り分け判断 | 切り分け判断 |
| 15 分 | 復旧しない場合エスカレーション | 経過観察継続 |
| 30 分 | 状況アップデート報告 | - |
| 60 分 | - | 復旧しない場合エスカレーション |
| 以降 | 30 分ごとに状況アップデート | 1 時間ごとに状況アップデート |

---

## 6. 品質チェック

- [x] monitoring.md SS6.2 の全 Critical/Warning アラームに対応する一次対応手順がある
  - Critical: HealthCheck-Failure, High-5xx-Rate-Critical
  - Warning: High-5xx-Rate-Warning, High-p95-Latency, High-ECS-CPU, High-ECS-Memory, High-RDS-CPU, Low-RDS-Storage, High-RDS-Connections, High-Login-Failures
- [x] 各対応手順に確認コマンド・Logs Insights クエリが含まれている
- [x] 切り分け観点（アプリ障害/DB障害/インフラ障害）が明確である
- [x] エスカレーション基準が具体的な条件（時間・状況）で定義されている
- [x] 日次/週次の定常確認項目が定義されている
- [x] 監視閾値の再定義を行っていない（monitoring.md を正本として参照のみ）
- [x] ロールバック手順・バックアップリストア手順は他文書に委譲し、参照リンクを記載している

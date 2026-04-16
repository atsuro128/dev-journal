# リリース手順

## 1. この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 安全に本番リリースするための確認手順・デプロイ手順・ロールバック条件を定義する |
| 正本情報 | リリース前後確認チェックリスト、デプロイ手順、ロールバック条件・手順 |
| 扱わない内容 | CI/CD パイプライン設計の正本（architecture.md に委譲）、環境変数の詳細（env_config.md に委譲）、アラート対応（runbook.md に委譲） |
| 主な参照元 | `../30_arch/architecture.md`, `../30_arch/diagrams.md` |
| 主な参照先 | `./runbook.md`, `./env_config.md` |

### ポートフォリオ対応

本文書は実務想定の構成（ECS Fargate）で記載している。ポートフォリオでの実装は [ADR-0004](../30_arch/adr/0004-infra.md) §ポートフォリオ対応を参照し、コンピュートを EC2 t3.micro に読み替えること。デプロイ手順（§4.2）・ロールバック手順（§7）の具体コマンドは EC2 向けに Step 11 で再設計する。リリース前後の確認項目（§3, §5）・ロールバック条件（§6）はコンピュート方式に依存しないため、そのまま適用可能。

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `30_arch/architecture.md` | デプロイパイプライン概要、システム構成 |
| `30_arch/adr/0004-infra.md` | インフラ構成（ECS Fargate / RDS / GitHub Actions） |
| `50_detail_design/monitoring.md` | アラート閾値（リリース後確認の基準） |
| `70_operations/env_config.md` | 環境変数一覧・シークレット管理 |
| `70_operations/runbook.md` | 障害発生時の対応手順 |
| `70_operations/backup_restore.md` | データ復旧手順 |

---

## 2. リリースフロー概要

```
[1] リリース前確認（SS3）
     |
[2] DB マイグレーション（該当する場合のみ、SS4.1）
     |
[3] アプリケーションデプロイ（SS4.2）
     |
[4] リリース後確認（SS5）
     |
[5] ロールバック判断（SS6 の条件に該当する場合）
     |--- Yes --> ロールバック実行（SS7）
     |
[6] リリース完了
```

---

## 3. リリース前確認チェックリスト

リリース実施前に以下の全項目を確認する。1 項目でも NG の場合はリリースを中止する。

### 3.1 コード品質

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 1 | ローカルテストが全て通過している | PR body の Local CI セクションを確認 | [ ] |
| 2 | lint エラーが 0 件である | ローカルの lint 実行結果を確認 | [ ] |
| 3 | 全テストが通過している | ローカルの test 実行結果を確認 | [ ] |
| 4 | Docker イメージのビルドが成功している | ローカルの build 実行結果を確認 | [ ] |

### 3.2 DB マイグレーション

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 5 | 未適用のマイグレーションがあるか確認した | `db/migrations/` ディレクトリの最新ファイルとステージング環境の適用状態を比較 | [ ] |
| 6 | マイグレーションがステージングで正常に実行済み | ステージング環境でのマイグレーション実行結果を確認 | [ ] |
| 7 | マイグレーションにロールバック用の down ファイルが存在する | 対応する `*.down.sql` ファイルの存在を確認 | [ ] |
| 8 | マイグレーションが既存データに悪影響を与えないことを確認した | ステージング環境での既存データ確認 | [ ] |

### 3.3 環境・設定

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 9 | 新しい環境変数の追加が必要な場合、本番環境に設定済みである | env_config.md と ECS タスク定義を照合 | [ ] |
| 10 | シークレット値の追加・変更が必要な場合、Secrets Manager に設定済みである | AWS Secrets Manager コンソールで確認 | [ ] |

### 3.4 デプロイ前最終確認

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 11 | 現在の本番環境が正常稼働している | ヘルスチェック OK、アラーム状態 ALARM なし | [ ] |
| 12 | 本番 RDS のバックアップが直近で取得されている | RDS コンソールで最新スナップショットの時刻を確認 | [ ] |
| 13 | リリース担当者がロールバック手順を把握している | 本書 SS7 を確認済み | [ ] |

---

## 4. デプロイ手順

### 4.1 DB マイグレーション（該当する場合のみ）

DB マイグレーションが必要な場合、アプリケーションデプロイの前に実行する。

```
[前提]
- 本番 RDS のスナップショットを取得済みであること（SS3.4 の #12）
- マイグレーションツール: golang-migrate

[手順]

1. 本番 DB へのマイグレーション実行
   マイグレーション用の踏み台（bastion）から実行する。

   $ export DATABASE_URL="postgres://expense_owner:<password>@<rds-endpoint>:5432/expense_saas?sslmode=require"
   $ migrate -path db/migrations -database "${DATABASE_URL}" up

   確認観点:
   - "no change" ではなく、期待するバージョンに進んだこと
   - エラーが発生していないこと

2. マイグレーション結果の確認
   $ migrate -path db/migrations -database "${DATABASE_URL}" version

   期待結果: 最新のマイグレーションバージョン番号が表示される

3. マイグレーション失敗時
   - エラーメッセージを記録する
   - down マイグレーションでロールバックする:
     $ migrate -path db/migrations -database "${DATABASE_URL}" down 1
   - リリースを中止する
```

**マイグレーション方針**:
- 後方互換性のあるマイグレーションを優先する（カラム追加はデフォルト値付き、カラム削除は段階的に実施）
- 破壊的変更（カラム型変更・テーブル削除等）はダウンタイムを伴うため、メンテナンスウィンドウで実施する

### 4.2 アプリケーションデプロイ

手動でデプロイする（deploy.yml は参考用デモとしてリポジトリに残す）。

```
[デプロイフロー]

1. ローカルテストの完了確認
   - PR body の Local CI セクションで lint / test / build が全て通過していることを確認

2. ビルド・イメージ作成
   a. フロントエンドビルド（npm run build）
   b. Docker イメージビルド（Go バイナリ + SPA 静的ファイル embed）
   c. ECR へのイメージプッシュ

3. デプロイ実行
   a. ECS タスク定義の更新（新しいイメージタグを指定）
   b. ECS サービスの更新（ローリングアップデート）

4. ローリングアップデートの動作
   - 新タスクが起動し、ALB ヘルスチェックに合格するまで待機
   - 旧タスクは新タスクが正常稼働後にドレイニング → 停止
   - minimumHealthyPercent: 100%（旧タスクは新タスクが正常になるまで停止しない）
   - maximumPercent: 200%（新旧タスクが一時的に共存する）
```

**手動デプロイ（緊急時）**:

```bash
# 1. 最新イメージの確認
$ aws ecr describe-images \
    --repository-name expense-saas \
    --query 'sort_by(imageDetails,& imagePushedAt)[-1].imageTags'

# 2. タスク定義の更新（新しいイメージタグに差し替え）
$ aws ecs register-task-definition \
    --cli-input-json file://task-definition.json

# 3. サービスの更新
$ aws ecs update-service \
    --cluster expense-saas-prod \
    --service expense-saas-api \
    --task-definition expense-saas-api:<new_revision> \
    --force-new-deployment

# 4. デプロイ状況の監視
$ aws ecs wait services-stable \
    --cluster expense-saas-prod \
    --services expense-saas-api
```

### 4.3 デプロイ所要時間の目安

| フェーズ | 所要時間 |
|---------|---------|
| ローカルテスト（lint + test + build） + イメージ push | 3-5 分 |
| ECS ローリングアップデート | 3-5 分 |
| 合計 | 6-10 分 |

---

## 5. リリース後確認チェックリスト

デプロイ完了後、以下の全項目を確認する。確認はデプロイ完了後 15 分以内に実施する。

### 5.1 即時確認（デプロイ完了後 5 分以内）

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 1 | ヘルスチェックが正常である | `curl -s https://<domain>/health` で `{"status":"ok"}` を確認 | [ ] |
| 2 | ECS タスクが正常に起動している | ECS コンソールで Running Task Count = Desired を確認 | [ ] |
| 3 | 旧タスクが停止している | ECS コンソールで STOPPED タスクの停止理由が正常終了であること | [ ] |
| 4 | CloudWatch Alarms に新たな ALARM がない | CloudWatch Alarms コンソールを確認 | [ ] |

### 5.2 機能確認（デプロイ完了後 15 分以内）

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 5 | ログイン機能が正常である | テスト用アカウントでログインを試行 | [ ] |
| 6 | 経費レポート一覧が表示される | ログイン後、レポート一覧ページにアクセス | [ ] |
| 7 | 経費レポート作成が正常に動作する | テスト用レポートを作成し、保存を確認 | [ ] |
| 8 | API レスポンスタイムが正常範囲内である | CloudWatch Logs Insights で直近 5 分のレスポンスタイムを確認 | [ ] |

**API レスポンスタイム確認クエリ**:

```
fields path, duration_ms
| filter ispresent(duration_ms)
| stats count() as requests,
        avg(duration_ms) as avg_ms,
        pct(duration_ms, 95) as p95_ms
  by path
| sort requests desc
| limit 10
```

### 5.3 エラー率確認（デプロイ完了後 15 分以内）

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 9 | 5xx エラーが発生していない | 直近 15 分の 5xx エラー件数を Logs Insights で確認 | [ ] |
| 10 | 4xx エラーに異常な増加がない | 直近 15 分の 4xx エラー件数を Logs Insights で確認 | [ ] |

**エラー率確認クエリ**:

```
fields @timestamp, status
| filter ispresent(status)
| stats count() as total,
        sum(status >= 500) as errors_5xx,
        sum(status >= 400 and status < 500) as errors_4xx
  by bin(1m)
| sort @timestamp desc
```

---

## 6. ロールバック条件

以下のいずれかの条件に該当する場合、ロールバックを実施する。

| # | ロールバック条件 | 判断基準 |
|---|----------------|---------|
| 1 | ヘルスチェックが復旧しない | デプロイ後 5 分以上ヘルスチェックが失敗し続ける |
| 2 | 5xx エラーレートが急増した | デプロイ前と比較して 5xx エラーレートが 5% を超過 |
| 3 | 主要機能が動作しない | ログイン・レポート一覧・レポート作成のいずれかが利用不可 |
| 4 | ECS タスクが起動しない | 新タスクが 3 回以上連続で起動に失敗 |
| 5 | レスポンスタイムが大幅に劣化した | p95 レスポンスタイムがデプロイ前の 3 倍以上、かつ 1000ms を超過 |

**判断者**: リリース担当者が上記条件に基づき判断する。判断に迷う場合は開発チームリードにエスカレーションする。

---

## 7. ロールバック手順

### 7.1 アプリケーションのロールバック

ECS タスク定義を前のリビジョンに戻すことでロールバックする。

```
[手順]

1. 現在のタスク定義リビジョンを確認する
   $ aws ecs describe-services \
       --cluster expense-saas-prod \
       --services expense-saas-api \
       --query 'services[0].taskDefinition'

   出力例: arn:aws:ecs:ap-northeast-1:123456789012:task-definition/expense-saas-api:15

2. 前のリビジョンを確認する（:15 の場合 :14 に戻す）
   $ aws ecs describe-task-definition \
       --task-definition expense-saas-api:14 \
       --query 'taskDefinition.{image:containerDefinitions[0].image,revision:revision}'

3. ロールバックを実行する
   $ aws ecs update-service \
       --cluster expense-saas-prod \
       --service expense-saas-api \
       --task-definition expense-saas-api:14 \
       --force-new-deployment

4. ロールバックの完了を待機する
   $ aws ecs wait services-stable \
       --cluster expense-saas-prod \
       --services expense-saas-api

5. ロールバック後の確認
   - SS5.1 の即時確認チェックリストを実行する
   - ヘルスチェックが正常であること
   - 5xx エラーが収束していること
```

### 7.2 DB マイグレーションのロールバック

マイグレーションが原因でロールバックが必要な場合。

```
[手順]

1. 適用したマイグレーションのバージョンを確認する
   $ migrate -path db/migrations -database "${DATABASE_URL}" version

2. 1 つ前のバージョンに戻す
   $ migrate -path db/migrations -database "${DATABASE_URL}" down 1

3. ロールバック後のデータ確認
   - 主要テーブル（expense_reports, expense_items, attachments）のレコード数が
     ロールバック前と同等であること
   - テスト用アカウントでの操作が正常であること

注意:
- down マイグレーションでデータが失われる可能性がある場合は、
  事前に取得したスナップショットからのリストアを検討する
  （復旧手順は backup_restore.md SS4 を参照）
```

### 7.3 ロールバック所要時間の目安

| 種類 | 所要時間 |
|------|---------|
| アプリケーションロールバック（ECS タスク定義の戻し） | 3-5 分 |
| DB マイグレーションロールバック（down 1） | 1-3 分 |
| RDS スナップショットからのリストア | 15-30 分（backup_restore.md 参照） |

---

## 8. リリース記録

各リリースの実施内容と結果を記録する。

### 8.1 リリース記録テンプレート

```
## リリース YYYY-MM-DD

- リリース担当: <名前>
- デプロイ開始: HH:MM (UTC)
- デプロイ完了: HH:MM (UTC)
- リリース内容:
  - <変更内容のサマリー>
- DB マイグレーション: あり / なし
- リリース前確認: OK / NG（NG の場合は理由）
- リリース後確認: OK / NG（NG の場合は理由）
- ロールバック: 実施 / 未実施（実施の場合は理由）
- 備考:
```

---

## 9. 品質チェック

- [x] リリース前確認チェックリストが CI 通過・DB マイグレーション・環境設定を網羅している
- [x] デプロイ手順が手動デプロイ + ECS ローリングアップデートの構成と整合している（architecture.md / ADR-0004）
- [x] ロールバック条件が具体的な数値基準で定義されている
- [x] ロールバック手順にアプリケーション・DB マイグレーションの両方が含まれている
- [x] ロールバック所要時間の目安が記載されている
- [x] リリース後確認チェックリストにヘルスチェック・機能確認・エラー率確認が含まれている
- [x] 環境変数の詳細は env_config.md に委譲し、重複定義していない
- [x] アラート対応は runbook.md に委譲し、参照リンクを記載している
- [x] デプロイ基盤の設計は architecture.md に委譲している

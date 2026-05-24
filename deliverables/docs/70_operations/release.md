# リリース手順

## 1. この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 安全に本番リリースするための確認手順・デプロイ手順・ロールバック条件を定義する |
| 正本情報 | リリース前後確認チェックリスト、デプロイ手順、ロールバック条件・手順 |
| 扱わない内容 | CI/CD パイプライン設計の正本（architecture.md に委譲）、環境変数の詳細（env_config.md に委譲）、アラート対応（runbook.md に委譲） |
| 主な参照元 | `../30_arch/architecture.md`, `../30_arch/diagrams.md` |
| 主な参照先 | `./runbook.md`, `./env_config.md` |

### 前提構成

本文書は EC2 t3.micro 単一インスタンス構成（[ADR-0004](../30_arch/adr/0004-infra.md) §ポートフォリオ対応）を前提に記述する。デプロイ方式は ECR pull + `systemctl restart`（issue #186 UD-6=A）。Terraform 実装の SSM 移行は issue #187 を参照。

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `30_arch/architecture.md` | デプロイパイプライン概要、システム構成 |
| `30_arch/adr/0004-infra.md` | インフラ選定（コンピュート / DB / CI/CD）。本文書では EC2 t3.micro 単一インスタンス（§ポートフォリオ対応）の前提に従う |
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
| 9 | 新しい環境変数の追加が必要な場合、本番環境に設定済みである | env_config.md と SSM Parameter Store パラメータ + systemd EnvironmentFile を照合（issue #186 UD-5=B 反映） | [ ] |
| 10 | シークレット値の追加・変更が必要な場合、SSM Parameter Store（SecureString）に設定済みである | `aws ssm get-parameters --names /expense-saas/prod/database/url /expense-saas/prod/database/app_url /expense-saas/prod/jwt/private_key /expense-saas/prod/jwt/public_key --with-decryption` で値が最新であることを確認（env_config.md §5.2 / §5.3 / issue #186 UD-5=B 反映） | [ ] |

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
   SSM Session Manager 経由で EC2 に接続し、EC2 上でアプリイメージを使ったワンショット
   `docker run --rm` で実行する（issue #186 UD-1=A / db_schema.md §マイグレーション運用 反映、bastion は使用しない）。

   $ aws ssm start-session --target i-xxxxxxxxx
   $ sudo docker run --rm \
       --env-file /etc/expense-saas/app.env \
       expense-saas:portfolio \
       /app/migrate up

   補足:
   - `--env-file /etc/expense-saas/app.env` から `DATABASE_URL`（owner ロール）を注入する。
   - `/app/migrate` はアプリイメージに同梱された golang-migrate バイナリ（パス・コマンド名はビルド成果物に合わせる）。
   - 直接 `migrate` CLI を EC2 上で実行する代替手順を採る場合も、DATABASE_URL は app.env（SSM Parameter Store 由来）から読み出すこと。

   確認観点:
   - "no change" ではなく、期待するバージョンに進んだこと
   - エラーが発生していないこと

2. マイグレーション結果の確認
   $ sudo docker run --rm \
       --env-file /etc/expense-saas/app.env \
       expense-saas:portfolio \
       /app/migrate version

   期待結果: 最新のマイグレーションバージョン番号が表示される

3. マイグレーション失敗時
   - エラーメッセージを記録する
   - down マイグレーションでロールバックする:
     $ sudo docker run --rm \
         --env-file /etc/expense-saas/app.env \
         expense-saas:portfolio \
         /app/migrate down 1
   - リリースを中止する
```

**マイグレーション方針**:
- 後方互換性のあるマイグレーションを優先する（カラム追加はデフォルト値付き、カラム削除は段階的に実施）
- 破壊的変更（カラム型変更・テーブル削除等）はダウンタイムを伴うため、メンテナンスウィンドウで実施する

### 4.2 アプリケーションデプロイ

EC2 t3.micro 単一インスタンス上で、ECR から新イメージを pull して `systemctl restart` でアプリを再起動する方式（issue #186 UD-6=A）。t3.micro 上でのビルドは行わない（リソース過食リスク回避）。ローリングアップデートは構成上不可能で、再起動の数秒〜十数秒のダウンタイムが発生する。

```
[デプロイフロー]

1. ローカルテストの完了確認
   - PR body の Local CI セクションで lint / test / build が全て通過していることを確認

2. ビルド・イメージ作成（開発者ローカル or GitHub Actions）
   a. フロントエンドビルド（npm run build）
   b. Docker イメージビルド（Go バイナリ + SPA 静的ファイル embed）
   c. ECR へのイメージプッシュ

3. EC2 上での再起動（SSM Session Manager 経由、issue #186 UD-1=A 反映）
   a. SSM 経由で EC2 に接続
   b. ECR から新イメージを pull
   c. pull したイメージをローカルタグ `expense-saas:portfolio` に付け替える
      （systemd unit は固定タグ `expense-saas:portfolio` を参照するため、
       user_data.sh.tpl L89-93 と同じ手順）
   d. systemctl restart で旧コンテナ停止 → 新コンテナ起動
```

**デプロイ手順（具体コマンド）**:

```bash
# a. ECR にイメージを push（ローカル or GitHub Actions から）
$ docker build -t <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG> .
$ docker push <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG>

# b. EC2 上で SSM Session Manager 経由で接続（issue #186 UD-1=A 反映）
$ aws ssm start-session --target i-xxxxxxxxx

# c. ECR から新イメージを pull
$ sudo docker pull <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG>

# d. ローカルタグ `expense-saas:portfolio` に付け替え（systemd unit は固定タグ参照）
#    systemd unit ファイルの書き換えは不要（user_data.sh.tpl と同じ方式）
$ sudo docker tag <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG> expense-saas:portfolio

# e. アプリを再起動（旧コンテナ停止 → 新コンテナ起動、ダウンタイム数秒〜十数秒）
$ sudo systemctl restart expense-saas

# f. ヘルスチェック疎通確認
$ curl -fsS http://localhost:8080/health
```

**補足**:
- EC2 単一インスタンス構成のため、ローリングアップデートは不可能。`systemctl restart` 中の数秒〜十数秒はリクエストが落ちる前提（ALB ヘルスチェックは数秒で unhealthy → healthy に復帰する想定）。
- GitHub Actions からの自動 pull/restart は SSM RunCommand 等で実装可能だが、MVP では手動運用とする（issue #187 で Terraform 移行と合わせて検討）。

### 4.3 デプロイ所要時間の目安

| フェーズ | 所要時間 |
|---------|---------|
| ローカルテスト（lint + test + build） + イメージ push | 3-5 分 |
| EC2 上の `docker pull` + `systemctl restart` | 1-2 分（ダウンタイム数秒〜十数秒） |
| 合計 | 4-7 分 |

---

## 5. リリース後確認チェックリスト

デプロイ完了後、以下の全項目を確認する。確認はデプロイ完了後 15 分以内に実施する。

### 5.1 即時確認（デプロイ完了後 5 分以内）

| # | 確認項目 | 確認方法 | 合否 |
|---|---------|---------|------|
| 1 | ヘルスチェックが正常である | `curl -s https://<domain>/health` で `{"status":"ok"}` を確認 | [ ] |
| 2 | EC2 上のアプリコンテナが正常に起動している | SSM 経由で EC2 に接続し `sudo docker ps` で expense-saas コンテナが Up 状態であることを確認、`sudo systemctl status expense-saas` で active (running) を確認 | [ ] |
| 3 | 旧コンテナが停止している | `sudo docker ps -a` で expense-saas の旧コンテナが Exited (0) で停止していることを確認 | [ ] |
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

以下のいずれかの条件に該当する場合、ロールバックを実施する。条件 (a)(b)(c) は issue #186 §実装計画 §2.6 I-02 に基づく EC2 構成固有の判定基準。

| # | ロールバック条件 | 判断基準 |
|---|----------------|---------|
| (a) | `systemctl restart` が連続失敗する | EC2 上で `sudo systemctl restart expense-saas` を 3 回連続実行しても `sudo systemctl status expense-saas` が `active (running)` にならない |
| (b) | ALB ヘルスチェックが復旧しない | `/health` への ALB ヘルスチェックが 5 分以上連続 unhealthy（`aws elbv2 describe-target-health --target-group-arn <ARN>` で `State=unhealthy` が継続） |
| (c) | 5xx エラーレートのアラートが発火 | ALERT-C2「5xx エラーレート > 5%」がデプロイ後 5 分以内に発火（monitoring.md 参照） |
| (d) | 主要機能が動作しない | ログイン・レポート一覧・レポート作成のいずれかが利用不可 |
| (e) | レスポンスタイムが大幅に劣化した | p95 レスポンスタイムがデプロイ前の 3 倍以上、かつ 1000ms を超過 |

**判定の優先**: (a)(b)(c) のいずれか 1 つを満たした時点でロールバックを開始する（issue #186 UD-6=A 採用に伴う合意）。(d)(e) は補助的な判断材料として併用する。

**判断者**: リリース担当者が上記条件に基づき判断する。判断に迷う場合は開発チームリードにエスカレーションする。

---

## 7. ロールバック手順

### 7.1 アプリケーションのロールバック

**前バージョンの ECR イメージタグを使って再デプロイすることでロールバックする**（具体的には §4.2 手順の `<TAG>` を前バージョンのタグに置き換えて実行する）。EC2 単一インスタンス構成のため、再 pull + restart で前バージョンに戻す。

```
[手順]

1. ロールバック対象の前バージョンタグを特定する
   ECR のイメージ一覧から、現在稼働中のタグの 1 つ前を確認する。

   $ aws ecr describe-images \
       --repository-name expense-saas \
       --query 'sort_by(imageDetails,& imagePushedAt)[-2:].[imageTags,imagePushedAt]'

   出力例: 直近 2 件のタグとプッシュ時刻が表示される。
   最新が問題のあるバージョン、その 1 つ前がロールバック対象。

2. SSM Session Manager 経由で EC2 に接続する（UD-1=A 反映）
   $ aws ssm start-session --target i-xxxxxxxxx

3. 前バージョンのイメージを ECR から pull する
   $ sudo docker pull <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<前TAG>

4. 前バージョンを ローカルタグ `expense-saas:portfolio` に付け替える
   （systemd unit は固定タグ参照のため unit ファイル編集は不要、user_data.sh.tpl と同じ方式）
   $ sudo docker tag <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<前TAG> expense-saas:portfolio

5. アプリを再起動する（ダウンタイム数秒〜十数秒）
   $ sudo systemctl restart expense-saas

6. ロールバック完了確認
   $ sudo systemctl status expense-saas   # active (running) であること
   $ sudo docker ps                       # expense-saas コンテナが Up 状態
   $ curl -fsS http://localhost:8080/health

7. ロールバック後の確認
   - SS5.1 の即時確認チェックリストを実行する
   - ALB ヘルスチェックが healthy に復帰していること
     $ aws elbv2 describe-target-health --target-group-arn <ARN>
   - 5xx エラーが収束していること（ALERT-C2 が clear になること）
```

### 7.2 DB マイグレーションのロールバック

マイグレーションが原因でロールバックが必要な場合。

```
[手順]

1. 適用したマイグレーションのバージョンを確認する
   （§4.1 と同じく SSM Session Manager 経由 + EC2 上の `docker run --rm` で実行する）
   $ aws ssm start-session --target i-xxxxxxxxx
   $ sudo docker run --rm \
       --env-file /etc/expense-saas/app.env \
       expense-saas:portfolio \
       /app/migrate version

2. 1 つ前のバージョンに戻す
   $ sudo docker run --rm \
       --env-file /etc/expense-saas/app.env \
       expense-saas:portfolio \
       /app/migrate down 1

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
| アプリケーションロールバック（前 ECR イメージタグの再 pull + restart） | 1-2 分（ダウンタイム数秒〜十数秒） |
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
- [x] デプロイ手順が ECR pull + `systemctl restart`（EC2 単一インスタンス）の構成と整合している（architecture.md / ADR-0004 §ポートフォリオ対応 / issue #186 UD-6=A）
- [x] ロールバック条件が具体的な数値基準で定義されている
- [x] ロールバック手順にアプリケーション・DB マイグレーションの両方が含まれている
- [x] ロールバック所要時間の目安が記載されている
- [x] リリース後確認チェックリストにヘルスチェック・機能確認・エラー率確認が含まれている
- [x] 環境変数の詳細は env_config.md に委譲し、重複定義していない
- [x] アラート対応は runbook.md に委譲し、参照リンクを記載している
- [x] デプロイ基盤の設計は architecture.md に委譲している

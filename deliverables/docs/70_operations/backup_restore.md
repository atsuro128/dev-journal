# バックアップ・リストア手順

## 1. この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | データ損失時に許容範囲内で復旧できることを保証する手順を定義する |
| 正本情報 | バックアップ対象・取得方式・保持期間・リストア手順・復旧確認手順 |
| 扱わない内容 | DB スキーマ定義の正本（db_schema.md に委譲）、インフラリソースの詳細設計（architecture.md に委譲） |
| 主な参照元 | `../30_arch/adr/0004-infra.md`, `../50_detail_design/db_schema.md` |
| 主な参照先 | `./runbook.md` |

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `30_arch/adr/0004-infra.md` | RDS 構成（db.t3.micro、Single-AZ、自動バックアップ 7 日間） |
| `30_arch/architecture.md` | システム構成、S3 バケット構成 |
| `50_detail_design/db_schema.md` | テーブル定義（バックアップ対象データの理解） |
| `70_operations/release.md` | ロールバック時のリストア連携 |

---

## 2. RPO / RTO 目標

### 2.1 目標値

| 指標 | 定義 | 目標値 | MVP での実現方式 |
|------|------|--------|-----------------|
| RPO（Recovery Point Objective） | データ損失を許容できる最大時間 | 24 時間以内 | RDS 自動バックアップ（日次）+ トランザクションログ（5 分間隔） |
| RTO（Recovery Time Objective） | 障害発生からサービス復旧までの最大時間 | 1 時間以内 | RDS スナップショットからの復元 + ECS サービスの再接続 |

### 2.2 MVP での実現範囲

| 項目 | 対応 |
|------|------|
| RDS 自動バックアップ | 有効。保持期間 7 日間（ADR-0004） |
| RDS ポイントインタイムリカバリ | 利用可能。直近 5 分以内の任意の時点に復元可能 |
| S3 バージョニング | 無効（MVP）。誤削除時の復旧は不可 |
| Multi-AZ フェイルオーバー | 無効（MVP、ADR-0004 でリスク受容済み）。AZ 障害時は手動復旧 |
| クロスリージョンバックアップ | 対象外（MVP） |

---

## 3. バックアップ対象

### 3.1 RDS（PostgreSQL）

経費精算SaaS の全業務データを格納する。バックアップの最優先対象。

| 項目 | 設定値 |
|------|--------|
| DB インスタンス | expense-saas-prod |
| エンジン | PostgreSQL |
| インスタンスクラス | db.t3.micro（MVP） |
| ストレージ | gp3 |
| 自動バックアップ | 有効 |
| バックアップウィンドウ | 18:00-19:00 UTC（JST 03:00-04:00、低トラフィック時間帯） |
| 保持期間 | 7 日間（ADR-0004） |
| トランザクションログ | 5 分間隔で S3 に自動保存（AWS 管理） |

**バックアップ対象テーブル**（db_schema.md 参照）:

| テーブル | 重要度 | 説明 |
|---------|--------|------|
| `tenants` | 高 | テナント情報 |
| `users` | 高 | ユーザー情報（認証データ含む） |
| `tenant_memberships` | 高 | テナント・ユーザー関連（ロール情報） |
| `expense_reports` | 高 | 経費レポート（業務データの中核） |
| `expense_items` | 高 | 経費明細 |
| `attachments` | 中 | 添付ファイルのメタデータ（ファイル実体は S3） |
| `categories` | 中 | カテゴリマスタ |
| `refresh_tokens` | 低 | リフレッシュトークン（損失時はユーザーの再ログインで復旧） |
| `password_reset_tokens` | 低 | パスワードリセットトークン（損失時は再発行で復旧） |

### 3.2 S3（領収書ストレージ）

領収書ファイル（JPEG / PNG / PDF）を格納する。

| 項目 | 設定値 |
|------|--------|
| バケット名 | expense-saas-receipts-prod |
| キー命名規則 | `{tenant_id}/{report_id}/{attachment_id}` |
| バージョニング | 無効（MVP） |
| ライフサイクルルール | なし（MVP ではファイルを無期限保持） |
| クロスリージョンレプリケーション | 無効（MVP） |

**S3 のバックアップ方針（MVP）**:
- S3 自体の耐久性は 99.999999999%（11 9's）であり、データ消失のリスクは極めて低い
- MVP ではバージョニング・クロスリージョンレプリケーションは導入しない
- 誤操作による削除は、DB の `attachments` テーブルと S3 のオブジェクト一覧を照合して特定する

### 3.3 バックアップ対象外

| 対象 | 理由 |
|------|------|
| ECS タスク定義 | Terraform / GitHub Actions で再構築可能 |
| ECR コンテナイメージ | Docker build で再作成可能。ECR のライフサイクルポリシーで世代管理 |
| CloudWatch Logs | ログ保持期間 30 日（monitoring.md SS9.1）。長期保存は将来検討 |
| Terraform state | S3 + DynamoDB で管理（ADR-0004）。別途バックアップ対象 |

---

## 4. リストア手順

### 4.1 RDS ポイントインタイムリカバリ（PITR）

直近 5 分以内の任意の時点にデータを復元する。最も一般的なリストア方法。

**使用場面**: 誤ったデータ変更・削除の復旧、アプリケーションバグによるデータ破損の復旧。

```
[前提]
- 復元先の時刻を特定済みであること
- 復元先の時刻が RDS の保持期間内（7 日間）であること

[手順]

1. 復元先の時刻を決定する
   - 障害発生の直前の時刻を特定する
   - CloudWatch Logs で正常なリクエストの最終時刻を確認する

   fields @timestamp, msg, status
   | filter ispresent(status) and status < 500
   | sort @timestamp desc
   | limit 1

2. ポイントインタイムリカバリで新しい DB インスタンスを作成する
   $ aws rds restore-db-instance-to-point-in-time \
       --source-db-instance-identifier expense-saas-prod \
       --target-db-instance-identifier expense-saas-prod-restored \
       --restore-time "2026-03-25T10:00:00Z" \
       --db-instance-class db.t3.micro \
       --db-subnet-group-name expense-saas-db-subnet \
       --vpc-security-group-ids <security-group-id>

   所要時間: 15-30 分（データ量に依存）

3. 復元されたインスタンスの起動を待機する
   $ aws rds wait db-instance-available \
       --db-instance-identifier expense-saas-prod-restored

4. 復元データの確認（SS5 の確認手順を実行）

5. アプリケーションの接続先を切り替える
   方法 A: 環境変数 DATABASE_URL を復元インスタンスのエンドポイントに変更し、
           ECS サービスを再デプロイする
   方法 B: 旧インスタンスをリネーム → 復元インスタンスを本番名にリネーム
           （ダウンタイムが最小限）

   方法 B の手順:
   a. 旧インスタンスをリネーム
      $ aws rds modify-db-instance \
          --db-instance-identifier expense-saas-prod \
          --new-db-instance-identifier expense-saas-prod-old \
          --apply-immediately

   b. 復元インスタンスを本番名にリネーム
      $ aws rds modify-db-instance \
          --db-instance-identifier expense-saas-prod-restored \
          --new-db-instance-identifier expense-saas-prod \
          --apply-immediately

   c. ECS サービスを再起動して新接続を確立する
      $ aws ecs update-service \
          --cluster expense-saas-prod \
          --service expense-saas-api \
          --force-new-deployment

6. 旧インスタンスの削除（復旧確認後、7 日以上経過してから）
   $ aws rds delete-db-instance \
       --db-instance-identifier expense-saas-prod-old \
       --skip-final-snapshot
```

### 4.2 RDS スナップショットからのリストア

日次バックアップスナップショットからの復元。PITR で対応できない場合に使用する。

**使用場面**: PITR の対象期間外（5 分以上前のトランザクションログが必要な場合）、スナップショット時点への完全なロールバックが必要な場合。

```
[手順]

1. 利用可能なスナップショット一覧を確認する
   $ aws rds describe-db-snapshots \
       --db-instance-identifier expense-saas-prod \
       --query 'DBSnapshots[].{id:DBSnapshotIdentifier,time:SnapshotCreateTime,status:Status}' \
       --output table

2. 適切なスナップショットを選択する
   - 障害発生前の最新スナップショットを選択する

3. スナップショットから復元する
   $ aws rds restore-db-instance-from-db-snapshot \
       --db-instance-identifier expense-saas-prod-restored \
       --db-snapshot-identifier <snapshot-identifier> \
       --db-instance-class db.t3.micro \
       --db-subnet-group-name expense-saas-db-subnet \
       --vpc-security-group-ids <security-group-id>

   所要時間: 15-30 分

4. 以降、SS4.1 の手順 3〜6 と同じ
```

### 4.3 S3 オブジェクトの復旧

MVP ではバージョニングが無効のため、削除されたオブジェクトの直接復旧はできない。

```
[誤削除が発生した場合の対処]

1. DB の attachments テーブルから、削除されたファイルのメタデータを確認する
   SELECT attachment_id, s3_key, file_name, mime_type
   FROM attachments
   WHERE deleted_at IS NOT NULL
     AND tenant_id = '<tenant_id>';

2. ファイルの復旧方法
   a. S3 オブジェクトが存在する場合（論理削除のみの場合）:
      - attachments テーブルの deleted_at を NULL に戻す
      - S3 オブジェクトは削除されていないため、アクセス可能

   b. S3 オブジェクトが物理削除されている場合:
      - MVP ではバージョニングが無効のため、復旧不可
      - ユーザーに再アップロードを依頼する
      - 将来: S3 バージョニングを有効化して対応する
```

---

## 5. リストア確認手順

リストア実行後、以下の手順でデータの整合性とアプリケーションの動作を確認する。

### 5.1 データ整合性確認

```
[1] 主要テーブルのレコード数を確認する
    復元前（障害発生直前）と復元後のレコード数を比較する。

    SELECT 'tenants' as table_name, count(*) as row_count FROM tenants
    UNION ALL
    SELECT 'users', count(*) FROM users
    UNION ALL
    SELECT 'tenant_memberships', count(*) FROM tenant_memberships
    UNION ALL
    SELECT 'expense_reports', count(*) FROM expense_reports WHERE deleted_at IS NULL
    UNION ALL
    SELECT 'expense_items', count(*) FROM expense_items WHERE deleted_at IS NULL
    UNION ALL
    SELECT 'attachments', count(*) FROM attachments WHERE deleted_at IS NULL
    UNION ALL
    SELECT 'categories', count(*) FROM categories;

[2] RLS ポリシーが有効であることを確認する
    - expense_app ロールで SET LOCAL app.current_tenant を実行し、
      他テナントのデータが見えないことを確認する

    SET ROLE expense_app;
    BEGIN;
    SET LOCAL app.current_tenant = '<test_tenant_id>';
    SELECT count(*) FROM expense_reports;  -- テナント内のレコードのみ返ること
    ROLLBACK;
    RESET ROLE;

[3] 外部キー制約の整合性を確認する
    - expense_reports.user_id が users テーブルに存在すること
    - expense_items.report_id が expense_reports テーブルに存在すること

    SELECT r.report_id, r.user_id
    FROM expense_reports r
    LEFT JOIN users u ON r.user_id = u.user_id
    WHERE u.user_id IS NULL AND r.deleted_at IS NULL;
    -- 結果が 0 件であること
```

### 5.2 S3 との整合性確認

```
[1] attachments テーブルの s3_key に対応する S3 オブジェクトが存在することを確認する

    -- DB から s3_key のリストを取得
    SELECT s3_key FROM attachments WHERE deleted_at IS NULL;

    -- S3 でオブジェクトの存在を確認（サンプリング）
    $ aws s3api head-object \
        --bucket expense-saas-receipts-prod \
        --key "<s3_key>"

    確認観点:
    - サンプル（10 件程度）で全て存在すること
    - 存在しないオブジェクトがある場合は、PITR の復元時刻と S3 削除操作の
      タイミングを照合する
```

### 5.3 アプリケーション動作確認

```
[1] ヘルスチェックの確認
    $ curl -s https://<domain>/health
    期待結果: {"status":"ok","checks":{"database":"ok"}}

[2] ログインの確認
    - テスト用アカウントでログインを試行する
    - JWT の発行が正常であること

[3] 業務操作の確認
    - 経費レポート一覧が表示されること
    - 経費レポートの詳細が表示されること（明細・添付含む）
    - 新規レポートの作成が成功すること

[4] エラーログの確認
    CloudWatch Logs Insights:

    fields @timestamp, level, msg, error
    | filter level = "ERROR"
    | sort @timestamp desc
    | limit 20

    確認観点: リストア後に新たな ERROR ログが発生していないこと
```

---

## 6. 手動スナップショットの取得

リリース前やメンテナンス前に、手動でスナップショットを取得する手順。

```
[手順]

1. 手動スナップショットを作成する
   $ aws rds create-db-snapshot \
       --db-instance-identifier expense-saas-prod \
       --db-snapshot-identifier expense-saas-manual-$(date +%Y%m%d-%H%M%S)

2. スナップショットの作成完了を待機する
   $ aws rds wait db-snapshot-available \
       --db-snapshot-identifier expense-saas-manual-<timestamp>

   所要時間: 5-10 分（データ量に依存）

3. スナップショットの確認
   $ aws rds describe-db-snapshots \
       --db-snapshot-identifier expense-saas-manual-<timestamp> \
       --query 'DBSnapshots[0].{status:Status,time:SnapshotCreateTime,size:AllocatedStorage}'
```

**手動スナップショットの保持ポリシー**:
- リリース前スナップショット: リリース後 7 日間保持し、問題がなければ削除する
- メンテナンス前スナップショット: メンテナンス後 7 日間保持し、問題がなければ削除する
- 命名規則: `expense-saas-manual-YYYYMMDD-HHMMSS`

---

## 7. バックアップ世代管理

| バックアップ種別 | 取得方式 | 保持期間 | 世代数 |
|----------------|---------|---------|--------|
| RDS 自動バックアップ | AWS 自動（日次） | 7 日間 | 最大 7 世代 |
| RDS トランザクションログ | AWS 自動（5 分間隔） | 7 日間（自動バックアップ保持期間と連動） | N/A |
| RDS 手動スナップショット | 手動（リリース前・メンテナンス前） | 7 日間（手動削除） | 運用に依存 |
| S3 オブジェクト | N/A（バージョニング無効） | 無期限 | 1 世代（現行のみ） |

---

## 8. 将来の拡張

| 項目 | 現状（MVP） | 将来の対応 |
|------|------------|-----------|
| RDS Multi-AZ | 無効（ADR-0004 でリスク受容） | テナント数増加時に有効化。自動フェイルオーバーで RTO を大幅短縮 |
| S3 バージョニング | 無効 | 有効化して誤削除からの復旧を可能にする |
| S3 クロスリージョンレプリケーション | 対象外 | DR 要件発生時に検討 |
| RDS クロスリージョンリードレプリカ | 対象外 | DR 要件発生時に検討 |
| バックアップ保持期間の延長 | 7 日間 | コンプライアンス要件に応じて 30 日 / 90 日に延長 |
| 自動バックアップ検証 | 未実施 | 定期的にリストアテストを自動実行する仕組みを導入 |

---

## 9. 品質チェック

- [x] バックアップ対象に RDS と S3 の両方が含まれている
- [x] RPO / RTO の目標値が定義され、MVP での実現方式が明記されている
- [x] RDS のバックアップ設定（自動バックアップ 7 日間）が ADR-0004 と整合している
- [x] ポイントインタイムリカバリの手順がコマンドレベルで記載されている
- [x] スナップショットからのリストア手順がコマンドレベルで記載されている
- [x] リストア後のデータ整合性確認手順（レコード数・RLS・外部キー）が定義されている
- [x] リストア後のアプリケーション動作確認手順が定義されている
- [x] S3 の復旧方針（MVP でのバージョニング無効のリスクと対処）が明記されている
- [x] 手動スナップショットの取得手順と保持ポリシーが定義されている
- [x] DB スキーマの正本は db_schema.md に委譲し、重複定義していない

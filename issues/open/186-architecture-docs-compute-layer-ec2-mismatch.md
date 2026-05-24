# 設計書のコンピュート層表記が ECS Fargate のままで実装実態（EC2 t3.micro 単一）と乖離

## 発見日
2026-05-20

## カテゴリ
architecture

## 影響度
中

## 発見経緯
review

## 関連ステップ
Step 11-E / issue #185 T5（設計書整合）/ ADR-0004（インフラ構成・ポートフォリオ対応）

## ブロッカー
なし

## 問題

複数の設計成果物がコンピュート層を「ECS Fargate」「2 タスク（Task1 / Task2）」と表記しているが、実装実態は EC2 t3.micro 単一インスタンスである。当初は下記 3 ファイルの軽微是正想定だったが、調査の結果、乖離は **10 ファイル**（02_scope / architecture / diagrams / security / files / db_schema / monitoring / env_config / release / runbook / backup_restore）に及び、かつコンピュート層の用語だけでなく以下まで波及していることが判明した（詳細は §実装計画 §1.3 対象ファイル / §3 ファイル別タスク分解 参照）。

- シークレット注入方式（ECS タスク定義経由 → EC2 user_data / SSM Parameter Store）
- デプロイ手順（`aws ecs update-service` → `docker build` + `systemctl restart`）
- ロールバック手順（タスク定義リビジョン戻し → `docker tag` + `systemctl restart`）
- 監視メトリクス（Container Insights → EC2 標準メトリクス + CloudWatch Agent）
- アラーム名（`High-ECS-CPU/Memory` → `High-EC2-CPU/Memory`）
- 運用手順（`aws ecs describe-services` → `aws ssm start-session` + `docker ps` / `journalctl`）

| カテゴリ | ファイル | 主な乖離内容 |
|---|---|---|
| X（高レベル言及） | `02_scope.md` | L70「AWS デプロイ（ECS Fargate / RDS / S3）」 |
| X | `30_arch/architecture.md` | §2 ほか「ECS Fargate 上の Go API サーバー」「Task」 |
| X | `30_arch/diagrams.md` | §1 `subgraph ECS["ECS Fargate"]` / `Task1` / `Task2`、§6 ECS ローリングアップデート |
| X | `50_detail_design/security.md` | L706「ECS コンテナとの通信は VPC 内部で HTTP」 |
| X | `50_detail_design/files.md` | L95-115 ECS タスクロール IAM、L492 ECS Scheduled Task |
| X | `50_detail_design/db_schema.md` | L941「ECS タスク（ワンショット）としてデプロイ前に実行」 |
| Y（詳細メトリクス・IAM） | `50_detail_design/monitoring.md` | §5.3 Container Insights、§6.2 `High-ECS-*` アラーム |
| Y | `70_operations/env_config.md` | §3.1 コンピュート行、§5.3 ECS タスク定義シークレット参照 |
| Z（詳細運用手順） | `70_operations/release.md` | §4.2 `aws ecs update-service`、§7 タスク定義リビジョン戻し |
| Z | `70_operations/runbook.md` | §3 `aws ecs describe-services` 等のコマンド集 |
| X | `70_operations/backup_restore.md` | L31「ECS サービスの再接続」、L97 ECS タスク定義 |

一方、実装済み Terraform（`expense-saas/infra/terraform/ec2.tf`）は単一 EC2 t3.micro インスタンスで、ALB ターゲットグループにも EC2 1 台のみがアタッチされている。ADR-0004 のポートフォリオ対応方針でも「EC2 t3.micro」を明記している。

さらに、`env_config.md` L25-27 および `release.md` L13-15 には「本文書は実務想定の構成（ECS Fargate）で記載している…代替手段（systemd 環境変数、EC2 ユーザーデータ等）を **Step 11 で設計する**」という注記があり、**Step 11 で履行すべき宿題が未履行のまま放置**されている。

## 影響

影響度: **低 → 中**（10 ファイル・~265-355 行差分・ユーザー判断 6 項目に拡大）。

- 設計成果物（構成図・環境設定・運用手順）と実装・ADR-0004 の間で二重表記となり、ポートフォリオの設計説明時に混乱を招く。
- issue #185 T5 で CloudFront を構成図に追加した際、同じ成果物群の中でコンピュート層だけが実態と乖離したままとなり整合性を損なっている（issue #185 T5 内部レビュー W-1 で検出）。
- **Step 11-E 実デプロイ時の参照手順（release.md §4.2 / §7、runbook.md §3 / §4）が動かない**。手順書に書かれている `aws ecs update-service` 等が EC2 実態と乖離しているため、実デプロイ時に運用できない。
- **「Step 11 で再設計する」と約束していた未履行宿題**（`env_config.md §5.3` シークレット注入 / `release.md §4.2` デプロイ手順 / `release.md §7` ロールバック手順）の存在。本対応で併せて履行する。

## 背景

ADR-0004 で初期はマネージドコンテナ（ECS Fargate）を想定していたが、ポートフォリオ対応方針として EC2 t3.micro に変更された。この変更が下流の設計成果物群（architecture.md / diagrams.md / env_config.md ほか）のコンピュート層表記に反映されていない（issue #185 以前からの既存乖離）。issue #185 T5 のスコープは CloudFront 追加に限定されていたため、本乖離は T5 では是正せず別 issue として切り出した。

（議論経緯は §方針確定経緯 参照）

## 提案

**案D を採用**: ADR-0004 を「Fargate 採用 → §ポートフォリオ対応で EC2 ピボット」の歴史記録として**不変**に保ち、下流 10 ファイル（architecture / diagrams / monitoring / env_config / runbook / release / backup_restore / files / db_schema / security / 02_scope）を **EC2 t3.micro 単一インスタンスを一次正本として書き直す**。Fargate への言及は ADR-0004 への参照に集約する（reference のみ。手順・スペック表は EC2 のみ）。

具体的なファイル別タスク分解 / 受け入れ基準 / 作業順序は §実装計画 §3 ファイル別タスク分解 / §4 受け入れ基準 / §5 作業順序と並列化判断 を参照。

Step 11-E 実デプロイ前に解決すべき（手順書として動く必要がある）。

---

## 方針確定経緯

- **当初想定**: architecture.md / diagrams.md / env_config.md の 3 ファイルのみを軽微是正（コンピュート層の用語置換）。影響度: 低。
- **中間案（案A: 注記統一）**: ADR-0004 流に揃え「読み替え注記」（"本文書は ECS Fargate 想定で記載。ポートフォリオでは EC2 に読み替え") を下流 10 ファイルに統一する案。
  - **不採用理由**: Fargate は実装しない（する予定もない）ため、動かない手順書を残す価値が低い。ECS タスク定義のサンプル JSON、`aws ecs update-service` コマンド集が残るとポートフォリオ評価時に「実装と乖離」と判定される。
- **確定案（案D: EC2 を一次正本）**: ADR-0004 を「Fargate 採用 → §ポートフォリオ対応で EC2 ピボット」の歴史記録として**不変**に保ち、下流 10 ファイルは EC2 を一次正本として書き直す。Fargate 言及は ADR-0004 への参照に集約。
- **案D 確定理由（4 点）**:
  - ① ADR-0004 が「Fargate → EC2」の物語を担保するため、下流ドキュメントに二重持ち不要
  - ② 「動く手順書」がポートフォリオ評価に直結する（Step 11-E 実デプロイで参照される）
  - ③ `env_config.md` L25-27 / `release.md` L13-15 の未履行宿題（"Step 11 で再設計する" 約束）を本対応で履行できる
  - ④ codex / 第三者レビューの突っ込みポイントが減る（「動かない手順書」「未履行宿題」の指摘を事前回避）

---

## ユーザー判断項目

本実装計画の着手前に、以下 6 項目 + 並列化方針の計 7 項目をユーザーが確定する必要がある。各項目の詳細は §実装計画 §2 / §5.2 を参照。

確定後、本セクションの「確定日」「確定結果」欄に書き戻す（後続セッションで追記される）。

### UD-1: 接続方式（§実装計画 §2.1）

- 選択肢:
  - A. SSM Session Manager + SSH 廃止 ★（推奨）
  - B. SSH のまま（実装実態）
- 影響範囲: A 採用時は `expense-saas/infra/terraform/` 修正（`iam.tf` への `AmazonSSMManagedInstanceCore` 付与、`ec2.tf` の `key_name` 廃止、SG 22 番削除）が必要。**別 issue 起票必要**（§実装計画 §6.3.1 仮素案あり）。
- 確定日: 2026-05-24
- 確定結果: **A**（SSM Session Manager）。自宅 Wi-Fi の IP 変動に対し SSH では SG 更新が常時発生する運用負荷があるため、SSM 採用が合理的と判断。連動して issue #187 を起票。

### UD-2: ログ転送方式（§実装計画 §2.2）

- 選択肢:
  - A. CloudWatch Agent（unified agent）
  - B. Docker awslogs ログドライバ ★（推奨）
  - C. Fluent Bit on EC2
- 影響範囲: monitoring.md §2.1 ログフォーマット導入文 / §5.x 概要表の書き換え方針に直結。`user_data.sh.tpl` / systemd unit の修正必要性に影響（B 採用なら systemd unit の `docker run` オプション追加のみ）。
- 確定日: 2026-05-24
- 確定結果: **B**（Docker awslogs ドライバ）。追加エージェント不要・systemd unit の `docker run` オプションのみで完結する最小構成を採用。

### UD-3: メトリクス収集方式（§実装計画 §2.3）

- 選択肢:
  - A. CWAgent + AWS/EC2 併用 ★（推奨）
  - B. AWS/EC2 のみ（メモリ閾値アラート ALERT-W3 削除）
- 影響範囲: monitoring.md §5.3 / §5.4 / §6.2 の書き換え方針に直結。UD-2 と組み合わせて Agent 構成を決定（2.2-B + 2.3-A: Agent はメトリクスのみモード、2.2-A + 2.3-A: Agent でログ・メトリクス両方）。
- 確定日: 2026-05-24
- 確定結果: **A**（CWAgent + AWS/EC2 併用）。UD-2=B との組み合わせで「CWAgent はメトリクスのみモード」運用。メモリ・ディスク使用率を取得し ALERT-W3（メモリ 80% 超）を維持、OOM Killer の予兆検知を確保。

### UD-4: 新規アラート `EC2-StatusCheckFailed` 採否（§実装計画 §2.4）

- 選択肢:
  - A. 追加する ★（推奨）— ALB ヘルスチェックと冗長だが SG / NACL 障害を別系統で検知できる（Critical）
  - B. 追加しない — ALB UnHealthyHostCount のみで十分とする
- 影響範囲: monitoring.md §6.2 アラーム定義 / runbook.md §3 対応手順に追記。
- 確定日: 2026-05-24
- 確定結果: **A**（追加する）。単一 EC2 構成のためインスタンスダウン = 即サービス停止であり、1 分間隔の早期検知価値が高い。SG / NACL 障害との切り分けにも有用。

### UD-5: シークレット注入方式（§実装計画 §2.5）

- 選択肢:
  - A. 現状方式（Terraform variable + user_data 直接埋め込み）
  - B. SSM Parameter Store（SecureString）+ systemd EnvironmentFile ★（推奨）
  - C. AWS Secrets Manager + cron で定期 fetch
- 影響範囲: env_config.md §5.3 / §5.4 / §6.2 DB パスワードローテーション手順の書き換え方針に直結。B 採用時は `expense-saas/infra/terraform/` 修正（`iam.tf` に `ssm:GetParameters` / `kms:Decrypt` 付与、`user_data.sh.tpl` の書き換え、`variables.tf` の SSM パラメータ名変数化）が必要。**別 issue 起票必要**（§実装計画 §6.3.1 仮素案あり）。
- 確定日: 2026-05-24
- 確定結果: **B**（SSM Parameter Store）。KMS 暗号化保存・無料・ローテーション容易。UD-1 と統合して issue #187 で対応。

### UD-6: デプロイ方式（§実装計画 §2.6）

- 選択肢:
  - A. ECR pull（`docker pull` + `systemctl restart`、所要 1-2 分）
  - B. EC2 上 docker build（`git pull` → `docker build` → `systemctl restart`、所要 5-10 分）
  - C. 両論併記（一次稿に両方記載、Step 11-E 時に確定）
- 影響範囲: release.md §4.2 デプロイ手順 / §4.3 所要時間 / diagrams.md §6 デプロイフロー図に直結。ローリングアップデート不可（ダウンタイム数秒〜十数秒）はいずれの案でも共通明記。
- 確定日: 2026-05-24
- 確定結果: **A**（ECR pull）。Step 11-E Phase 5 で既にこの方式でデプロイ済みであり実装実態と整合。t3.micro 上の本番ビルドはリソース過食リスクあり。業界標準パターンとしてポートフォリオでもアピール材料。

### UD-7: Phase 2-3 の並列化方針（§実装計画 §5.2 末尾）

- 選択肢:
  - A. 並列実行（Designer-B / -C / -D / -E / -F を別ブランチで並列、想定 3-4 セッション）
  - B. シリアル実行（単一ブランチ `issue-186/ec2-rewrite` で T1 → T11 順次、想定 5-6 セッション、マージコンフリクト回避優先）
- 影響範囲: §実装計画 §5.2 委譲単位とブランチ案。並列化採用時は Phase 1（T1/T4/T6/T11）はシリアル、Phase 2-3 の重 4 タスク（T7/T8/T9/T10）のみ並列に限定する方針（§5.2 I-01 反映）。
- 確定日: 2026-05-24
- 確定結果: **A**（並列実行）。Phase 1 のみシリアル束、Phase 2-3 の T7/T8/T9/T10 を並列、Phase 4（T3 diagrams）は最後にシリアル。memory `feedback_parallel_agents_by_scope` 準拠。

---

## 実装計画

### 1. スコープ確定と前提

#### 1.1 採用方針: 案D「EC2 を正本、Fargate は ADR-0004 だけに残す」

指揮役とユーザーの議論（2026-05-22）で、当初想定の「3 ファイル軽微修正」を超える広範な乖離が判明し、最終的に以下が確定した。

- **案A（注記統一）は不採用**。理由: Fargate は実装しないので「動かない手順書」を残しても価値がない。むしろ ECS タスク定義のサンプル JSON、`aws ecs update-service` コマンド集が残るとポートフォリオ評価時に「実装と乖離」と判定される。
- **案D を採用**。
  - **ADR-0004 は不変**（決定時点記録）。「初期想定は ECS Fargate → §ポートフォリオ対応で EC2 t3.micro にピボット」という歴史を ADR が担保する。
  - **下流設計成果物（architecture.md / diagrams.md / 70_operations/* / 50_detail_design 各種）は EC2 を一次正本として書き直す**。
  - Fargate への言及は ADR-0004 への参照に集約する（reference のみ。手順・スペック表は EC2 のみ）。

#### 1.2 既存「ポートフォリオ対応」注記の扱い

`env_config.md` L25-27 および `release.md` L13-15 に存在する以下の注記は**削除し、本文を EC2 で書き直す**。

> 本文書は実務想定の構成（ECS Fargate）で記載している。ポートフォリオでの実装は ADR-0004 §ポートフォリオ対応を参照し、コンピュートを EC2 t3.micro に読み替えること。…代替手段（systemd 環境変数、EC2 ユーザーデータ等）を **Step 11 で設計する**。

この注記は本来「Step 11 で履行する宿題」を約束していたが未履行のまま放置されていた。本計画はこの宿題（`env_config.md §5.3` シークレット注入 / `release.md §4.2` デプロイ手順 / `release.md §7` ロールバック手順）を併せて履行する。

注記そのものを残すと「実務想定で書いた」というメタな逃げ口上になり、案D の意図（EC2 が正本）と矛盾する。

#### 1.3 対象外（明示）

| 対象外ファイル | 理由 |
|---|---|
| `30_arch/adr/0001-tech-stack.md` | 決定時点記録 |
| `30_arch/adr/0004-infra.md` | 決定時点記録。§ポートフォリオ対応で EC2 を既に明示済み |
| `30_arch/adr/0005-monitoring-logging.md` | 決定時点記録 |
| `pre_step/project_summary.md` | 過去スナップショット |

---

### 2. EC2 equivalent の技術方針（ユーザー判断項目）

実装実態 `expense-saas/infra/terraform/ec2.tf` および `user_data.sh.tpl` を参照した。以下、ユーザー判断が必要な選択肢を列挙する。**指揮役の推奨案を ★ で示すが、最終確定はユーザーが行う**。

#### 2.1 SSH / 接続方式

実装実態: `ec2.tf` に `key_name = var.key_pair_name` あり（SSH キーペア前提）。SSM Session Manager の有効化は user_data 内で確認できず。

| 案 | メリット | デメリット |
|---|---|---|
| ★ **A. SSM Session Manager + SSH 廃止** | キー管理不要、IAM で監査可能、SG で 22 番開放不要 | SSM Agent 導入と IAM ロールにポリシー追加（`AmazonSSMManagedInstanceCore`）が必要。Terraform 修正範囲が広がる |
| B. SSH のまま（実装実態） | 追加実装ゼロ。`expense-saas/infra/terraform/ec2.tf` と整合 | キーペア配布・運用負担、22 番 SG 開放、監査ログがない |

**推奨理由**: ポートフォリオの "AWS 運用のベストプラクティス" 観点では SSM が圧倒的に評価される。ただし実装実態が SSH のため、A 採用なら `expense-saas/` 側の Terraform 修正 issue を別途切る必要がある（本 issue のスコープ外）。

**判断保留時の暫定方針**: 一次稿は「SSM 推奨だが MVP では SSH も可。SG で運用者 IP に限定する」と両論併記し、Terraform 修正と同時に確定する。

#### 2.2 ログ転送方式

実装実態: `user_data.sh.tpl` には CloudWatch Agent / awslogs ドライバいずれの記述もない。コンテナは `docker run` 直接起動（systemd unit `expense-saas.service`、L67-85）。

| 案 | メリット | デメリット |
|---|---|---|
| A. CloudWatch Agent（unified agent） | EC2 メトリクス（メモリ・ディスク）も同時取得可。`/var/log/*` も拾える | エージェント常駐とフルパッケージ追加、IAM ロールに `CloudWatchAgentServerPolicy` 必要 |
| ★ **B. Docker awslogs ログドライバ** | systemd unit に `--log-driver=awslogs --log-opt awslogs-group=... awslogs-region=...` を追加するだけ。設定が単純 | コンテナ stdout のみ。EC2 ホスト側ログ（`/var/log/messages` 等）は別途必要 |
| C. Fluent Bit on EC2 | 高機能・将来拡張容易 | MVP には過剰 |

**推奨理由**: アプリログ（slog JSON 構造化ログ）の転送が主目的。host 側 OS ログまでは MVP では不要。実装実態の `docker run` 起動と最も親和性が高いのは B。

#### 2.3 メトリクス収集方式（Container Insights 代替）

`monitoring.md §5.3 ECS Fargate（Container Insights）` の以下メトリクスを EC2 代替に差し替える必要がある。

| Fargate 元メトリクス | EC2 代替候補 |
|---|---|
| `CpuUtilized` / `CpuReserved`（Container Insights） | `CPUUtilization`（AWS/EC2 標準） |
| `MemoryUtilized` / `MemoryReserved`（Container Insights） | `mem_used_percent`（CloudWatch Agent / 名前空間 `CWAgent`） |
| `RunningTaskCount` / `DesiredTaskCount` | 不要（単一 EC2、固定 1 台）。または `StatusCheckFailed`（AWS/EC2） |

| 案 | 名前空間 | 内容 |
|---|---|---|
| ★ **A. CWAgent + AWS/EC2 併用** | `CWAgent`（メモリ）, `AWS/EC2`（CPU / StatusCheck） | 推奨。CloudWatch Agent をインストールしメモリのみカスタムメトリクス、CPU は標準で取得 |
| B. AWS/EC2 のみ | `AWS/EC2` | メモリ取得を諦める（メモリ閾値アラート ALERT-W3 を削除） |

**推奨理由**: monitoring.md `ALERT-W3 High-ECS-Memory` を維持するならメモリ取得は必須。CloudWatch Agent を 2.2 で見送る場合でも、メトリクス収集だけ別途有効化する選択肢はある（agent インストールのみ）。

**ログ転送（2.2）とメトリクス（2.3）の組み合わせ**:
- 2.2-B + 2.3-A: Agent は「メトリクスのみ」モードで設定（ログ機能は off）。シンプル。
- 2.2-A + 2.3-A: Agent でログ・メトリクス両方。設定が複雑だが統合される。

#### 2.4 アラート名・閾値の差し替えマップ

`monitoring.md §6.2` および `runbook.md §3.x` で参照される全アラーム名を列挙する。

| 旧（Fargate） | 新（EC2） | メトリクス変更 |
|---|---|---|
| `High-ECS-CPU` | `High-EC2-CPU` | `CpuUtilized/CpuReserved*100` → `AWS/EC2 CPUUtilization` |
| `High-ECS-Memory` | `High-EC2-Memory` | `MemoryUtilized/MemoryReserved*100` → `CWAgent mem_used_percent` |
| `HealthCheck-Failure` | 名前変更なし | `ALB UnHealthyHostCount` のまま（変更不要） |
| `High-5xx-Rate-Critical/Warning`, `High-p95-Latency` | 名前変更なし | `ExpenseSaaS/API` 名前空間（変更不要） |
| `High-RDS-*`, `Low-RDS-Storage`, `High-Login-Failures` | 名前変更なし | 変更不要 |

新規追加候補（オプション）:

| アラーム名 | メトリクス | 閾値 |
|---|---|---|
| `EC2-StatusCheckFailed` | `AWS/EC2 StatusCheckFailed` | > 0（連続 2 回） |

**推奨**: `StatusCheckFailed` はヘルスチェック（ALB）と冗長だが、ALB バックエンド到達不能（SG / NACL 障害）を別系統で検知できるため追加することを推奨（Critical）。

#### 2.5 シークレット注入方式

実装実態: `user_data.sh.tpl` L33-60 を読む限り、**現状は Terraform variable で user_data に PEM/環境変数を直接埋め込み**、`/etc/expense-saas/keys/private.pem` および `/etc/expense-saas/app.env` に書き出している。`DATABASE_URL` は `CHANGEME_AFTER_MIGRATE` プレースホルダで、§5.7.3 完了後に手動更新する想定（L44-47 コメント）。Secrets Manager / SSM Parameter Store は使用していない。

| 案 | 内容 | メリット | デメリット |
|---|---|---|---|
| A. 現状方式（Terraform variable + user_data） | terraform.tfvars に PEM/パスワード平文、user_data で `/etc/expense-saas/` に展開 | 追加 AWS リソース不要、実装実態通り | tfvars 平文（gitignore 必須）、ローテーション時に terraform apply 必要、Secrets Manager のような自動ローテーション不可 |
| ★ **B. SSM Parameter Store（SecureString）+ systemd EnvironmentFile** | パラメータを SSM に保存、user_data で `aws ssm get-parameters --with-decryption` → `/etc/expense-saas/app.env` 生成 | IAM で監査・アクセス制御、ローテーションが Terraform apply 不要、無料 | EC2 ロールに `ssm:GetParameters` ポリシー追加が必要、user_data の複雑化 |
| C. AWS Secrets Manager + cron で定期 fetch | Secrets Manager の自動ローテーションを享受 | 90 日自動ローテ可能 | Secrets Manager 有料、MVP には過剰 |

**推奨理由**: ポートフォリオ評価上は「平文 tfvars」は減点要因。SSM Parameter Store の `SecureString` は無料枠で使え、IAM 統合される。ただし「現状の実装実態と乖離」する選択肢のため、**ユーザーが env_config.md でどちらを正本として書くか確定する必要**がある。

**判断保留時の暫定方針**: 一次稿は「実装実態（案A）として記載しつつ、Secrets Manager + 二重展開はポートフォリオ拡張として将来対応とする」と書く。case の選択は本 issue 着手前にユーザー判断を仰ぐ。

#### 2.6 デプロイ手順（release.md §4.2 書き直し）

実装実態: `ec2.tf` L40-49 の user_data には「`image_tag` が空なら EC2 上で `git clone` + `docker build`（案B）」「`image_tag` 指定時は `docker pull`（案A/C）」の分岐がある。systemd unit `expense-saas.service` は L67-85 で定義済み、`docker run --env-file /etc/expense-saas/app.env -v /etc/expense-saas/keys:/app/keys:ro -p 8080:8080 expense-saas:portfolio` を起動。

**EC2 デプロイ手順（案 § 11 Q1 案B = EC2 上ビルド の場合）**:

```bash
# 1. EC2 に SSM Session Manager で接続（2.1-A 採用時）
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx

# 2. 最新コードを取得
cd /home/ec2-user/expense-saas
git fetch && git checkout master && git pull

# 3. Docker イメージビルド
docker build -t expense-saas:portfolio .

# 4. 旧コンテナ停止 → 新コンテナ起動（systemd 経由）
sudo systemctl restart expense-saas.service

# 5. ヘルスチェック確認
curl -s http://localhost:8080/health | jq
# 期待結果: {"status":"ok","checks":{"database":"ok"}}
```

**EC2 デプロイ手順（案 § 11 Q1 案A = ECR pull の場合、将来用）**:

```bash
# 1. ECR ログイン
aws ecr get-login-password --region ap-northeast-1 \
  | docker login --username AWS --password-stdin <account>.dkr.ecr.ap-northeast-1.amazonaws.com

# 2. 新イメージ pull
docker pull <account>.dkr.ecr.ap-northeast-1.amazonaws.com/expense-saas:<tag>
docker tag <account>.dkr.ecr.ap-northeast-1.amazonaws.com/expense-saas:<tag> expense-saas:portfolio

# 3. 再起動
sudo systemctl restart expense-saas.service
```

**ロールバック手順（release.md §7 書き直し）**:

```bash
# 1. 前回イメージのタグ確認（docker image ls）
docker image ls expense-saas

# 2. 旧タグを :portfolio に付け替え
docker tag expense-saas:<前回のタグ> expense-saas:portfolio

# 3. 再起動
sudo systemctl restart expense-saas.service

# 所要時間目安: 1-2 分（コンテナ起動のみ）
```

**ダウンタイム**: 単一 EC2 × 単一コンテナ構成のため、`systemctl restart` 中（数秒〜十数秒）はサービス停止する。**ローリングアップデートは不可能**（ECS Fargate との大きな差異）。`release.md §4.3 デプロイ所要時間の目安` および §6 のロールバック条件（5 分以上 / 3 回失敗の表現）も EC2 実態に書き直す必要がある。

**ロールバック条件の EC2 実態での具体化（I-02 反映）**:

旧版（Fargate 想定）: 「5 分以上 unhealthy / 3 回失敗」

新版（EC2 実態）: 以下のいずれかを満たした時点でロールバック判断。

| 条件 | 具体的閾値 | 検知方法 |
|---|---|---|
| (a) systemctl restart の連続失敗 | `sudo systemctl restart expense-saas.service` を **3 回連続実行しても active (running) にならない** | `systemctl status expense-saas` の出力で `Active: failed` または `activating` のまま |
| (b) ALB ヘルスチェックの unhealthy 継続 | ALB ターゲットグループで対象 EC2 が **5 分以上連続で unhealthy** | CloudWatch `AWS/ApplicationELB UnHealthyHostCount >= 1` が 5 分継続、または `aws elbv2 describe-target-health` で `State=unhealthy` |
| (c) 5xx エラー急増 | ALERT-C2 `High-5xx-Rate-Critical` が発火 | monitoring.md §6.2 の既存アラート（変更不要）|

**release.md §6 への反映方針**: 上記 3 条件を併記し、いずれか 1 つを満たしたらロールバック手順（§7）に進む、と明記する。

---

### 3. ファイル別タスク分解

カテゴリ凡例:
- **X**: 高レベル言及（用語置換 + ADR-0004 参照リンク追加）
- **Y**: 詳細メトリクス・IAM 設定（EC2 equivalent 書き下ろし必要）
- **Z**: 詳細運用手順（EC2 equivalent 書き下ろし必要）

#### T1. `02_scope.md`（カテゴリ X / 小）

- **修正範囲**: L70「AWS デプロイ（ECS Fargate / RDS / S3）」
- **修正タイプ**: 用語置換
- **新表記**: 「AWS デプロイ（EC2 t3.micro / RDS / S3、ADR-0004 §ポートフォリオ対応）」
- **想定差分**: 1 行
- **依存**: なし

#### T2. `30_arch/architecture.md`（カテゴリ X / 中）

- **修正範囲**: L24（ADR-0004 説明）, L39（§2 リクエストフロー本文）, L372（§6.1 SG 説明）, L412（§7 メトリクス）, L441（§8.2 デプロイ枠）, L467（§9.1 マッピング表）
- **修正タイプ**: 用語置換中心、L39 のみ本文書き直し
- **重要な書き換え**:
  - L39: 「ECS Fargate 上の Go API サーバー」→「EC2 t3.micro 上の Docker コンテナ（Go API サーバー）」、`Browser → CloudFront → ALB → Task` → `Browser → CloudFront → ALB → EC2`
  - L372: 「ECS / RDS のアクセス制御」→「EC2 / RDS のアクセス制御」
  - L412: 「ECS/RDS 自動収集」→「EC2/RDS 自動収集（CloudWatch Agent + 標準メトリクス）」
  - L441: 「ECS サービス更新」→「EC2 上で `systemctl restart expense-saas`（手動デプロイ。release.md §4.2 参照）」
  - L467: 「ADR-0004（Fargate スペック）」→「ADR-0004（EC2 t3.micro スペック）」
- **想定差分**: 8-10 行
- **依存**: なし

#### T3. `30_arch/diagrams.md`（カテゴリ X / 中）

- **修正範囲**: §1 システム構成図（L17-60）、§6 デプロイフロー（L204-222、L219 ECS）
- **修正タイプ**: 図書き直し
- **§1 修正内容**:
  - `subgraph ECS["ECS Fargate"]` → `subgraph EC2["EC2 t3.micro (Docker)"]`
  - `Task1` / `Task2` → 単一 `App["Go API サーバー<br/>(Docker container)"]`
  - 2 タスクへの分岐エッジを 1 本に集約
  - `ECS --> CWMetrics` → `EC2 --> CWMetrics`、CloudWatch Agent 経由を注記
- **§6 修正内容**:
  - `G["Docker Build<br/>& Push to ECR"]` → 案 §11 Q1 案B 採用なら「`git push → EC2 上で docker build`」に書き換え、案A 採用なら「Docker Build & Push to ECR」のまま残し H を `EC2: docker pull + systemctl restart`
  - `H["ECS<br/>ローリングアップデート"]` → `H["EC2 上で<br/>systemctl restart"]`
- **想定差分**: §1 で 15 行、§6 で 5 行
- **依存**: 2.1 / 2.5 / 2.6 の判断確定後（本文と図の整合のため）

#### T4. `50_detail_design/security.md`（カテゴリ X / 極小）

- **修正範囲**: L706「TLS 終端 | ALB で終端。ECS コンテナとの通信は VPC 内部で HTTP」
- **修正タイプ**: 用語置換
- **新表記**: 「TLS 終端 | ALB で終端。EC2 上の Docker コンテナとの通信は VPC 内部で HTTP」
- **想定差分**: 1 行
- **依存**: なし

#### T5. `50_detail_design/files.md`（カテゴリ X / 小〜中）

- **修正範囲**: L95-115（ECS タスクロールの IAM ポリシー）、L492（バッチ削除 ECS Scheduled Task）
- **修正タイプ**: 用語置換 + 周辺文の調整
- **L95-115 修正**:
  - 見出し「ECS タスクロールの IAM ポリシー」→「EC2 インスタンスロールの IAM ポリシー」
  - L115「S3 操作は全て ECS タスクロール経由」→「S3 操作は全て EC2 インスタンスロール経由（実装は `expense-saas/infra/terraform/iam.tf` の `ec2_role`）」
  - JSON 自体（Statement / Action）は変更不要（ロールの種別が変わるだけで内容は同じ）
- **L492 修正**: 「日次（cron or ECS Scheduled Task）」→「日次（EC2 上の cron）」※MVP では Phase 3 検討で実装範囲外だが用語整合のため修正
- **想定差分**: 4-5 行
- **依存**: なし

#### T6. `50_detail_design/db_schema.md`（カテゴリ X / 極小）

- **修正範囲**: L941「本番 | ECS タスク（ワンショット）としてデプロイ前に実行」（マイグレーション実行方式の表）
- **修正タイプ**: 用語置換
- **新表記**: 「本番 | EC2 上で `migrate` バイナリを直接実行（リリース手順 release.md §4.1 参照）」
- **想定差分**: 1 行
- **依存**: T9 (release.md) と整合

#### T7. `50_detail_design/monitoring.md`（カテゴリ Y / 大）

- **修正範囲**:
  - L23（参照表 ADR-0004 ラベル）: 「ECS Fargate / RDS / CloudWatch」→ **中立表現**「インフラ選定（コンピュート / DB / 監視基盤）」に統一。さらに本文側 §1 概要等に「コンピュート: EC2 t3.micro（ADR-0004 §ポートフォリオ対応）」と参照誘導を 1 行追加（W-01 反映）<br/>　**理由**: ADR-0004 本文は「決定時点記録」として Fargate のままなので、参照ラベルを「EC2 t3.micro / RDS / CloudWatch」に書き換えるとメタ矛盾（参照先と参照ラベルが乖離）が生じる。中立ラベル + 本文での具体化で両立する
  - L33（出力先サマリー）: 「stdout → CloudWatch Logs（ECS 自動転送）」→ 2.2 で確定した方式（推奨 B なら「stdout → Docker awslogs ドライバ → CloudWatch Logs」）
  - L45（§2.1 ログフォーマット導入）: 「ECS Fargate の awslogs ログドライバー」→「Docker awslogs ログドライバ」（実態 docker run なので Fargate 限定の表現を外す）
  - **L330（W-03 反映）**: §5.x 概要表「ECS Container Insights / RDS 標準メトリクス」→「EC2 標準メトリクス + CloudWatch Agent / RDS 標準メトリクス」
  - **L352（W-03 反映）**: §5.3 導入文「ECS Container Insights および RDS 標準メトリクスにより…」→「EC2 標準メトリクス（`AWS/EC2`）、CloudWatch Agent によるカスタムメトリクス（`CWAgent`）、および RDS 標準メトリクスにより…」
  - **§5.3 自動収集メトリクス（インフラ）**: 「ECS Fargate（Container Insights）」表（L354-361）を**丸ごと EC2 セクションに書き換え**。2.3 推奨案 A 採用時:
    ```
    #### EC2 t3.micro（標準メトリクス + CloudWatch Agent）
    | メトリクス | 説明 | 名前空間 |
    | CPUUtilization | EC2 インスタンス CPU 使用率 | AWS/EC2 |
    | StatusCheckFailed | ステータスチェック失敗 | AWS/EC2 |
    | NetworkIn / NetworkOut | ネットワーク I/O | AWS/EC2 |
    | mem_used_percent | メモリ使用率（要 CloudWatch Agent） | CWAgent |
    | disk_used_percent | ディスク使用率（要 CloudWatch Agent） | CWAgent |
    ```
  - **§5.4 MVP で計測すべき指標一覧**: L391-392「ECS タスク CPU/メモリ使用率」→「EC2 CPU 使用率（`AWS/EC2 CPUUtilization`）」「EC2 メモリ使用率（`CWAgent mem_used_percent`）」
  - **§6.2 CloudWatch Alarms 定義**: `High-ECS-CPU` / `High-ECS-Memory` を `High-EC2-CPU` / `High-EC2-Memory` にリネーム + メトリクス差し替え。`EC2-StatusCheckFailed` を新規追加（2.4 推奨）
- **修正タイプ**: 表書き直し（§5.3, §5.4, §6.2）+ 用語置換
- **想定差分**: 30-40 行
- **依存**: 2.2 / 2.3 / 2.4 確定後

#### T8. `70_operations/env_config.md`（カテゴリ Y / 大）

- **修正範囲**:
  - **§ポートフォリオ対応（L25-27）削除**
  - **§2 環境一覧（L31-35）**: `stg` / `prod` 列「ECS Fargate + RDS」→「EC2 t3.micro + RDS」
  - **§3.1 インフラ構成差分（L43-53）**: コンピュート行「ECS Fargate（0.25 vCPU / 0.5 GB x 1 タスク）」→「EC2 t3.micro（2 vCPU / 1 GiB、単一インスタンス）」※stg/prod 同値で表記
  - **§3.2 アプリケーション設定差分（L57-67）**: 「Container Insights | 無効 / 有効 / 有効」→「CloudWatch Agent | 無効 / 有効 / 有効」、「ALB ヘルスチェック」行は据え置き
  - **§4.4 S3 / オブジェクトストレージ（L120-128）**: 「ECS タスクロール」→「EC2 インスタンスロール」（4 箇所程度）
  - **§5.1 シークレットの分類（L162-166）**: 「ECS タスクロール（stg/prod）」→「EC2 インスタンスロール（stg/prod）」
  - **§5.3 ECS タスク定義でのシークレット参照（L194-235）を全面書き直し**: 2.5 確定方針に応じて。推奨 B（SSM Parameter Store）の場合:
    ```
    ### 5.3 EC2 起動時のシークレット注入
    EC2 user_data で SSM Parameter Store から SecureString を取得し、
    /etc/expense-saas/app.env と /etc/expense-saas/keys/*.pem に展開する。
    
    # /etc/expense-saas/app.env を生成
    aws ssm get-parameters --with-decryption \
      --names /expense-saas/prod/database_url /expense-saas/prod/app_database_url \
      --region ap-northeast-1 --query 'Parameters[].[Name,Value]' --output text \
      | awk '{ ... }' > /etc/expense-saas/app.env
    
    # systemd unit は EnvironmentFile=/etc/expense-saas/app.env で読み込む
    ```
    案A（現状方式）採用の場合は「Terraform variable + user_data で展開（実装実態は user_data.sh.tpl 参照）」を本文化。
  - **§5.4 アクセス制御（L237-246）**: 「ECS タスクロール」→「EC2 インスタンスロール」（複数行）
  - **§6.2 DB パスワードローテーション手順（L268-300）**: `aws ecs update-service ... --force-new-deployment` → `aws ssm put-parameter` + `sudo systemctl restart expense-saas.service`
  - **§8 品質チェック（L356-368）**: 「ECS タスク定義でのシークレット参照方法が具体的に記載されている」→「EC2 user_data でのシークレット注入方式が具体的に記載されている」、「stg/prod での AWS 認証が ECS タスクロール経由」→「EC2 インスタンスロール経由」
- **修正タイプ**: 大幅書き換え
- **想定差分**: 60-80 行
- **依存**: 2.1 / 2.5 確定後（特に §5.3 はユーザー判断が必須）

#### T9. `70_operations/release.md`（カテゴリ Z / 大）

- **修正範囲**:
  - **§ポートフォリオ対応（L13-15）削除**
  - **§1 参照表（L22 ADR-0004 ラベル）**: 「ECS Fargate / RDS / GitHub Actions」→ **中立表現**「インフラ選定（コンピュート / DB / CI/CD）」に統一。本文 §1 末尾または §3 冒頭に「コンピュート: EC2 t3.micro（ADR-0004 §ポートフォリオ対応）」と参照誘導を 1 行追加（W-01 反映）
  - **§3.3 環境・設定（L73-76）**: 「env_config.md と ECS タスク定義を照合」→「env_config.md と SSM Parameter Store（または terraform.tfvars）を照合」（2.5 で B 採用時）
  - **§3.4 デプロイ前最終確認（L80-84）**: 「ECS タスク定義リビジョン」関連の項目を「EC2 上の現コンテナイメージタグ確認」に置換
  - **§4.1 DB マイグレーション（L94-121）**: 踏み台（bastion）→ EC2 自身に SSM Session Manager で接続して実行する手順に修正（または既存の踏み台想定を活かす場合はその旨明記）
  - **§4.2 アプリケーションデプロイ（L127-176）を全面書き直し**: 2.6 で示した EC2 用手順（`git pull` → `docker build` → `systemctl restart`）に置換。「手動デプロイ（緊急時）」の `aws ecs` コマンド集も全削除し、`docker tag` + `systemctl restart` に書き換え
  - **§4.3 デプロイ所要時間の目安（L178-184）**: 「ECS ローリングアップデート 3-5 分」→「EC2 上で docker build + restart 5-10 分（案 §11 Q1 案B）」または「docker pull + restart 1-2 分（案A）」。**ローリングアップデートが不可能なため「ダウンタイム数秒〜十数秒発生」を必ず明記**
  - **§5.1 即時確認（L194-199）**: 「ECS タスクが正常に起動」→「EC2 上のコンテナが Running 状態」（`docker ps` で確認）、「旧タスクが停止」項目は削除（単一コンテナのため概念がない）
  - **§6 ロールバック条件（L246-256）**: 「新タスクが 3 回以上連続で起動に失敗」→「`systemctl restart` 後 3 回以上連続で `docker ps` に Running が現れない」等、EC2 実態に置換
  - **§7.1 アプリケーションのロールバック（L262-298）を全面書き直し**: 2.6 で示した `docker tag :前回のタグ :portfolio` + `systemctl restart` 手順に置換
  - **§7.3 ロールバック所要時間の目安（L324-330）**: ECS タスク定義の戻し 3-5 分 → docker tag + restart 1-2 分
  - **§9 品質チェック（L357-368）**: 「ECS ローリングアップデートの構成と整合」→「EC2 単一インスタンス × `docker run` 構成と整合（ローリングアップデート不可、ダウンタイムを明記）」
- **修正タイプ**: 大幅書き換え（実質的な再執筆）
- **想定差分**: 100-150 行
- **依存**: 2.1 / 2.5 / 2.6 確定後

#### T10. `70_operations/runbook.md`（カテゴリ Z / 大）

- **修正範囲**:
  - **§1 参照表（L19 ADR-0004 ラベル）**: 「ECS Fargate / RDS / S3」→ **中立表現**「インフラ選定（コンピュート / DB / ストレージ）」に統一。本文 §1 末尾または §2 冒頭に「コンピュート: EC2 t3.micro（ADR-0004 §ポートフォリオ対応）」と参照誘導を 1 行追加（W-01 反映）
  - **§2.1 日次確認 #2（L34）**: 「ECS タスク稼働状態 / Running Task Count = Desired（2 タスク）」→「EC2 上の Docker コンテナ稼働状態 / `docker ps` で `expense-saas` が Running、`systemctl status expense-saas` で active」
  - **§2.2 週次確認 #4（L58）**: 「ECS CPU/メモリ推移（Container Insights）」→「EC2 CPU/メモリ推移（`AWS/EC2 CPUUtilization` + `CWAgent mem_used_percent`）」
  - **§3.1 ALERT-C1 HealthCheck-Failure（L79-123）**: `aws ecs describe-services` コマンドを `aws ec2 describe-instances` + SSM `aws ssm start-session` + `docker ps` / `journalctl -u expense-saas` に置換。「runningCount < desiredCount」概念を「コンテナが exited 状態」に置換
    - **L81（W-02 反映）**: ALERT-C1 概要文「ECS タスクまたは DB に問題がある可能性」→「EC2 上の Docker コンテナまたは DB に問題がある可能性」
    - **L118（W-02 反映）**: 「→ [インフラ障害] ECS タスク定義・コンテナイメージを確認」→「→ [インフラ障害] EC2 上の Docker コンテナ起動状態 / コンテナイメージタグを確認（`docker ps -a` / `docker image ls`）」
  - **§3.1 ALERT-C2 High-5xx-Rate-Critical（L127-178）**: 切り分け末尾「ECS タスク・ネットワーク設定を確認」→「EC2 インスタンス・SG・コンテナ起動ログを確認」
  - **§3.2 ALERT-W3 High-ECS-CPU/Memory → High-EC2-CPU/Memory（L251-281）に節リネーム**:
    - 概要文: 「ECS タスクのリソース使用率」→「EC2 インスタンスのリソース使用率」
    - 手順 [1]: 「Container Insights で使用率の推移を確認」→「CloudWatch メトリクス（`AWS/EC2 CPUUtilization` / `CWAgent mem_used_percent`）の推移を確認」
    - 手順 [4]: 「ECS タスクのスケールアウト `aws ecs update-service --desired-count 3`」→ **削除またはコメントアウト**。EC2 単一インスタンスではスケールアウト不可。代替として「インスタンスタイプの一時アップグレード（t3.small へ変更）」または「コンテナ再起動 `sudo systemctl restart expense-saas`」を提示
  - **§4.2 インフラ障害（L499-523）**: `aws ecs describe-services` / `aws ecs list-tasks` / `aws ecs describe-tasks` の 3 コマンドを `aws ec2 describe-instances` + `aws ssm start-session` 経由の `docker ps` / `docker logs` / `journalctl -u expense-saas` に置換
  - **§6 品質チェック（L572-583）**: アラーム名一覧の `High-ECS-CPU, High-ECS-Memory` を `High-EC2-CPU, High-EC2-Memory` に修正
- **修正タイプ**: 大幅書き換え（コマンド集の全面差し替え）
- **想定差分**: 80-120 行
- **依存**: 2.1（SSM 採用なら `aws ssm start-session` 前提のコマンドが書ける） / 2.3 / 2.4 確定後

#### T11. `70_operations/backup_restore.md`（カテゴリ X / 小）

- **修正範囲**:
  - L31「ECS サービスの再接続」→「EC2 上のアプリケーション再起動（`systemctl restart expense-saas`）」
  - L97「ECS タスク定義 | Terraform / GitHub Actions で再構築可能」→「EC2 user_data / systemd unit | Terraform / `expense-saas/infra/terraform/user_data.sh.tpl` で再構築可能」
- **修正タイプ**: 用語置換
- **想定差分**: 2 行
- **依存**: なし

---

### 4. 受け入れ基準

| # | 基準 | 確認方法 |
|---|---|---|
| 1 | 全 10 ファイル（T1-T11）で「ECS Fargate」「Task1/Task2」「ECS タスクロール」「ECS タスク」「Container Insights」「ECS サービス」「ECS タスク定義」の表記が EC2 等価に置換されているか、文脈上必要な箇所のみ ADR-0004 参照に集約されている | 下記手順で確認。<br/>**手順1**: `grep -rnE "ECS\|Fargate\|Container Insights\|タスク定義\|タスクロール\|ECS サービス\|ECS タスク定義" /root-project/dev-journal/deliverables/docs/ --include="*.md"` を実行（`-r` で再帰、`-E` 内では `\|` ではなく `|` を使う点に注意）<br/>**手順2**: ヒットを 0 にはできないため、残った各行が以下のいずれかに該当することを目視で確認する：<br/>　(a) ADR-0004（`adr/0004-infra.md`）への参照リンク内の記述<br/>　(b) 歴史的経緯の説明（「初期想定は ECS Fargate だったが ADR-0004 §ポートフォリオ対応で EC2 にピボット」等）<br/>　(c) ADR ファイル本体（`adr/0001-tech-stack.md` / `adr/0004-infra.md` / `adr/0005-monitoring-logging.md`、`pre_step/project_summary.md` — §1.3 で対象外）<br/>**注**: シェル内では `|` がパイプ扱いになるため、引数全体を二重引用符で囲むこと。`grep -E` 拡張正規表現では `|` を `\|` でエスケープしない |
| 2 | ADR-0004 への参照リンク（`adr/0004-infra.md`）が切れていない | リンク先パスの存在を確認 |
| 3 | ADR-0007（CloudFront / `adr/0007-cloudfront-https.md`）との整合が取れている（CloudFront → ALB → EC2 という経路表現に統一） | architecture.md L39 / diagrams.md §1 を目視 |
| 4 | 既存「ポートフォリオ対応」注記の引用ブロック（§1.2 で引用した「本文書は実務想定の構成（ECS Fargate）で記載している…代替手段（systemd 環境変数、EC2 ユーザーデータ等）を Step 11 で設計する」のブロックそのもの。env_config.md L25-27 / release.md L13-15）が削除されている。**境界**: (a) 上記の引用ブロック（メタな逃げ口上）は削除対象、(b) 本文中の `[ADR-0004](../30_arch/adr/0004-infra.md)` リンクや「ADR-0004 §ポートフォリオ対応参照」の文言は**維持**（むしろ ADR-0004 への参照集約のため積極的に残す） | `grep -n "本文書は実務想定の構成" env_config.md release.md` で 0 件、かつ `grep -n "ADR-0004" env_config.md release.md` でリンクが維持されていることを確認 |
| 5 | env_config.md §5.3 が EC2 のシークレット注入方式（SSM Parameter Store または user_data 直接展開）で書き直されている | §5.3 に `aws ecs` / `secrets[]` 配列が登場しないこと |
| 6 | release.md §4.2 が EC2 デプロイ手順（`docker build` / `docker pull` + `systemctl restart`）で書き直されている | §4.2 に `aws ecs update-service` が登場しないこと |
| 7 | release.md §7 が EC2 ロールバック手順（`docker tag` + `systemctl restart`）で書き直されている | §7.1 に `aws ecs update-service --task-definition` が登場しないこと |
| 8 | release.md §4.3 にダウンタイム発生（ローリングアップデート不可）が明記されている | §4.3 に「ダウンタイム」キーワードが存在 |
| 9 | monitoring.md §6.2 のアラーム名が `High-EC2-CPU` / `High-EC2-Memory` に変更され、runbook.md §3 と整合 | 両ファイルでアラーム名を `grep` し一致 |
| 10 | diagrams.md §1 が EC2 t3.micro × 1（単一インスタンス）を反映 | `subgraph` が `EC2` 系の名称で 1 ノードに集約されている |
| 11 | runbook.md §3 / §4 の**コマンド集（手順本文・コードブロック内）**が EC2 ベース（`docker ps` / `journalctl` / SSM Session Manager 等）に置換されている | `grep -n "aws ecs" /root-project/dev-journal/deliverables/docs/70_operations/runbook.md` の結果が、コマンド集（コードブロック / 手順番号付き本文）に含まれないこと。「歴史的経緯の説明文」「ADR-0004 への参照」の中での `aws ecs` 言及は許容（受け入れ基準 #1 と整合）。**手順本文内のヒット件数が 0 件であることを目視確認** |

---

### 5. 作業順序と並列化判断

#### 5.1 依存関係グラフ

```
[Phase 0] ユーザー判断（2.1 SSH/SSM, 2.2 ログ転送, 2.3 メトリクス, 2.4 新規アラート EC2-StatusCheckFailed 追加採否（I-05 反映）, 2.5 シークレット, 2.6 デプロイ方式）
      ↓
[Phase 1] 軽微な用語置換（並列可。設計判断に依存しない）
  - T1  02_scope.md
  - T4  security.md
  - T6  db_schema.md  ※T9 と用語整合のみ
  - T11 backup_restore.md
      ↓
[Phase 2] 高レベル設計の書き換え（並列可）
  - T2  architecture.md
  - T5  files.md（IAM 詳細）
      ↓
[Phase 3] 詳細書き直し（並列可、ただし内部参照あり）
  - T7  monitoring.md（§5.3, §5.4, §6.2 アラーム定義）
  - T8  env_config.md（§5.3 シークレット注入）
  - T9  release.md（§4.2 デプロイ手順、§7 ロールバック）
  - T10 runbook.md（§3 アラート対応、§4 コマンド集）
      ↓
[Phase 4] 図表更新（Phase 2-3 で本文確定後）
  - T3  diagrams.md（§1 構成図、§6 デプロイフロー）
      ↓
[Phase 5] 受け入れ基準セルフチェック（指揮役 + reviewer）
```

#### 5.2 designer エージェントへの委譲単位の提案

`feedback_parallel_agents_by_scope` メモリに従い、ファイルが重ならない単位で並列化する。

| 委譲単位 | 含むタスク | ブランチ案 |
|---|---|---|
| Designer-A: 軽微置換 | T1, T4, T6, T11 | `issue-186/light-replacements` |
| Designer-B: アーキ・ファイル | T2, T5 | `issue-186/architecture-files` |
| Designer-C: 監視 | T7 | `issue-186/monitoring` |
| Designer-D: env_config | T8 | `issue-186/env-config` |
| Designer-E: release | T9 | `issue-186/release` |
| Designer-F: runbook | T10 | `issue-186/runbook` |
| Designer-G: 図表（最後） | T3 | `issue-186/diagrams` |

**Phase 1 と Phase 2-3 は並列実行可**。Phase 4 (T3 diagrams) のみ本文確定後に着手。

**I-01 反映 / 指揮役判断**: Phase 1（T1 / T4 / T6 / T11 = 合計差分 ~10 行）は 4 並列にすると分割オーバーヘッド（ブランチ作成・PR レビュー）が成果物書き換え量を上回る。**Phase 1 はシリアル実行を推奨**（同一ブランチで T1 → T4 → T6 → T11 を順次、合計 0.5 セッション程度）。並列化は Phase 2-3 の重いタスク（T7 / T8 / T9 / T10）に絞る。

**注意**: 全て dev-journal/ 配下なのでブランチ分離はワーキングコピー競合回避のため。実際は単一ブランチで順次やる方が（マージコンフリクト回避の観点で）安全な可能性もある。**ユーザー判断**: 並列化したい場合は上記単位、シリアル化したい場合は `issue-186/ec2-rewrite` 単一ブランチで Phase 1 → 2 → 3 → 4 を順次。

---

### 6. リスク・懸念点

#### 6.1 作業量見積もり

| Phase | 想定行数（差分合計） | 想定セッション |
|---|---|---|
| Phase 0（判断） | - | 1 セッション（本計画レビュー + 2.x 確定） |
| Phase 1 | ~10 行 | 0.5 セッション |
| Phase 2 | ~15 行 | 0.5 セッション |
| Phase 3 | ~220-310 行 | **2-3 セッション**（T7 / T8 / T9 / T10 が重い） |
| Phase 4 | ~20 行 | 0.5 セッション |
| Phase 5（レビュー） | - | 1 セッション |
| **合計** | **~265-355 行** | **下記両論を参照** |

**作業量見積 — 並列 / シリアル両論併記（I-04 反映）**:

| 戦略 | 想定セッション数 | 根拠 |
|---|---|---|
| **並列（Phase 2-3 で designer-B〜F を並列実行）** | **3-4 セッション** | Phase 0（判断）= 1、Phase 1（シリアル）= 0.5、Phase 2-3 並列ピーク = 1-1.5（最重 T9 release.md が律速）、Phase 4 + 5 = 0.5-1 |
| **シリアル（単一ブランチで T1 → T11 順次）** | **5-6 セッション** | Phase 0 = 1、Phase 1 = 0.5、Phase 2 = 0.5、Phase 3 = 2-3（重 4 タスク順次）、Phase 4 = 0.5、Phase 5 = 1 |

**1 セッション完結は困難**。Phase 3 を 2-3 セッションに分割するのが現実的。**並列化判断はユーザーが §5.2 末尾の判断項目で確定する**。

#### 6.2 検証方法の制約

- **実環境で動く EC2 手順を書く必要があるが、Step 11-E 実デプロイ前**で実証できない箇所がある。
- 緩和策:
  1. `user_data.sh.tpl` / `iam.tf` / `ec2.tf` を実装根拠として明示参照（既に存在することで「絵に描いた餅」回避）
  2. `docker run --env-file` / `systemctl restart` 等の標準的コマンドは実証なしでも信頼性が高い
  3. SSM Session Manager 採用時のコマンド `aws ssm start-session` は AWS 公式手順
  4. **Step 11-E 着手時に再点検する旨をコミットメッセージに明記**

#### 6.3 他チケット・他 issue への波及

| 影響先 | 影響内容 | 対応 |
|---|---|---|
| Step 11-E チケット §6 | runbook 参照箇所があれば EC2 化と整合確認が必要 | 計画確定後に Step 11-E チケットを grep して確認 |
| issue #185（CloudFront）| 既に解決済みだが diagrams.md §1 で CloudFront ノードを保持する必要 | T3 で CloudFront ノードはそのまま、ECS だけ EC2 に置換 |
| issue #181（バックアップ 1 日緩和） | env_config.md §3.1 注釈および backup_restore.md に登場 | T8, T11 で当該注釈は維持（コンピュート層と独立） |
| issue #184（SPA HEAD） | 本 issue とコンピュート層で独立、波及なし（W-04 反映） | 対応不要。`gh issue view 184` で内容を再確認したログを残す |
| `expense-saas/infra/terraform/` | 2.1（SSM）/ 2.5（SSM Parameter Store）採用時は Terraform 修正が必要 | **本 issue のスコープ外**。別 issue 起票を推奨（下記 §6.3.1 で素案） |

#### 6.3.1 別 issue 起票案（仮素案 / W-05 反映）

§2.1 案A（SSM Session Manager）または §2.5 案B（SSM Parameter Store）がユーザー確定した場合、`expense-saas/infra/terraform/` 側の Terraform 修正を別 issue として起票する。

- **タイトル案**: `terraform/ec2.tf: SSM Session Manager + SSM Parameter Store 対応（issue #186 設計確定に追従）`
- **スコープ案**:
  - `iam.tf`: EC2 インスタンスロールに `AmazonSSMManagedInstanceCore` マネージドポリシーをアタッチ（SSM Session Manager 有効化）
  - `iam.tf`: EC2 インスタンスロールに `ssm:GetParameters` / `kms:Decrypt` カスタムポリシーを追加（SSM Parameter Store SecureString 読み取り）
  - `ec2.tf`: `key_name = var.key_pair_name` 行の廃止検討（SSM 接続に統一する場合）
  - `user_data.sh.tpl`: シークレット展開ロジックを `aws ssm get-parameters --with-decryption` 呼び出しに置換、Terraform variable 直接埋め込みの廃止
  - `sg.tf` / `vpc.tf`: SG 22 番（SSH）ルールの削除（SSM 統一する場合）
  - `variables.tf`: SSM Parameter Store のパラメータ名（`/expense-saas/{env}/database_url` 等）の入力変数化
- **着手条件**: 本 issue #186 で §2.1 案A（SSM）/ §2.5 案B（SSM Parameter Store）がユーザー確定したら起票。逆に案B（SSH 維持）/ 案A（tfvars 直接埋め込み維持）採用時は本 issue 起票不要
- **依存**: 本 issue #186 の §実装計画レビュー通過後、本計画 §2 の判断確定とセットで起票
| Step 11 Q1 案B（EC2 上 docker build） | release.md §4.2 のビルド手順の根拠 | Step 11 議論ログ（progress.md 等）と整合を取る |

#### 6.4 案D 特有のリスク

- **「ECS Fargate を選定したという ADR-0004 の歴史」と「下流が EC2 で書かれている」のギャップがレビュアーに説明しづらい可能性**。
  - 緩和策: architecture.md L24 で `adr/0004-infra.md` に「Fargate → §ポートフォリオ対応で EC2 ピボット」の歴史記録、と一行だけ書く
  - architecture.md / monitoring.md / release.md の本文冒頭で「インフラ構成: EC2 t3.micro 単一インスタンス（ADR-0004 §ポートフォリオ対応で確定）」と明示する

- **将来 Fargate に戻す場合の追跡性低下**。
  - 緩和策: ADR-0004 が物語を保持するため、Fargate ピボット時は新規 ADR を起票するルートが確保される

---

### 7. issue 本文の修正提案

当初の §問題 / §影響 / §提案 は「3 ファイル軽微修正」スコープで書かれており、案D 確定（10 ファイル EC2 化）に追従していない。以下を提案する。

#### 7.1 §問題（L21-31）の修正

- 現状の 3 ファイル表に加えて、調査で判明した 7 ファイル（architecture.md / security.md / files.md / db_schema.md / monitoring.md / runbook.md / release.md / backup_restore.md / 02_scope.md）を追記
- 「コンピュート層」だけでなく「シークレット注入方式（ECS タスク定義経由）」「デプロイ手順（`aws ecs update-service`）」「ロールバック手順（タスク定義リビジョン戻し）」「監視メトリクス（Container Insights）」も乖離している旨を追記
- 「Step 11 で再設計する」と約束していた未履行宿題（env_config.md §5.3 / release.md §4.2 / §7）の存在を明記

#### 7.2 §影響（L33-36）の修正

- 影響度を「**低 → 中**」に格上げ提案（10 ファイル・~300 行差分・ユーザー判断 5 項目）
- 影響範囲を以下に拡張:
  - ポートフォリオ説明時の混乱（既存）
  - Step 11-E 実デプロイ時の参照手順が動かない（新規）
  - 「Step 11 で再設計する」の未履行宿題（新規）

#### 7.3 §提案（L42-46）の修正

- 「architecture.md / diagrams.md / env_config.md の表記統一」→「**案D: EC2 を一次正本として下流設計成果物を書き直す（10 ファイル）。Fargate 言及は ADR-0004 への参照に集約する**」
- 「post-MVP 対応で可」→ 削除または「Phase 3」明記。Step 11-E 実デプロイ前に解決すべき（手順書として動く必要がある）

#### 7.4 追加セクションの推奨

- **§方針確定経緯**（新規）: 「案A → 案D へのピボット」の議論ログ要点を 3-5 行で記録（再オープン時の脈絡保持）
- **§ユーザー判断項目**（新規）: 2.1 / 2.2 / 2.3 / 2.5 / 2.6 の選択肢を箇条書きにし、本実装計画着手前にユーザー判断を仰ぐ箇所として明示

#### 7.5 修正の実施タイミング

本実装計画（§実装計画）がレビュー通過した後、issue 本文 §問題 / §影響 / §提案 を「§実装計画と整合」させる形で更新する。**本計画追記と同時に書き換えると差分が読みにくくなるため別コミット推奨**。

---

### v2 改訂履歴

v1（initial architect 起動）に対する reviewer 指摘（CONDITIONAL PASS / blocker 1 + warning 6 + info 5）を反映した。

#### blocker（反映 1 件）

- **B-01 反映**: 受け入れ基準 #1 の grep コマンドを修正。`-r` を追加し再帰検索化、`-E` 内のエスケープを `\|` から `|` に訂正、引数全体を二重引用符で囲む形に統一。さらに「grep 結果を 0 にはできない」性質を明示し、ヒット行を (a) ADR-0004 参照 / (b) 歴史的経緯 / (c) 対象外 ADR ファイル のいずれかに目視分類するワークフローを追記。

#### warning（反映 6 件、すべて反映）

- **W-01 反映**: T7（monitoring）/ T9（release）/ T10（runbook）の §1 参照表 ADR-0004 ラベル行を「中立表現（インフラ選定）」に統一し、本文側で「EC2 t3.micro（ADR-0004 §ポートフォリオ対応）」へ参照誘導する方針を明記。ADR-0004 本文との meta 矛盾を解消。
- **W-02 反映**: T10 runbook.md 修正範囲に **L81**（ALERT-C1 概要文「ECS タスクまたは DB に問題がある可能性」）と **L118**（「→ [インフラ障害] ECS タスク定義・コンテナイメージを確認」）の修正を明示追記。
- **W-03 反映**: T7 monitoring.md 修正範囲に **L330**（§5.x 概要表）と **L352**（§5.3 導入文）の修正を明示追記。
- **W-04 反映**: §6.3 他 issue 波及表に「**issue #184（SPA HEAD）: 本 issue とコンピュート層で独立、波及なし**」の行を追加。`gh issue view 184` で確認ログを残す旨を併記。
- **W-05 反映**: §6.3 末尾に **§6.3.1 別 issue 起票案（仮素案）** を新設。タイトル案 / スコープ案（`iam.tf` への `AmazonSSMManagedInstanceCore`・`ssm:GetParameters` 付与、`ec2.tf` の `key_name` 廃止検討、`user_data.sh.tpl` の `aws ssm get-parameters` 化、SG 22 番ルール削除、`variables.tf` の SSM パラメータ名変数化）/ 着手条件を明示。
- **W-06 反映**: 受け入れ基準 #11 を緩和。「`grep "aws ecs" runbook.md` で 0 件」を「**コマンド集（手順本文・コードブロック内）に `aws ecs` が登場しない**」に修正。歴史的経緯・ADR-0004 参照の中での言及は許容（基準 #1 と整合）。

#### info（反映方針を明記）

- **I-01 反映（採用）**: §5.2 に「Phase 1 はシリアル実行を推奨（指揮役判断）」を追記。Phase 1 の合計差分 ~10 行に対して 4 並列のオーバーヘッドが過大。並列化は Phase 2-3 の重 4 タスクに限定。
- **I-02 反映（採用）**: §2.6 末尾に「ロールバック条件の EC2 実態での具体化」表を追加。(a) `systemctl restart` 3 回連続失敗、(b) ALB ヘルスチェック 5 分以上 unhealthy、(c) ALERT-C2 5xx 急増 のいずれか 1 つを満たしたらロールバックと明確化。
- **I-03 反映（採用）**: 受け入れ基準 #4 で「削除対象 = §1.2 引用ブロック（"本文書は実務想定の構成（ECS Fargate）で記載している…" のメタ逃げ口上）」「維持 = 本文中の `ADR-0004` リンクおよび「ADR-0004 §ポートフォリオ対応参照」文言」と境界を明示。
- **I-04 反映（採用）**: §6.1 作業量見積表を「並列 3-4 セッション」「シリアル 5-6 セッション」の両論併記に変更。並列化判断はユーザーが §5.2 末尾で確定する旨を明記。
- **I-05 反映（採用）**: §5.1 Phase 0 グラフに「2.4 新規アラート `EC2-StatusCheckFailed` 追加採否」を追加。§2.4 で推奨案を提示済みだが、ユーザー判断項目として明示化。

#### 補足修正（reviewer 注記）

- **ADR-0007 ファイル名修正**: 受け入れ基準 #3 で `0007-tls-cloudfront.md` と引用していたが、実ファイル名は `0007-cloudfront-https.md`（`/root-project/dev-journal/deliverables/docs/30_arch/adr/` で `ls` 確認済み）。`adr/0007-cloudfront-https.md` に訂正。

#### v1 で評価された良い点（v2 でも維持）

- 案D 一貫性（EC2 を一次正本、ADR-0004 は不変）
- 実装実態への行番号付き根拠（`ec2.tf` L40-49 / L67-85、`user_data.sh.tpl` L33-60 等）
- ユーザー判断項目の公平性（複数案 + 推奨 ★ スタイル）
- 未履行宿題の明示（`env_config.md §5.3` / `release.md §4.2` / §7）
- 依存関係グラフ（Phase 0-5）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

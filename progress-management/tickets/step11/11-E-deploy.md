# デプロイ・スモークテスト

- 担当: platform-builder
- 依存: 11-D
- ブランチ: `step11/11-E-deploy`
- 出力先: expense-saas/（deploy.yml 等）、AWS リソース
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| 運用設計（リリース） | deliverables/docs/70_operations/release.md | リリース手順 |
| 運用設計（環境設定） | deliverables/docs/70_operations/env_config.md | 環境変数・シークレット |
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | インフラ構成 |
| ADR-0004 | deliverables/docs/30_arch/adr/0004-infra.md | ポートフォリオ対応方針 |
| CI/CD パイプライン | expense-saas/.github/workflows/deploy.yml | デプロイステップ |

## 責務

- AWS リソース構築（ADR-0004 のポートフォリオ対応方針に基づき、EC2 t3.micro / RDS / S3）
- deploy.yml のデプロイステップ有効化（Docker ビルド → ECR プッシュ → デプロイ）
- ヘルスチェック疎通確認
- スモークテスト: 主要フロー1本（申請 → 承認 → 支払）をデプロイ済み環境で手動実行
- 含めない: 本番運用設定（監視・アラート等）、テストコードの追加

## 進め方

1. 11-D のレビューレポートを確認し、ブロッカーが残っていないことを確認する
2. `release.md` / `env_config.md` / `architecture.md` / ADR-0004 を読み、デプロイ対象構成と必要な環境変数を確定する
3. AWS リソースを構築または更新し、DB マイグレーション・S3・アプリ起動に必要な設定を投入する
4. Docker イメージをビルドし、ECR へ push してアプリケーションをデプロイする
5. ヘルスチェック、ログ、フロントエンド表示、API 疎通を確認する
6. デプロイ済み環境で主要フロー1本（申請 → 承認 → 支払完了）を手動実行する
7. 公開 URL、確認日時、確認アカウント、残課題を記録する

## 実施ログ

| 区分 | 対象 | 結果 | メモ |
|------|------|------|------|
| 事前確認 | 11-D ブロッカーなし | 完了（2026-05-13） | Phase 0 ローカル消化 3 条件すべて PASS。詳細は §実施ログ - Phase 0 ローカル消化を参照 |
| インフラ | Terraform IaC 雛形作成 | 完了（2026-05-13、PR #145 / `f3381cd`） | 16 ファイル作成、reviewer + codex 3 ラウンドで全 finding 解消、ADR-0004 整合の Terraform 採用維持。詳細は §実施ログ - Phase 1 IaC 雛形作成を参照 |
| インフラ | AWS リソース構築・更新 | 未実施 | |
| 設定 | 環境変数・シークレット | 未実施 | 値そのものは記録しない |
| デプロイ | Docker build / ECR push / サービス更新 | 未実施 | |
| ヘルスチェック | API / frontend | 未実施 | |
| スモーク | 申請 → 承認 → 支払完了 | 未実施 | |

### 実施ログ - Phase 0 ローカル消化（2026-05-13）

11-D CONDITIONAL 4 条件のうち、ローカルで消化可能な 3 件（#1 Go 統合テスト / #2 npm run e2e / #4 frontend クリーンビルド）を完走。#3 時刻ドリフト smoke は Phase 7（実 AWS 環境必須）に持ち越し。

#### Phase 0-4: frontend クリーンビルド（CONDITIONAL #4）

| 項目 | 内容 |
|------|------|
| 実施環境 | Windows PowerShell（`C:\...\root-project\expense-saas\frontend`） |
| コマンド | `Remove-Item -Recurse -Force .\dist`（Vite の `emptyOutDir: true` でも代替可）→ `npm ci` → `npm run build` |
| 結果 | **PASS** |
| ビルド時間 | 3.86s |
| モジュール数 | 1554 transformed |
| 出力サイズ | `dist/assets/index-B1Q7IuJ_.js` 1,291.29 kB（gzip 391.47 kB） |
| エラー | なし |
| 警告 | chunk size > 500 kB（issue #177 起票、post-MVP）/ npm audit 7 件（issue #176 起票、post-MVP） |
| 旧 root 所有ファイル | クリーンビルドで完全消滅を確認（`index-BmTjhuJz.js` → `index-B1Q7IuJ_.js` にハッシュ更新） |

#### Phase 0-2: Go 統合テスト再実行（CONDITIONAL #1）

| 項目 | 内容 |
|------|------|
| 実施環境 | VS Code タスク `BE: full test (lint + unit + integration)`（Windows ホストから `docker compose --profile test`） |
| 結果 | **PASS** |
| lint | `0 issues.`（golangci-lint v2.11.4） |
| unit test | 全 7 パッケージ ok（cached、前回 11-D セッションから expense-saas/ 差分なしを示す）、3 パッケージ no test files |
| integration test | 全 7 パッケージ ok、handler 88.592s（最長、cross_cutting_test.go + rate_limit_test.go 1363 行含む）、その他 0.004s〜1.384s |
| エラー | なし |
| 結果ファイル | `dev-journal/logs/test-results/{lint,unit,integration}.txt` |

副次起票: Go バージョン定義の環境間不整合（devcontainer 1.24.1 固定 / production `golang:1.24-alpine` パッチ浮動 / test-be image 同梱 Go 暗黙依存）を発見し issue #178 起票（post-MVP、現実害なし）。

#### Phase 0-3: npm run e2e 10件 PASS（CONDITIONAL #2）

| 項目 | 内容 |
|------|------|
| 実施環境 | Windows PowerShell（`C:\...\root-project\expense-saas\`）+ `docker compose` 通常起動（`Docker: フルリセット + seed + ブラウザ(シークレット)` タスク経由） |
| コマンド | `npm run e2e`（= `playwright test --config e2e/playwright.config.ts`） |
| 結果 | **10/10 PASS** |
| 総実行時間 | 26.7s |
| ブラウザ | Chromium 単独（test_strategy.md §10.1 準拠） |

| # | テストケース | 結果 | 時間 |
|---|------------|------|------|
| 1 | CRS-055: フロー1 申請→承認→支払完了 | ✓ | 10.5s |
| 2 | CRS-056: Member ログイン単独確認 | ✓ | 599ms |
| 3 | CRS-060: Approver ログイン単独確認 | ✓ | 584ms |
| 4 | CRS-063: Accounting ログイン単独確認 | ✓ | 578ms |
| 5 | CRS-066: フロー2 却下→再申請 | ✓ | 10.4s |
| 6 | CRS-071: 再申請時の添付配列空確認 | ✓ | 96ms |
| 7 | CRS-073: Admin ダッシュボード | ✓ | 592ms |
| 8 | CRS-074: Admin 全レポート一覧 | ✓ | 897ms |
| 9 | CRS-075: テナント情報ページ | ✓ | 767ms |
| 10 | CRS-075 補足: Member リダイレクト | ✓ | 709ms |

CRS-066（11-C で JWT iat 未来扱い 401 を起こした箇所）が 10.4s で安定 PASS。PR #141/#143 で `domain.JWTVerifier` と `pkg/jwt.Verifier` の両系統に leeway 60s を適用済みである効果を再確認。

#### Phase 0 副次起票一覧

| ID | タイトル | 検出元 |
|----|---------|-------|
| #176 | frontend dev 依存の既知 CVE 棚卸し（esbuild + postcss） | 0-4 の `npm ci` |
| #177 | frontend バンドルの chunk size 500 kB 超過警告 | 0-4 の `npm run build` |
| #178 | Go バージョン定義の環境間不整合 | 0-2 着手時の環境点検 |

すべて post-MVP 扱い。MVP リリース判定ブロッカーにならない。

#### Phase 0 判定

- CONDITIONAL #1（Go 統合テスト）: **PASS**
- CONDITIONAL #2（E2E 10/10）: **PASS**
- CONDITIONAL #4（FE クリーンビルド）: **PASS**
- CONDITIONAL #3（時刻ドリフト smoke）: **Phase 7 に持ち越し**（実 AWS 環境必須）

ローカル消化分は完走、Phase 1（Terraform 雛形作成）への前提条件を満たした。

### 実施ログ - Phase 1 IaC 雛形作成（2026-05-13）

`expense-saas/infra/terraform/` 配下に 16 ファイルの Terraform IaC 雛形を作成。reviewer 内部レビュー + codex 外部レビュー（3 ラウンド）を経て全 finding 解消、PR #145 / merge commit `f3381cd` で master へマージ。

#### 経緯

| ラウンド | 担当 | 結果 | コミット |
|--------|------|------|---------|
| 初回作成 | platform-builder | 16 ファイル作成 | `8249593` |
| 内部 reviewer 初回 | reviewer | blocker 0 / warning 2 (W-01/W-02) / info 6 | - |
| 内部 reviewer FIX | platform-builder | W-01 templatefile 未使用引数削除 / W-02 .terraform.lock.hcl コミット化 / I-02 validation 強化 | `4223821` |
| 内部 reviewer 再レビュー | reviewer | 全 PASS | - |
| codex 初回 | codex | blocker 0 / P1×3 / P2×2（lock の OpenTofu レジストリ不整合、README DB bootstrap 漏れ、RDS max_allocated_storage、SSE bucket_key_enabled、CORS validation 不完全） | - |
| codex FIX 第 1 弾 | platform-builder | OpenTofu 採用に切り替え（agent 環境制約の都合）/ P1-2/P1-3/P2-1/P2-2 解消 | `aebe174` |
| codex 再レビュー | codex | P1-1 未解消（ADR-0004 と乖離） | - |
| codex FIX 第 2 弾 | platform-builder | **Terraform 採用に巻き戻し**（ADR-0004 整合）/ .terraform.lock.hcl 削除 / README を Terraform 前提に戻す | `2450822` |
| codex 3 回目 | codex | **全 5 件解消 PASS** | - |
| マージ | 指揮役 | スカッシュマージ | `f3381cd` |

#### 作成物

`infra/terraform/` 配下 16 ファイル:
- `README.md`（apply 手順、§5.7.3 c 案 DB bootstrap Step 1〜7 詳細）
- `backend.tf` / `providers.tf` / `variables.tf`（14 変数）/ `outputs.tf` / `locals.tf` / `.gitignore`
- `network.tf` / `security_groups.tf` / `ec2.tf` / `rds.tf` / `s3.tf` / `alb.tf` / `iam.tf`
- `user_data.sh.tpl`（EC2 起動スクリプト）
- `terraform.tfvars.example`

`.terraform.lock.hcl` は **コミットしない**設計（Phase 2-2 完了後に USER の Windows ホストで生成・コミット）。

#### Q1〜Q6 反映確認

| Q | 確定 | 反映箇所 |
|---|------|---------|
| Q1 | 案B EC2 上 docker build | `variables.tf` の `image_tag` default = "" / `user_data.sh.tpl` の分岐 |
| Q2 | HTTP のみ ALB DNS 直接 | `alb.tf` の listener 80 のみ |
| Q3 | Single-AZ | `rds.tf` の `multi_az = false` |
| Q4 | AdministratorAccess（IAM ユーザー外部運用）| Terraform 内で扱わず、README §1 に手順 |
| Q5 | 13 ヶ月で terraform destroy | README §9 に手順 |
| Q6 | deploy.yml 別チケット | 影響なし |

#### 副次起票

- **issue #179**: devcontainer egress allowlist の github.com 全許可によるツール install リスク（post-MVP）
- **memory `feedback_no_silent_tool_install.md`**: サブエージェントは事前承認なくツールをインストールしない（governance 補完）

#### 関連 root-project 変更

- root-project commit `695703b`: `.devcontainer/Dockerfile` に Terraform CLI 1.9.8 ビルド時 install 追加（allowlist 拡張なし、最小 egress 維持）
- **devcontainer rebuild が必要**: 次セッション開始時に USER が手動で実施（rebuild しないと Phase 2 以降で `terraform fmt` 等が使えない）

#### Phase 1 判定: **PASS**

reviewer + codex 両レビュー完走、ADR-0004 整合（Terraform 採用）、無料枠制約遵守、全 Q1〜Q6 反映確認。Phase 2 への前提条件を満たす。

## 11-F への引き継ぎ

> ※TLS / 公開 URL は ADR-0007（CloudFront 前段化で HTTPS 化）を正とする。詳細は §11 Q2 詳細【現行構成】参照。

- 公開 URL: CloudFront ドメイン（`https://<cloudfront_domain_name>/`）。エンドユーザー・UAT 担当はこの URL でアクセスする。ALB DNS への直アクセスは B-1-b により 403 となり使えない（`<cloudfront_domain_name>` は `terraform output` 名、apply 後に確定）
- UAT 用アカウントとロール
- デプロイ日時・コミット
- 既知の非ブロッカー（ADR-0007 で受容した逸脱 B-3「CloudFront〜ALB 間 HTTP」/ B-5「viewer 側 TLS 最小バージョン TLSv1 固定」、Secrets Manager 不採用。旧 TLS 案1 の HTTP 運用差分は ADR-0007 への方針変更により失効）
- UAT で重点確認すべき箇所（CloudFront 経由の HTTPS 配信・HSTS・CORS が正しく機能すること。検証詳細は issue #185 T6 に委ねる）

## 完了条件

- デプロイ済み環境で第三者がアクセスできる状態になっている
- ヘルスチェックエンドポイントが応答する
- 主要フロー1本（申請 → 承認 → 支払完了）がデプロイ済み環境で通る

---

## 実施計画（詳細 runbook、§0–§14、reviewer 3 ラウンド + codex 5 ラウンドで確定）

作成日: 2026-05-08
作成者: planner（Claude）
対象チケット: `dev-journal/progress-management/tickets/step11/11-E-deploy.md`
依存: Step 11-D（CONDITIONAL PASS, 2026-05-07）/ PR #144（issue #175）

---

### 0. このドキュメントの位置付け

- 11-E チケットの詳細化版。本書を runbook として参照しながら作業を進められる粒度で記述する。
- 実コード・Terraform ファイルの作成は本フェーズでは行わず、計画策定のみ。
- ADR-0004 §ポートフォリオ対応（EC2 t3.micro / RDS db.t3.micro / S3 / ALB、無料枠内）に厳密に従う。Fargate 案は採用しない。
- サブエージェントは外部通信コマンド（AWS CLI / Terraform apply / docker push 等）を実行できない。これらは「ユーザー実行」とマークする（feedback_subagent_permissions.md）。
- 監視・アラート等の本番運用設定は 11-E スコープ外（11 完了後の post-MVP 対応）。

---

### 1. 全体作業フロー

```
[Phase 0] 事前条件確認
   |  (a) PR #144 (issue #175 deploy.yml 整合) merged
   |  (b) 11-D の CONDITIONAL 4 条件のうち、ローカル可能分を先消化
   |  (c) AWS アカウント・支払い設定・IAM ユーザー(Terraform 用) 準備完了 [USER]
   v
[Phase 1] Terraform IaC 作成 [SUBAGENT: platform-builder]
   |  - infra/terraform/ ディレクトリ構成・main.tf/variables.tf/...
   |  - 静的検証のみ（terraform fmt / terraform validate がローカルで通る状態）
   v
[Phase 2] state バックエンド初期化 [USER]
   |  - S3 (state) バケット + DynamoDB (lock) を AWS CLI で先行作成
   |  - terraform init -backend-config=...
   v
[Phase 3] terraform plan -> apply [USER]
   |  - VPC / SG / EC2 / RDS / S3 / ALB / IAM をプロビジョニング
   |  - apply 後に EC2 public DNS / RDS endpoint / ALB DNS を outputs から取得
   v
[Phase 4] DB マイグレーション投入 [USER 主導 / 指揮役支援]
   |  - 一時的に EC2 から (or ローカルから踏み台経由で) migrate up
   v
[Phase 5] アプリ Docker イメージ配布 [USER]
   |  - ECR を使うか EC2 上 build かは §3.4 の判断による
   |  - 初期方針: GHCR 利用 or EC2 上で git clone + docker build
   v
[Phase 6] EC2 上でアプリ起動 [USER]
   |  - systemd unit or docker run --restart=always
   |  - JWT 鍵ペアを EC2 上に配置（Secrets Manager は無料枠外なので使わない）
   v
[Phase 7] ヘルスチェック・スモークテスト [指揮役 + USER]
   |  - curl https://<cloudfront_domain_name>/health（CloudFront 経由＝正規経路。§6.1）
   |  - curl http://<alb_dns_name>/health で 403 を確認（B-1-b 閉域確認。§6.1）
   |  - 11-D CONDITIONAL の 4 条件残り（時刻ドリフト確認）
   |  - 申請 -> 承認 -> 支払 の手動フロー
   v
[Phase 8] deploy.yml の if: false 解除（任意）[SUBAGENT: platform-builder]
   |  - PR #144 マージ後、E2E ジョブを実環境ではなく docker-compose.e2e.yml で動かす
   |  - スコープ判断: MVP では deploy.yml は参考実装のままでも 11-E 完了条件は満たす
   v
[Phase 9] 11-F への引き継ぎ
   |  - 公開 URL / UAT アカウント / デプロイ日時 / 既知非ブロッカー
```

依存関係の要点:

- Phase 1 と Phase 0(c) は並列可（Terraform 雛形作成は AWS アカウント不要）。
- Phase 2 以降は AWS 認証が必要。サブエージェントは関与不可。
- Phase 7 のスモークは Phase 6 完了が前提。
- Phase 8 は 11-E 完了条件には含めない（deploy.yml は §8.2 で「現時点では手動デプロイ」と明記済み）。

---

### 2. AWS リソース構成と無料枠制約

#### 2.1 ポートフォリオ実装の構成図（テキスト）

```
[Internet]
   |
   v
[ALB] (HTTPS:443, HTTP:80 -> redirect)
   - subnet: public-a, public-c (2AZ 必須)
   - target group: HTTP:8080 -> EC2
   - ACM 証明書: ALB に貼る（必要なら Route53 + ACM、簡略化のため
                 ALB の DNS をそのまま使う案 §11 Q2 で判断）
   |
   v
[EC2 t3.micro]  Amazon Linux 2023 (Docker 入り)
   - subnet: public-a (Single-AZ で受容)
   - SG: ALB SG からの 8080 のみ許可。SSH (22) は my-IP のみ
   - ボリューム: gp3 8GB (無料枠は 30GB まで)
   - IAM Role:
       * S3: GetObject / PutObject / DeleteObject (受領書バケットのみ)
       * (CloudWatch Logs は MVP スコープ外)
   - Docker:
       expense-saas API コンテナ（Go バイナリ + SPA embed）
       host network or bridge で 8080 公開
   |
   v (private)
[RDS PostgreSQL 16, db.t3.micro]
   - subnet group: db-subnet-private-a, db-subnet-private-c (2AZ 必須)
   - SG: EC2 SG からの 5432 のみ許可
   - Single-AZ（ADR-0004: MVP リスク受容）
   - Storage: gp3 20GB（無料枠は 20GB まで）
   - Backup: 7 日間（NFR-AVAIL-003）
   - Public access: No
   |
   ====================== 別系統 ======================
[S3]  expense-saas-receipts-prod (or -portfolio)
   - region: ap-northeast-1
   - パブリックアクセス全面ブロック
   - SSE-S3 (デフォルト暗号化)
   - 署名付き URL でのみアクセス（ATT-010-012）

[S3]  expense-saas-tfstate-portfolio (Terraform state 用)
[DynamoDB] expense-saas-tflock (Terraform lock 用、PAY_PER_REQUEST)
```

#### 2.2 無料枠条件と超過リスク

| リソース | 無料枠条件 | 超過リスク | 緩和策 |
|---|---|---|---|
| EC2 t3.micro | 12ヶ月、750h/月 | 13ヶ月目以降は約 $8/月 | 期限到来前に停止判断。`terraform destroy` を runbook 化 |
| RDS db.t3.micro Single-AZ | 12ヶ月、750h/月、Storage 20GB | Multi-AZ 化や 20GB 超で課金 | Multi-AZ にしない、`allocated_storage = 20` 固定 |
| S3 | 12ヶ月、5GB、20k GET、2k PUT | 領収書蓄積で 5GB 超 | UAT 後に `aws s3 rm --recursive` で掃除 |
| ALB | 12ヶ月、750h/月、15GB データ | 既存に 1 つでもあると同時に同一勘定 | 他用途 ALB を作らない |
| データ転送 | アウトバウンド 100GB/月 | UAT 規模では問題なし | - |
| EBS | 30GB gp3 | EC2 1 台 8GB なら問題なし | - |
| Elastic IP | EC2 にアタッチされていれば無料 | 未アタッチで $3.6/月 | 使わない（ALB 経由でアクセスする） |
| NAT Gateway | 無料枠なし。$32/月+データ課金 | **絶対に作らない** | EC2 を public subnet に配置し、private subnet は RDS のみで NAT 不要にする |
| CloudWatch | 詳細監視は無料枠外 | - | MVP では基本メトリクスのみ |
| Secrets Manager | 30 日無料枠後に $0.4/secret/月 | - | 使わず、JWT 鍵は **Terraform variable（sensitive）+ user_data 経由で `/etc/expense-saas/keys/` に書き出し** に一本化（W-02 対応）。SSM Parameter Store SecureString 経由は post-MVP の選択肢として §15 に記録のみ |
| ECR | 無料枠なし（500MB まで $0.10/月程度） | - | §3.4 で「ECR 使わない」案と比較判断 |
| Route53 Hosted Zone | $0.50/月（無料枠なし） | - | カスタムドメイン不要なら ALB DNS をそのまま使う |
| ACM 証明書 | 無料 | - | ただし ACM は Route53 か外部 DNS 検証が必要。§11 Q2 で判断 |

**最大想定月額**（無料枠超過後 / 13ヶ月目以降）: §7 で算定。

#### 2.3 Terraform state 管理方針

採用案: **S3 + DynamoDB バックエンド**（推奨）

- 理由:
  - 無料枠内（S3 < 1GB、DynamoDB PAY_PER_REQUEST で実質無料）
  - apply 失敗時の lock 残骸を扱える
  - 「ローカル state」案は紛失リスクが高く、ポートフォリオでも避ける
- 制約: state バケットと lock テーブルは Terraform で作れない（chicken-and-egg）。
  → AWS CLI で先行作成する手順を §5.3 に明記。

代替案: ローカル state（不採用、参考まで）

- メリット: セットアップ不要
- デメリット: PC 紛失で復旧不能、複数端末で作業不可

---

### 3. Terraform ファイル構成

#### 3.1 ディレクトリ構成（提案）

```
expense-saas/
└── infra/
    └── terraform/
        ├── README.md                 # 本書 §5 の抜粋（apply 手順）
        ├── backend.tf                # S3 + DynamoDB backend 設定
        ├── providers.tf              # aws provider, version 制約
        ├── variables.tf              # 入力変数
        ├── outputs.tf                # ALB DNS、RDS endpoint 等
        ├── locals.tf                 # 命名プレフィックス等
        ├── network.tf                # VPC, subnets, route tables, IGW
        ├── security_groups.tf        # ALB SG, EC2 SG, RDS SG
        ├── ec2.tf                    # EC2 instance, key_pair (data), user_data
        ├── rds.tf                    # DB subnet group, RDS instance, parameter group
        ├── s3.tf                     # 領収書バケット、ライフサイクル、暗号化
        ├── alb.tf                    # ALB, target group, listener, ACM (option)
        ├── iam.tf                    # EC2 instance profile (S3 access)
        ├── terraform.tfvars.example  # tfvars テンプレート（gitignore せず）
        └── .gitignore                # *.tfstate, .terraform/, terraform.tfvars
```

#### 3.2 各ファイルの責務と最低限定義すべきリソース

| ファイル | 主なリソース / 内容 |
|---|---|
| `backend.tf` | `terraform { backend "s3" { ... } }`（bucket, key, region, dynamodb_table, encrypt=true） |
| `providers.tf` | `aws` プロバイダ v5 系、`required_version >= 1.6` |
| `variables.tf` | `aws_region`, `project_name`, `environment` (= "portfolio"), `key_pair_name`, `db_password`（sensitive）, `allowed_ssh_cidr`（自宅 IP）, `jwt_private_key_pem`（sensitive）, `jwt_public_key_pem`（sensitive）, `cors_allowed_origins` |
| `network.tf` | `aws_vpc`（10.0.0.0/16）, `aws_subnet` ×4（public-a/c, private-a/c）, `aws_internet_gateway`, `aws_route_table` × public のみ。NAT Gateway は作らない |
| `security_groups.tf` | `alb_sg`（80/443 from 0.0.0.0/0）, `ec2_sg`（8080 from alb_sg, 22 from allowed_ssh_cidr）, `rds_sg`（5432 from ec2_sg） |
| `ec2.tf` | `aws_instance`（t3.micro, AL2023 AMI data source, instance_profile, user_data でDocker・コンテナ起動） |
| `rds.tf` | `aws_db_subnet_group`（private-a/c）, `aws_db_instance`（engine=postgres 16, db.t3.micro, allocated_storage=20, backup_retention_period=7, multi_az=false, publicly_accessible=false, storage_encrypted=true） |
| `s3.tf` | `aws_s3_bucket`（領収書）、`aws_s3_bucket_public_access_block`（all true）、`aws_s3_bucket_server_side_encryption_configuration`（AES256）。**`aws_s3_bucket_versioning` は採用しない（上流追従、F-117 対応、§4.2 末尾差分表参照）**。上流 `backup_restore.md` §3.2 / §4.2 で MVP は S3 versioning 無効・誤削除復旧不可と定義されているため、portfolio 構成でもこれに合わせる。lifecycle も versioning 無効に伴い不要 |
| `alb.tf` | `aws_lb`（application, public-a/c）, `aws_lb_target_group`（HTTP:8080, health_check.path=/health）, `aws_lb_target_group_attachment`（EC2）, `aws_lb_listener`（443 + ACM 連携 or 80 のみ §11 Q2 で判断）。**脚注（W-03 対応）**: ALB → EC2 の health check は ALB SG 内部からの HTTP 8080 で行う。アプリ側のレート制限（CRS-077 相当）は `/health` も対象だが、env_config.md §3.2 の health check 30 秒間隔（= 2 req/min）であれば `UNAUTH_RATE_LIMIT_PER_MINUTE` デフォルト 20 の global IP 制限内に余裕で収まるため、誤遮断のリスクはない |
| `iam.tf` | `aws_iam_role`（EC2 用）, `aws_iam_instance_profile`, `aws_iam_role_policy`（受領書バケットへの GetObject/PutObject/DeleteObject、prefix で tenant 制限はしない MVP は IAM ではなくアプリ層で担保） |
| `outputs.tf` | `alb_dns_name`, `ec2_public_ip`, `rds_endpoint`（sensitive）, `s3_bucket_name` |

#### 3.3 変数化すべき値

| 変数 | 例 | 備考 |
|---|---|---|
| `aws_region` | `ap-northeast-1` | 固定でよい |
| `project_name` | `expense-saas` | リソース名プレフィックス |
| `environment` | `portfolio` | env_config.md の prod は使わず portfolio とする |
| `key_pair_name` | `expense-saas-portfolio` | 事前に AWS コンソールで作成 |
| `allowed_ssh_cidr` | `xxx.xxx.xxx.xxx/32` | ユーザーの自宅 IP |
| `db_password` | `(sensitive)` | `terraform.tfvars` 直書き or `TF_VAR_db_password` 環境変数 |
| `jwt_private_key_pem` | `(sensitive)` | `openssl genrsa` で生成済みのものを TF_VAR で渡す |
| `jwt_public_key_pem` | `(sensitive)` | 同上 |
| `cors_allowed_origins` | `https://<alb-dns>` | apply 後に再設定する 2 段階デプロイ or プレースホルダ |
| `image_tag` | `latest` or git SHA、または空文字（default = ""） | **§11 Q1 で「案A GHCR / 案C ECR」を選んだ場合のみ user_data で `docker pull ghcr.io/.../expense-saas:${image_tag}` に展開。「案B EC2 上 build」を選んだ場合は空文字のまま user_data 側で分岐し pull をスキップ**。Q1 確定前に platform-builder へ指示する場合は default = "" の optional 変数として定義する（W-01 対応） |

---

### 4. シークレット・環境変数一覧

env_config.md §4 を全文確認のうえ、本番（= portfolio 環境）で必要なものを列挙。

> **方針（W-02 対応）**: シークレットは「Terraform variable（sensitive）→ user_data → EC2 上 `/etc/expense-saas/{keys,app.env}` に root:600 で配置」の単一経路で管理する。SSM Parameter Store / Secrets Manager は本 MVP では使わない。post-MVP で SSM 経由に移行する案は §15（あれば）に記録。

#### 4.1 AWS 側で管理するもの

##### IAM ユーザー（Terraform 実行用、ユーザーローカル）

- 用途: `terraform apply` 実行時の AWS API 呼び出し
- 推奨ポリシー: 簡略化のため `AdministratorAccess` を一時的に付与（ポートフォリオ用途）。実務想定では VPC/EC2/RDS/S3/ALB/IAM の必要権限のみに絞る
- アクセスキー: `~/.aws/credentials` に `[expense-saas-portfolio]` プロファイルとして登録

##### IAM Role（EC2 instance profile）

- 用途: EC2 上のアプリから S3 にアクセス
- ポリシー: 領収書バケットへの `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket`

##### キーペア

- AWS コンソールで `expense-saas-portfolio.pem` を作成 → ローカル `~/.ssh/` に保存（chmod 400）

#### 4.2 アプリ側環境変数（EC2 上のコンテナに渡す）

env_config.md §4 prod 列を写したもの。Secrets Manager は無料枠外のため、JWT 鍵と DB URL は **EC2 上の `/etc/expense-saas/` 配下のファイルに root:600 で配置**して `docker run -v` でマウントする。

| 変数 | 値 | 取得元 |
|---|---|---|
| `PORT` | `8080` | 固定 |
| `LOG_LEVEL` | `info` | 固定 |
| `DATABASE_URL` | `postgres://expense_owner:<owner_pw>@<rds-endpoint>:5432/expense_saas?sslmode=require` | tfvar の `expense_owner_db_password`（§5.7.2 / §5.7.3）。**§5.7.3 Step 1（000001/000002 をマスターで migrate）→ Step 2（マスターで `ALTER ROLE expense_owner WITH PASSWORD '${OWNER_DB_PW}'` + `ALTER TABLE schema_migrations OWNER TO expense_owner`）→ Step 4（残り migrate を `expense_owner` で）を完了した後の値**。アプリ・seed・000003 以降の migrate がいずれもこのロールで接続するため、`expense_owner` がテーブル owner（RLS bypass）かつ `schema_migrations` の owner として成立していることが前提（F-115 再々修正） |
| `APP_DATABASE_URL` | `postgres://expense_app:<app_pw>@<rds-endpoint>:5432/expense_saas?sslmode=require` | tfvar の `expense_app_db_password`（§5.7.2 / §5.7.3）。**§5.7.3 Step 2 で `ALTER ROLE expense_app WITH PASSWORD '${APP_DB_PW}'`、Step 3 で `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner ... GRANT ... TO expense_app`、Step 5 で `GRANT ... ON ALL TABLES IN SCHEMA public TO expense_app` を完了した後の値**。本番 9 テーブルへの DML 権限は Step 3（default privileges）+ Step 5（既存テーブル一括 GRANT）の二重ガードで担保し、Step 6 で `has_table_privilege()` メタデータ検証 + `users` への live SELECT で実地検証する（F-115 + F-118 + F-119 対応） |
| `JWT_PRIVATE_KEY_PATH` | `/app/keys/private.pem` | コンテナ内パス（host の `/etc/expense-saas/keys/` を bind） |
| `JWT_PUBLIC_KEY_PATH` | `/app/keys/public.pem` | 同上 |
| `S3_BUCKET` | `expense-saas-receipts-portfolio-<8桁ランダムサフィックス>` | **Terraform output 値で動的に決まる**。env_config.md §4.4 prod の固定名 `expense-saas-receipts-prod` とは差分あり（S3 グローバル名前空間衝突回避のためランダムサフィックスを付与する設計、I-04 対応） |
| `AWS_REGION` | `ap-northeast-1` | 固定 |
| `S3_ENDPOINT` | （未設定）| AWS デフォルトを使う |
| `S3_PUBLIC_ENDPOINT` | （未設定）| 同上 |
| `AWS_ACCESS_KEY_ID` | （未設定）| EC2 instance profile を使う |
| `AWS_SECRET_ACCESS_KEY` | （未設定）| 同上 |
| `CORS_ALLOWED_ORIGINS` | `http://<alb-dns>`（HTTPS 化したら `https://...`） | Terraform output |
| `LOGIN_RATE_LIMIT_PER_MINUTE` | （未設定。デフォルト 5） | env_config.md §4.6 の本番方針 |
| `UNAUTH_RATE_LIMIT_PER_MINUTE` | （未設定。デフォルト 20） | 同上 |

##### env_config.md prod との差分（11-F 引き継ぎ必須項目）

本計画の portfolio 環境は env_config.md prod と以下の点で差分がある。11-F 引き継ぎ表に記録する。

| 項目 | env_config.md prod | 本計画 portfolio | 差分理由 |
|---|---|---|---|
| `DATABASE_URL` / `JWT_*` の保管 | Secrets Manager | Terraform variable + EC2 ファイル（`/etc/expense-saas/`） | Secrets Manager 無料枠外（B-01 / W-02） |
| `CORS_ALLOWED_ORIGINS` | `https://...` 固定 | §11 Q2 案1 採用時は `http://<alb-dns>`、案2 採用時は `https://<custom-domain>` | TLS 案選定による（B-02） |
| `S3_BUCKET` | `expense-saas-receipts-prod`（固定） | `expense-saas-receipts-portfolio-<8桁ランダム>` | S3 グローバル名前空間衝突回避（I-04） |
| TLS | ACM 証明書必須 | §11 Q2 案1 採用時は HTTP のみ | ポートフォリオ簡易構成（B-02） |
| S3 バケット versioning | 無効（上流 `backup_restore.md` §3.2 / §4.2、誤削除復旧不可） | **無効（上流追従）** | 上流の MVP 復旧期待値（誤削除復旧不可）を維持。本計画初版では Enabled + noncurrent 30 日 lifecycle を提案していたが、上流 `backup_restore.md` の MVP 前提との整合と destroy 時の noncurrent 残置による二重課金回避を優先して a 案（上流追従）に確定（F-117） |
| `expense_app` / `expense_owner` ロールのパスワード | Secrets Manager から `ALTER ROLE ... WITH PASSWORD` で投入（env_config.md §6.4） | **`migrate up 2`（マスター）→ `ALTER ROLE` + `ALTER TABLE schema_migrations OWNER TO expense_owner`（マスター）→ `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner`（`expense_owner`）→ 残り `migrate up`（`expense_owner`）→ `GRANT ON ALL TABLES`（`expense_owner`、保険）の 5 段階**（§5.7.3） | migration `000002_create_roles.up.sql` が両ロールを `localdev` で作成し、かつ 000001 の `CREATE EXTENSION pgcrypto` / 000002 の `CREATE ROLE` は superuser 権限が必須のため、最初の 2 本のみマスターで実行する。一方で 000003 以降を `expense_owner` で実行しないとテーブル owner がマスターになり、上流 `release.md` §4.1 と `cmd/server/main.go` の expense_owner 接続前提が崩れる。さらに `ALTER DEFAULT PRIVILEGES` は実行ロール限定挙動のため、c 案では `expense_owner` 接続側でも default privileges を仕込み直す必要がある（F-115 + F-118 再々修正） |
| migration 実行主体 | `expense_owner` 単独（release.md §4.1） | **000001/000002 はマスター、000003 以降は `expense_owner`**（§5.7.3） | release.md は CREATE EXTENSION / CREATE ROLE が migrate に同梱されることを想定していないため、本計画で実マイグレーションに即した分割を行う。テーブル owner = `expense_owner` という上流方針は維持（F-115 再修正） |
| `expense_app` への DML 権限付与 | `migrate up` 内（`000002_create_roles.up.sql`）で完結（env_config.md §6.4 / release.md §4.1） | **000002 のマスター実行版は本番テーブルに効かないため、§5.7.3 Step 3（`expense_owner` 接続で `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner ... GRANT ... TO expense_app`）+ Step 5（`GRANT ... ON ALL TABLES IN SCHEMA public TO expense_app`）の二重ガードで再投入** | `ALTER DEFAULT PRIVILEGES` は実行ロールが今後作成するテーブルにのみ効く PostgreSQL 仕様。c 案で 000003 以降の作成主体が `expense_owner` に切り替わるため、000002 のマスター実行 default privileges は効かない（F-118 対応） |

> **注（F-117 a 案採用時の追加注意）**: S3 versioning を本計画に**逆らって**有効化する場合（b 案）は、(a) 11-F 引き継ぎで「誤削除復旧可」へ復旧期待値が変わる旨を明記、(b) §7.2 コスト見積りに noncurrent 課金（lifecycle ありの場合は 30 日分のみ）を追記、(c) `terraform destroy` 前に `aws s3api delete-objects` で noncurrent versions を一括削除する手順を §5.8 に追記、の 3 点が必要。MVP では a 案（上流追従）を採るため、本計画では b 案関連の手順は記載しない。

#### 4.3 GitHub Secrets に登録すべき項目（Phase 8 で deploy.yml を有効化する場合のみ）

11-E 完了条件には含めない。参考のみ。

| Secret | 用途 |
|---|---|
| `AWS_ACCESS_KEY_ID` | GHA から ECR push する場合 |
| `AWS_SECRET_ACCESS_KEY` | 同上 |
| もしくは `AWS_DEPLOY_ROLE_ARN` | OIDC 連携を使う場合（推奨） |

§3.4 で「ECR 使わず EC2 上で `git clone + docker build`」案を採るなら、これらは不要。

---

### 5. デプロイ手順（runbook ドラフト）

#### 5.1 事前準備

1. AWS アカウント作成と支払い情報登録（USER）
2. Budget アラートを $5/月で設定（無料枠超過の早期警告）（USER）
3. IAM ユーザー `terraform-deployer` 作成、AdministratorAccess（ポートフォリオ簡略化）、アクセスキー発行（USER）
4. ローカル `~/.aws/credentials` に追加:
   ```
   [expense-saas-portfolio]
   aws_access_key_id = ...
   aws_secret_access_key = ...
   region = ap-northeast-1
   ```
5. キーペア `expense-saas-portfolio` を AWS コンソール（EC2）で作成、`.pem` をローカル保存、`chmod 400`
6. Terraform をローカルにインストール（>= 1.6）
7. JWT 鍵ペア生成:
   ```bash
   mkdir -p /tmp/expense-saas-keys
   openssl genrsa -out /tmp/expense-saas-keys/private.pem 2048
   openssl rsa -in /tmp/expense-saas-keys/private.pem -pubout -out /tmp/expense-saas-keys/public.pem
   ```

#### 5.2 11-D CONDITIONAL の事前消化（指揮役）

11-E 進入条件 4 つのうち、ローカルで完了できるものを先に消化する:

| # | 条件 | 実施タイミング | 担当 |
|---|---|---|---|
| 1 | test DB 起動 + 統合テスト再実行 | **事前消化（Phase 0）** | 指揮役（`/test` スキル） |
| 4 | frontend build をクリーン環境で実行 | **事前消化（Phase 0）** | 指揮役 |
| 2 | 起動済み API/frontend で `npm run e2e` 10件 PASS | **事前消化（Phase 0、ローカル E2E で代替）** | 指揮役（root で `npm run e2e`） |
| 3 | デプロイ環境の時刻同期 smoke | **Phase 7（実環境必須）** | USER + 指揮役 |

条件 1, 2, 4 は本物のローカル環境で先消化することで、AWS デプロイ後の手戻りを減らす。

#### 5.3 Terraform state バックエンド初期化（USER 一回限り）

```bash
# AWS CLI で先行作成
export AWS_PROFILE=expense-saas-portfolio
aws s3api create-bucket \
  --bucket expense-saas-tfstate-<8桁ランダム> \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1
aws s3api put-bucket-versioning \
  --bucket expense-saas-tfstate-<8桁ランダム> \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption \
  --bucket expense-saas-tfstate-<8桁ランダム> \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws dynamodb create-table \
  --table-name expense-saas-tflock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-northeast-1
```

`backend.tf` の `bucket` を上で作成したバケット名に書き換える（または `-backend-config=bucket=...` で渡す）。

#### 5.4 Terraform init / plan / apply（USER）

```bash
cd expense-saas/infra/terraform

# 初期化
terraform init

# tfvars 作成（terraform.tfvars.example をコピー）
cp terraform.tfvars.example terraform.tfvars
# エディタで編集: db_password, allowed_ssh_cidr, jwt_*_pem を埋める
# あるいは環境変数で渡す:
export TF_VAR_db_password='...'
export TF_VAR_jwt_private_key_pem="$(cat /tmp/expense-saas-keys/private.pem)"
export TF_VAR_jwt_public_key_pem="$(cat /tmp/expense-saas-keys/public.pem)"
export TF_VAR_allowed_ssh_cidr="$(curl -s ifconfig.me)/32"

# 計画確認
terraform plan -out=tfplan

# 確認後 apply（10-15 分かかる、特に RDS 作成）
terraform apply tfplan

# outputs 確認
terraform output
# alb_dns_name = "expense-saas-portfolio-alb-xxxxxxxx.ap-northeast-1.elb.amazonaws.com"
# ec2_public_ip = "xx.xx.xx.xx"
# rds_endpoint = (sensitive)
# s3_bucket_name = "expense-saas-receipts-portfolio-xxxxxxxx"
```

#### 5.5 Docker イメージ配布の方針判断（要 USER 判断）

**案A: GHCR（GitHub Container Registry）経由**（推奨）

- メリット: 無料、private repo の image も pull できる、GHA から push 簡単
- デメリット: PAT (GITHUB_TOKEN) を EC2 に渡す必要

**案B: EC2 上で git clone + docker build**

- メリット: レジストリ不要、一番シンプル
- デメリット: ビルド時間が EC2 リソースを食う（t3.micro メモリ 1GB は厳しい、swap 必要）

**案C: ECR**

- メリット: AWS ネイティブ、IAM Role で認証
- デメリット: 無料枠なし（500MB で $0.05/月、データ転送で多少課金）

**推奨**: **案A（GHCR）**。GHA で master push 時に GHCR へ push、EC2 の user_data で `docker pull` する。GITHUB_TOKEN は read-only PAT を SSM Parameter Store SecureString（無料）で管理。

ただし最初の 1 回目は **案B（EC2 上で build）** を使って素早く検証し、運用化フェーズで案A に移行するのが現実的。

##### 5.5.1 3 案で共通化される runbook 終点（F-116 対応）

どの案を採っても以下の終点に到達することを保証する:

1. **EC2 上のローカル image 名は `expense-saas:portfolio` に統一する**（§5.6 で `docker tag` で揃える）
   - systemd unit の `ExecStart` は `expense-saas:portfolio` 固定（案ごとに systemd unit を書き換えない）
   - seed 実行（§6.3.1）も `expense-saas:portfolio` 固定
2. **migration SQL（`db/migrations/`）は image に同梱されていない**（Dockerfile 確認済み、2026-05-08）。したがって 3 案いずれでも EC2 上に SQL を持ち込む必要があり、§5.7.4 で持ち込み手段を案ごとに定義する
3. アプリ起動コマンド（`docker run --env-file ... -v /etc/expense-saas/keys:/app/keys:ro -p 8080:8080 expense-saas:portfolio`）と `/health` 200 確認は案によらず同一

##### 5.5.2 各案で追加で必要となる準備

| 案 | image 取得 | image retag | migration SQL 持ち込み | 認証 |
|---|---|---|---|---|
| A: GHCR | `docker pull ghcr.io/<user>/expense-saas:<tag>` | `docker tag ghcr.io/.../expense-saas:<tag> expense-saas:portfolio` | `git clone` または `scp db/migrations/` | GHCR PAT（read:packages） |
| B: EC2 build | `git clone && docker build -t expense-saas:portfolio .` | retag 不要（build 時に名前付け） | git clone 済み | 不要 |
| C: ECR | `docker pull <account>.dkr.ecr.<region>.amazonaws.com/expense-saas:<tag>` | `docker tag <account>.dkr.ecr.../expense-saas:<tag> expense-saas:portfolio` | `git clone` または `scp db/migrations/` | EC2 instance profile に ECR ReadOnly |

#### 5.6 EC2 上でのアプリ起動（USER）

apply 完了後の user_data 自動実行か、手動 ssh で:

```bash
# ssh
ssh -i ~/.ssh/expense-saas-portfolio.pem ec2-user@<ec2_public_ip>

# Docker 動作確認
sudo systemctl status docker

# 鍵ペア配置（scp で先に置く想定）
sudo mkdir -p /etc/expense-saas/keys
sudo cp /tmp/private.pem /etc/expense-saas/keys/private.pem
sudo cp /tmp/public.pem /etc/expense-saas/keys/public.pem
sudo chmod 600 /etc/expense-saas/keys/*.pem

# 環境変数ファイル
sudo tee /etc/expense-saas/app.env <<EOF
PORT=8080
LOG_LEVEL=info
DATABASE_URL=postgres://expense_owner:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require
APP_DATABASE_URL=postgres://expense_app:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require
JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
S3_BUCKET=<bucket>
AWS_REGION=ap-northeast-1
CORS_ALLOWED_ORIGINS=http://<alb-dns>
EOF
sudo chmod 600 /etc/expense-saas/app.env

# ─────────────────────────────────────────
# Docker イメージ取得（§11 Q1 で選んだ案ごとに分岐、F-116 対応）
#   いずれの案でも最終的に `expense-saas:portfolio` というローカル image 名で揃える
#   （systemd unit の ExecStart が `expense-saas:portfolio` を参照するため）
# ─────────────────────────────────────────

# 案A (GHCR) の場合 ────────────────────────
# 前提: GHA で master push 時に ghcr.io/<user>/expense-saas:<tag> を push 済み
# <tag> は `latest` または git SHA。recommended は git SHA（runbook 上で再現性が高い）
sudo dnf install -y git
echo "$GHCR_PAT" | sudo docker login ghcr.io -u <user> --password-stdin
GHCR_TAG="latest"   # または "<git-sha>"
sudo docker pull "ghcr.io/<user>/expense-saas:${GHCR_TAG}"
# systemd ExecStart の image 名と揃えるため retag
sudo docker tag "ghcr.io/<user>/expense-saas:${GHCR_TAG}" expense-saas:portfolio
# migration SQL は image に同梱されていないため、§5.7.4 の手順で別途持ち込む
git clone https://github.com/<user>/expense-saas.git ~/expense-saas  # migrations のみ使う
sudo docker logout ghcr.io

# 案B (EC2 build) の場合 ────────────────────
sudo dnf install -y git
git clone https://github.com/<user>/expense-saas.git ~/expense-saas
cd ~/expense-saas
sudo docker build -t expense-saas:portfolio .
# migration SQL は git clone 済みのものを使う

# 案C (ECR) の場合 ────────────────────────
# 前提: GHA から ECR push 済み、EC2 instance profile に
#       AmazonEC2ContainerRegistryReadOnly 同等のポリシーが付与済み
sudo dnf install -y git awscli
ECR_REGISTRY="<account-id>.dkr.ecr.ap-northeast-1.amazonaws.com"
ECR_TAG="latest"   # または "<git-sha>"
aws ecr get-login-password --region ap-northeast-1 \
  | sudo docker login --username AWS --password-stdin "$ECR_REGISTRY"
sudo docker pull "${ECR_REGISTRY}/expense-saas:${ECR_TAG}"
sudo docker tag "${ECR_REGISTRY}/expense-saas:${ECR_TAG}" expense-saas:portfolio
# migration SQL は image 同梱されていないため git clone（または scp）
git clone https://github.com/<user>/expense-saas.git ~/expense-saas

# ─────────────────────────────────────────
# 共通: image 取得後の確認
# ─────────────────────────────────────────
sudo docker images | grep expense-saas
# 期待: REPOSITORY=expense-saas, TAG=portfolio が存在する

# systemd unit 配置
sudo tee /etc/systemd/system/expense-saas.service <<'EOF'
[Unit]
Description=Expense SaaS API
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStartPre=-/usr/bin/docker rm -f expense-saas
ExecStart=/usr/bin/docker run --name expense-saas \
  --env-file /etc/expense-saas/app.env \
  -v /etc/expense-saas/keys:/app/keys:ro \
  -p 8080:8080 \
  expense-saas:portfolio
ExecStop=/usr/bin/docker stop expense-saas

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now expense-saas
sudo systemctl status expense-saas
```

#### 5.7 DB マイグレーション投入（USER + 指揮役）

EC2 から実行（RDS は private なので EC2 経由が最も簡単）。

##### 5.7.1 確認済み事項（B-01 検証結果）

実コード調査（2026-05-08 時点）の結果は以下:

- **RLS ポリシー本体は `expense-saas/db/migrations/*.up.sql` に内包されている**:
  - 以下 6 テーブルで `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` + `CREATE POLICY tenant_isolation_{select,insert,update,delete}` を定義:
    - `000003_create_tenants.up.sql` (tenants)
    - `000005_create_tenant_memberships.up.sql` (tenant_memberships)
    - `000006_create_categories.up.sql` (categories)
    - `000007_create_expense_reports.up.sql` (expense_reports)
    - `000008_create_expense_items.up.sql` (expense_items)
    - `000009_create_attachments.up.sql` (attachments)
  - ポリシーは `USING (tenant_id = current_setting('app.current_tenant')::uuid)` 形式
  - `users / refresh_tokens / password_reset_tokens` は RLS 非対象（仕様）。`users` は `tenant_id` を持たず、認可はアプリ層で `tenant_memberships` 経由で行う設計
  - **したがって `migrate up` 完了でテナント分離 RLS は完全に反映される**。手動で追加 SQL を流す必要はない
- **`expense-saas/scripts/init-db.sh` は localdev 用の自動初期化スクリプト**:
  - 内容は `expense_app` ロール作成（パスワード固定 `localdev`）+ スキーマ USAGE + 既存テーブル DML 権限 + ALTER DEFAULT PRIVILEGES のみ
  - 本番（portfolio 環境）では `/docker-entrypoint-initdb.d/` 経由で実行されないため、**同等の SQL を psql で手動投入する必要がある**
- **`expense-saas/db/migrations/000002_create_roles.up.sql` が prod でも `expense_owner` / `expense_app` を `PASSWORD 'localdev'` で作成する**（F-115 検証根拠）:
  - migration 内で `IF NOT EXISTS` ガード付きで `CREATE ROLE ... WITH LOGIN PASSWORD 'localdev'` を 2 ロール分実行する
  - **GRANT（USAGE, 既存テーブル DML, ALTER DEFAULT PRIVILEGES）は同 migration に含まれているが、c 案では「マスター実行」になる**ため、(a) `GRANT ... ON ALL TABLES` は実行時点で業務テーブル不在で空振り、(b) `ALTER DEFAULT PRIVILEGES IN SCHEMA public ...` は **「実行ロール（=マスター）が今後作成するテーブル」にのみ効く** という PostgreSQL 仕様により、000003 以降を `expense_owner` が作成するシナリオでは `expense_app` への自動付与が成立しない。**`expense_app` の本番 9 テーブルへの DML は §5.7.3 Step 3（`ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner ...`）と Step 5（`GRANT ... ON ALL TABLES IN SCHEMA public TO expense_app`）で再投入する**（F-118 対応）
  - 残課題は **localdev パスワードの上書き** + **schema_migrations の owner 移管** + **expense_app への DML 再付与** の 3 点。後続で `IF NOT EXISTS` 付き `CREATE ROLE` を流しても既存ロール扱いで NOOP になり、`APP_DATABASE_URL` の prod パスワードと不一致が起こるため、**§5.7.3 では `CREATE ROLE` ではなく `ALTER ROLE ... WITH PASSWORD` で必ず上書きする** 手順に統一する（F-115 対応）
- **migration 実行権限の二層性**（F-115 再修正で確認）:
  - `000001_create_extensions.up.sql` は `CREATE EXTENSION pgcrypto` を実行する。RDS PostgreSQL では `pgcrypto` の作成に `rds_superuser` が必要なため、`expense_owner` では `permission denied` で失敗する。
  - `000002_create_roles.up.sql` は `CREATE ROLE` を実行する。superuser が必要であり、かつ `expense_owner` ロール自身がこの migration で作られるため、論理的にも `expense_owner` 接続では実行不可能。
  - 一方 000003 以降は `CREATE TABLE` 系で、上流 `release.md` §4.1 / 実装 `cmd/server/main.go` line 60-82 / `cmd/seed/main.go` line 27-33 / `scripts/init-db-test.sh` line 35-39 のいずれも **`expense_owner` がテーブル owner として成立している**前提。したがって 000003 以降は `expense_owner` で実行する必要がある。
  - 結論: **000001 / 000002 は RDS マスター、000003 以降は `expense_owner`** という分割が、上流前提と migration 実装の両方を満たす唯一の経路（§5.7.3 c 案）。
- **シーケンス使用の有無**（F-118 補足検討）:
  - `db/migrations/000003_*.up.sql` 〜 `000011_*.up.sql` の `CREATE TABLE` 定義を全件 grep（`SERIAL` / `IDENTITY` / `BIGSERIAL` / `nextval` / `CREATE SEQUENCE`）した結果、**MVP 9 テーブルは全て UUID 主キー（`UUID PRIMARY KEY DEFAULT gen_random_uuid()` または複合主キー）で、シーケンスは未使用**。
  - したがって `GRANT USAGE ON SEQUENCES TO expense_app` は現状必要ない。ただし将来テーブル追加でシーケンスが導入されたときの保険として、§5.7.3 Step 3 の `ALTER DEFAULT PRIVILEGES ... GRANT USAGE ON SEQUENCES` と Step 5 の `GRANT USAGE ON ALL SEQUENCES IN SCHEMA public` は害がないため runbook に含めておく（実行時は対象 0 件）。
- **`schema_migrations` の所有権**（F-115 残課題）:
  - golang-migrate は接続ロールで `schema_migrations` テーブルを作成する。c 案では Step 1 の `migrate up 2` をマスター接続で実行するため、**Step 1 完了時点で `schema_migrations.owner = master`**。Step 4 で `expense_owner` 接続から `migrate up` を継続するには、`schema_migrations` への INSERT/UPDATE 権限が必要だが、PostgreSQL のデフォルト権限モデルでは取得できない（master owner のテーブル）。
  - したがって §5.7.3 Step 2 でマスター権限のうちに **`ALTER TABLE schema_migrations OWNER TO expense_owner`** を実行する手順を追加する（F-115 残課題対応）。

##### 5.7.2 prod 用パスワード生成と保存

`expense_owner` / `expense_app` の DB パスワードを事前に**それぞれ独立に**生成し、以下 2 経路で保管する。両ロールとも migration により `localdev` パスワードで作成されるため、後続の `ALTER ROLE` で上書きすることが前提（§5.7.1 / F-115）。

```bash
# パスワード生成（ローカル、一回限り）。同一値の使い回しを避け、ロールごとに別値とする
OWNER_DB_PW=$(openssl rand -base64 24 | tr -d '=+/' | cut -c1-32)
APP_DB_PW=$(openssl rand -base64 24 | tr -d '=+/' | cut -c1-32)
echo "$OWNER_DB_PW"  # ← 控える（DATABASE_URL 用）
echo "$APP_DB_PW"    # ← 控える（APP_DATABASE_URL 用）
```

| 保存先 | 用途 |
|---|---|
| `terraform.tfvars` の `expense_owner_db_password` / `expense_app_db_password`（sensitive 変数として `variables.tf` に追加） | EC2 user_data から `/etc/expense-saas/app.env` の `DATABASE_URL` / `APP_DATABASE_URL` に展開する経路（§4.2 / §5.6） |
| 手元のパスワードマネージャ | psql 手動投入時に使う（5.7.3 で `ALTER ROLE` の値として使用） |
| RDS マスターユーザーパスワード（`db_master_password`、tfvar） | §5.7.3 で `ALTER ROLE` を実行する psql 接続用。RDS 作成時の管理ユーザー（`db_master_user` 既定 `postgres`）の認証に使う |

`variables.tf` 追加例:

```hcl
variable "expense_owner_db_password" {
  description = "expense_owner ロールの DB パスワード（migrate 後に RDS マスターユーザーで ALTER ROLE で投入）"
  type        = string
  sensitive   = true
}

variable "expense_app_db_password" {
  description = "expense_app ロールの DB パスワード（migrate 後に RDS マスターユーザーで ALTER ROLE で投入）"
  type        = string
  sensitive   = true
}
```

`terraform output` で値を再表示はしない（sensitive のため）。EC2 user_data 内で `templatefile` 経由で `/etc/expense-saas/app.env` の `DATABASE_URL` / `APP_DATABASE_URL` を組み立てる。

> **note**: ロールごとに別パスワードを採るのは、(a) `expense_owner` は migration 用・`expense_app` は実行時用で漏洩時の影響が違う、(b) env_config.md §6.4 でも 90 日サイクルで個別ローテーション前提、の 2 点による。1 つの PW を両ロール共通にする運用は採用しない。

##### 5.7.3 マイグレーション + ロール投入手順

> **重要（F-115 根本修正、再々修正 2026-05-08）**: 修正の論点は 4 つあり、全てを同時に満たす必要がある。
>
> 1. **パスワード一致**: `000002_create_roles.up.sql` が `expense_owner` / `expense_app` を **`PASSWORD 'localdev'` で作成する**ため、`ALTER ROLE ... WITH PASSWORD '${...}'` で prod 値に上書きしないと `APP_DATABASE_URL` / `DATABASE_URL` の値と DB 上の値が一致せずアプリ起動・migrate のいずれも失敗する。
> 2. **テーブル owner = `expense_owner`**: 上流 `release.md` §4.1 と実装（`cmd/server/main.go` / `cmd/seed/main.go` / `scripts/init-db-test.sh`）はいずれも **`DATABASE_URL=postgres://expense_owner:...` で接続し、`expense_owner` がテーブル owner かつ RLS bypass ロールである**前提で書かれている。`migrate up` を **RDS マスターユーザーで一括実行する**と、000003 以降で作成される全テーブル（tenants / users / tenant_memberships / categories / expense_reports / expense_items / attachments / refresh_tokens / password_reset_tokens / schema_migrations）の owner がマスターになり、上流前提が崩れる。
> 3. **`schema_migrations` の owner 移管（F-115 残課題）**: golang-migrate は接続ロールで `schema_migrations` を作成するため、Step 1 完了時点で owner はマスター。Step 3 を `expense_owner` 接続で継続するには、Step 2 で **`ALTER TABLE schema_migrations OWNER TO expense_owner`** が必要。これを欠くと Step 3 開始時点で `INSERT/UPDATE` 権限不足で migrate 不能。
> 4. **`expense_app` への DML 権限再付与（F-118 新規 blocker）**: `000002_create_roles.up.sql` の `ALTER DEFAULT PRIVILEGES IN SCHEMA public` は **「実行したロール（=マスター）が後で作るテーブル」にのみ効く** PostgreSQL 仕様。c 案では 000003 以降を `expense_owner` が作成するため、本番 9 テーブルへの `expense_app` の DML が**自動付与されない**。Step 3a で `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner ... GRANT ... TO expense_app` を投入し、Step 4 で既存テーブル一括 `GRANT ... ON ALL TABLES` を保険として実行する二重ガードが必要。
>
> したがって migrate 実行主体を **「000001 / 000002 はマスター」「000003 以降は `expense_owner`」と分割**し、000002 完了直後に `ALTER ROLE` + `schema_migrations` owner 移管 + `expense_owner` 用 default privileges を仕込んでから残りを進める。Step 4 完了後に `expense_app` への保険 GRANT と疎通確認（`SELECT * FROM tenants LIMIT 1` 等）で DML 権限の実地検証まで行う。

###### 採用案の論拠（旧 a 案 / b 案 / 新案 c 案 の比較）

| 案 | migrate 実行主体 | テーブル owner | パスワード一致 | 採否 |
|---|---|---|---|---|
| 旧 a 案 | 全て `expense_owner`（localdev で接続） | `expense_owner` | migrate 後 `ALTER ROLE` で揃う | **不採用**: 000001 の `CREATE EXTENSION pgcrypto` は superuser 権限必須、000002 の `CREATE ROLE` も superuser 権限必須。`expense_owner` は両方とも実行できない |
| 旧 b 案 | 全て RDS マスター | **マスター（NG）** | migrate 後 `ALTER ROLE` で揃う | **不採用（commit d714373 の副作用）**: テーブル owner がマスターになり、`expense_owner` で接続するアプリ・seed・migration down が権限不足になる |
| **新 c 案** | 000001/000002 はマスター → ALTER ROLE + `schema_migrations` owner 移管 + default privileges 仕込み → 000003 以降は `expense_owner` → 既存テーブル一括 GRANT で expense_app DML を担保 | **`expense_owner`** | 000002 直後の `ALTER ROLE` で揃う | **採用（F-115/F-118 反映）** |

###### 手順全体像（c 案、F-115 残課題 + F-118 反映）

1. **Step 1（マスター）**: RDS マスターユーザーで psql/migrate 接続を確立し、**`migrate up 2`** で 000001（CREATE EXTENSION）+ 000002（CREATE ROLE）のみを進める。golang-migrate の `up N` は最新から N ステップ進める指定で、初期状態 v0 からは「先頭から N 本」を意味する。この時点で `schema_migrations` テーブルがマスター owner で作成される。
2. **Step 2（マスター、ロール / 権限基盤の整備）**: 同じマスター接続で以下を一括投入する。
    - `ALTER ROLE expense_owner / expense_app WITH PASSWORD '${...}'` で両ロールのパスワードを prod 値に上書き
    - `GRANT CREATE, USAGE ON SCHEMA public TO expense_owner`（PG15+ 対応、Step 3 で `CREATE TABLE` するため必要）
    - **`ALTER TABLE schema_migrations OWNER TO expense_owner`**（Step 3 で `expense_owner` 接続が migrate を継続できるよう、管理テーブルの owner を移管。F-115 残課題対応）
3. **Step 3（接続切替→`expense_owner`、default privileges 仕込み）**: 接続主体を `expense_owner` に切り替え、000003 以降の `CREATE TABLE` 前に `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO expense_app` を投入する。これにより以降 `expense_owner` が作成するテーブルへ `expense_app` の DML が自動付与される（F-118 一次対策）。
4. **Step 4（`expense_owner` で migrate 継続）**: `migrate up`（バージョン無指定 = 残り全部）で 000003 以降を進める。これにより 000003〜000011 で `CREATE TABLE` されるテーブルの owner はすべて `expense_owner` になり、Step 3 の default privileges 経由で `expense_app` の DML 権限も自動付与される。
5. **Step 5（`expense_owner` で保険 GRANT）**: 既存テーブル全件への明示 GRANT として `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO expense_app` を実行（F-118 二重ガード）。シーケンスは MVP 範囲では未使用（全テーブル UUID 主キー）だが、害がないため `GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO expense_app` も保険で投入する。
6. **Step 6（疎通確認）**: `expense_owner` / `expense_app` の双方で `SELECT 1`、加えて `expense_app` で **`has_table_privilege()` で 9 テーブル × 4 権限のメタデータ検証**（RLS の影響を受けない）+ **RLS 非対象 `users` への live `SELECT count(*)`** を実行して DML 系権限を実地検証する（F-119 対応：RLS 対象テーブルへの直接 SELECT は tenant context 未設定で false negative になるため避ける）。
7. **Step 7（環境変数反映 + 再起動）**: `/etc/expense-saas/app.env` の `DATABASE_URL` / `APP_DATABASE_URL` を確定値で更新後、systemd 再起動でアプリ本体に prod パスワードを反映する。

> **000001 / 000002 をマスターで実行する根拠**:
> - `000001_create_extensions.up.sql` は `CREATE EXTENSION IF NOT EXISTS pgcrypto;` 一行のみ。RDS PostgreSQL では pgcrypto は `rds_superuser` 権限が必要で、`expense_owner` では `permission denied to create extension "pgcrypto"` で失敗する。
> - `000002_create_roles.up.sql` は `CREATE ROLE expense_owner / expense_app` を実行する。これらのロールが存在しない時点で `expense_owner` 接続自体が成立しないため、論理的にも `expense_owner` では実行できない。
> - これら 2 本にはテーブル / シーケンスの作成は含まれないため、マスターで実行しても **テーブル owner 問題は発生しない**。owner が問題になるのは 000003 以降の `CREATE TABLE` 系のみ。

###### 実コマンド

```bash
# EC2 上
ssh -i ~/.ssh/expense-saas-portfolio.pem ec2-user@<ec2_public_ip>

# psql / golang-migrate を取得
sudo dnf install -y postgresql16
GOMIG_VER=v4.17.0
curl -L https://github.com/golang-migrate/migrate/releases/download/${GOMIG_VER}/migrate.linux-amd64.tar.gz \
  | tar xz
sudo mv migrate /usr/local/bin/

# 5.7.2 で生成したパスワードを export（履歴に残らないよう read -s を使う）
read -s -p "OWNER_DB_PW: " OWNER_DB_PW; echo
read -s -p "APP_DB_PW:   " APP_DB_PW;   echo
read -s -p "MASTER_PW:   " MASTER_PW;   echo

RDS_HOST="<rds-endpoint>"
DB_NAME="expense_saas"
DB_MASTER_USER="${db_master_user:-postgres}"  # tfvar で上書きしていない限り postgres

# マイグレーション SQL を持ち込み（§5.7.4 / Q1 案ごとに方法が異なる）
# 例: 案B（EC2 build）の場合は git clone 済みなのでそのまま。
cd ~/expense-saas

# ─────────────────────────────────────────
# Step 1: 000001 + 000002 をマスターで実行
#   - 000001: CREATE EXTENSION pgcrypto（rds_superuser 必須）
#   - 000002: CREATE ROLE expense_owner / expense_app（superuser 必須、PASSWORD 'localdev' で作成される）
#             + GRANT / ALTER DEFAULT PRIVILEGES（後者はマスター実行のため expense_owner が後で作る
#             テーブルには効かない。F-118 で Step 3 にて再投入する）
#   - この時点ではまだ業務テーブル作成は走らないので owner 問題は発生しない
#   - schema_migrations テーブルが本ステップで「マスター owner」として作成される（Step 2 で移管）
# ─────────────────────────────────────────
export MIGRATE_DB_URL_MASTER="postgres://${DB_MASTER_USER}:${MASTER_PW}@${RDS_HOST}:5432/${DB_NAME}?sslmode=require"
migrate -path db/migrations -database "$MIGRATE_DB_URL_MASTER" up 2

# 確認: schema_migrations.version = 2 / dirty = false
PGPASSWORD="$MASTER_PW" psql -h "$RDS_HOST" -U "$DB_MASTER_USER" -d "$DB_NAME" \
  -c 'SELECT version, dirty FROM schema_migrations;'

# ─────────────────────────────────────────
# Step 2: マスターでロール / 管理テーブル / スキーマ権限を整備
#   - 両ロールのパスワードを prod 値に上書き（migration が PASSWORD 'localdev' で作成済み）
#   - schema_migrations の owner を expense_owner へ移管（Step 4 の migrate 継続のため必須、F-115 残課題）
#   - public スキーマへの CREATE/USAGE 付与（PG15+ 対応、Step 4 の CREATE TABLE のため必須）
# ─────────────────────────────────────────
export PGPASSWORD="$MASTER_PW"
psql -h "$RDS_HOST" -U "$DB_MASTER_USER" -d "$DB_NAME" -v ON_ERROR_STOP=1 <<SQL
-- 両ロールのパスワードを prod 値に上書き（CREATE ROLE は流さない: 既存 + IF NOT EXISTS で NOOP のため）
ALTER ROLE expense_owner WITH LOGIN PASSWORD '${OWNER_DB_PW}';
ALTER ROLE expense_app   WITH LOGIN PASSWORD '${APP_DB_PW}';

-- expense_owner が public スキーマ上にテーブルを作成できるよう、CREATE 権限を保証する
-- （RDS のデフォルトでは postgres マスターのみが public に CREATE 可。
--  expense_owner で 000003 以降の CREATE TABLE を実行するために必要）
GRANT CREATE, USAGE ON SCHEMA public TO expense_owner;

-- schema_migrations の owner を expense_owner に移管する（F-115 残課題）
-- Step 1 で golang-migrate がマスター接続側で作成しているため owner = マスター。
-- このまま Step 4 で expense_owner 接続から migrate up を継続すると、
-- INSERT/UPDATE 権限不足で「permission denied for table schema_migrations」になる。
ALTER TABLE schema_migrations OWNER TO expense_owner;

-- 確認
\du expense_owner
\du expense_app
\dt schema_migrations
-- expected: schema_migrations の Owner 列が expense_owner
SQL
unset PGPASSWORD

# 確認 SQL（schema_migrations 移管確認）— 単独でも実行できるよう抜き出し
PGPASSWORD="$MASTER_PW" psql -h "$RDS_HOST" -U "$DB_MASTER_USER" -d "$DB_NAME" <<'SQL'
SELECT tablename, tableowner
  FROM pg_tables
 WHERE schemaname = 'public' AND tablename = 'schema_migrations';
-- expected: tableowner = expense_owner
SQL

# ─────────────────────────────────────────
# Step 3: 接続切替→ expense_owner 用 default privileges を仕込む（F-118 一次対策）
#   - 000002 の ALTER DEFAULT PRIVILEGES はマスター実行だったため、
#     expense_owner が後で作るテーブルには適用されない（PostgreSQL 仕様：
#     ALTER DEFAULT PRIVILEGES は「実行ロール」の今後の作成物にのみ効く）。
#   - そこで FOR ROLE expense_owner で改めて投入する。
#   - ※ 003 以降の migrate を始める「前」に投入することで、CREATE TABLE と同時に
#     expense_app への DML 権限が自動付与される。
# ─────────────────────────────────────────
export PGPASSWORD="$OWNER_DB_PW"
psql -h "$RDS_HOST" -U expense_owner -d "$DB_NAME" -v ON_ERROR_STOP=1 <<'SQL'
-- expense_owner が今後 public スキーマで作成するテーブルへ、自動で expense_app に DML を付与する
ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO expense_app;

-- シーケンスは MVP 範囲では未使用（全テーブル UUID 主キー、000003〜000011 検査済み）
-- だが、害がないため将来追加された場合の保険として default privileges だけ仕込む
ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO expense_app;

-- 確認
\ddp
SQL
unset PGPASSWORD

# ─────────────────────────────────────────
# Step 4: 残りの migration（000003 以降）を expense_owner で実行
#   - これにより 000003〜000011 で CREATE TABLE されるテーブルの owner はすべて expense_owner
#   - Step 3 の default privileges により、各 CREATE TABLE と同時に expense_app へ DML が付与される
#   - schema_migrations は Step 2 で expense_owner へ移管済みなので INSERT/UPDATE できる
# ─────────────────────────────────────────
export MIGRATE_DB_URL_OWNER="postgres://expense_owner:${OWNER_DB_PW}@${RDS_HOST}:5432/${DB_NAME}?sslmode=require"
migrate -path db/migrations -database "$MIGRATE_DB_URL_OWNER" up

# 確認: 最終バージョン到達 / テーブル owner が expense_owner であること
PGPASSWORD="$OWNER_DB_PW" psql -h "$RDS_HOST" -U expense_owner -d "$DB_NAME" \
  -c 'SELECT version, dirty FROM schema_migrations;'
PGPASSWORD="$OWNER_DB_PW" psql -h "$RDS_HOST" -U expense_owner -d "$DB_NAME" <<'SQL'
-- 主要テーブルの owner を一覧（expected: 全行 expense_owner、schema_migrations を含む）
SELECT schemaname, tablename, tableowner
  FROM pg_tables
 WHERE schemaname = 'public'
 ORDER BY tablename;
SQL

# ─────────────────────────────────────────
# Step 5: 既存テーブルへの保険 GRANT（F-118 二重ガード）
#   - default privileges に頼らず、既に作られた本番 9 テーブル全件へ
#     expense_app の DML を明示付与する（万一 Step 3 が漏れてもここで埋まる）。
#   - GRANT は冪等なので、Step 3 の自動付与と二重に投入しても害はない。
# ─────────────────────────────────────────
export PGPASSWORD="$OWNER_DB_PW"
psql -h "$RDS_HOST" -U expense_owner -d "$DB_NAME" -v ON_ERROR_STOP=1 <<'SQL'
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO expense_app;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO expense_app;  -- 現在は対象 0 件、保険

-- 確認: 代表テーブルへの権限が expense_app に付与されているか
SELECT grantee, privilege_type
  FROM information_schema.role_table_grants
 WHERE table_schema = 'public'
   AND table_name = 'tenants'
   AND grantee = 'expense_app'
 ORDER BY privilege_type;
-- expected: SELECT / INSERT / UPDATE / DELETE の 4 行
SQL
unset PGPASSWORD

# ─────────────────────────────────────────
# Step 6: 疎通確認（両ロール、DML 権限の実地検証含む）
# ─────────────────────────────────────────
PGPASSWORD="$OWNER_DB_PW" psql -h "$RDS_HOST" -U expense_owner -d "$DB_NAME" -c 'SELECT 1;'
PGPASSWORD="$APP_DB_PW"   psql -h "$RDS_HOST" -U expense_app   -d "$DB_NAME" -c 'SELECT 1;'

# F-118 受け入れ条件: expense_app の DML 権限を実地検証する（F-119 対応で 2 段階化）
# (a) メタデータ検証: has_table_privilege() で全業務テーブルの SELECT/INSERT/UPDATE/DELETE を確認
#     has_table_privilege() は RLS の影響を受けず権限の有無のみを返すため、
#     RLS 対象テーブル（tenants / tenant_memberships / categories / expense_reports /
#     expense_items / attachments）でも tenant context 設定なしで安全に検証できる
PGPASSWORD="$APP_DB_PW" psql -h "$RDS_HOST" -U expense_app -d "$DB_NAME" <<'SQL'
SELECT
    table_name,
    has_table_privilege('expense_app', 'public.' || table_name, 'SELECT') AS has_select,
    has_table_privilege('expense_app', 'public.' || table_name, 'INSERT') AS has_insert,
    has_table_privilege('expense_app', 'public.' || table_name, 'UPDATE') AS has_update,
    has_table_privilege('expense_app', 'public.' || table_name, 'DELETE') AS has_delete
FROM (VALUES
    ('tenants'), ('tenant_memberships'), ('categories'),
    ('users'), ('expense_reports'), ('expense_items'), ('attachments'),
    ('refresh_tokens'), ('password_reset_tokens')
) AS t(table_name)
ORDER BY table_name;
-- 全 9 テーブルで has_select / has_insert / has_update / has_delete が全て t（true）であること
-- f が 1 つでもあれば工程失敗、Step 3 の ALTER DEFAULT PRIVILEGES または Step 5 の一括 GRANT を再実行
SQL

# (b) ライブ実地検証: RLS 非対象の `users` への SELECT で実接続を確認
#     users / refresh_tokens / password_reset_tokens は RLS 非対象（§5.7.1 参照、
#     tenant_id を持たず認可はアプリ層 tenant_memberships 経由）。
#     したがって app.current_tenant 未設定でも SELECT が成立する
PGPASSWORD="$APP_DB_PW" psql -h "$RDS_HOST" -U expense_app -d "$DB_NAME" -c 'SELECT count(*) FROM users;'
# 0 行（seed 前）が返ること。permission denied が出たら工程失敗（has_table_privilege が t を返したのに
# 実 SELECT が失敗するなら server / connection 経路の権限問題）

# 注意: tenants / expense_reports など RLS 対象テーブルへの直接 SELECT は実施しない。
# RLS policy が `current_setting('app.current_tenant')::uuid` を要求するため、
# tenant context 未設定の psql 直接実行では false negative（権限ありなのに 0 行返却）になる（F-119）。
# 本物のテナント分離検証はアプリ起動後の Phase 7 スモーク（実 API 経由でのフロー1）で行う

# ─────────────────────────────────────────
# Step 7: 環境変数ファイル更新（§5.6 で雛形を置いた /etc/expense-saas/app.env を ALTER 後の値で確定）
# ─────────────────────────────────────────
# DATABASE_URL=postgres://expense_owner:${OWNER_DB_PW}@${RDS_HOST}:5432/${DB_NAME}?sslmode=require
# APP_DATABASE_URL=postgres://expense_app:${APP_DB_PW}@${RDS_HOST}:5432/${DB_NAME}?sslmode=require
# 上記 2 行を /etc/expense-saas/app.env で確実に更新後、systemd を再起動
sudo systemctl restart expense-saas
```

> **F-115 / F-118 修正の要点（再々修正版、2026-05-08）**:
> - Step 1 で **`migrate up 2` のみ** をマスターで実行する（000001/000002 のみ）。`up`（無引数）でないことに注意。`schema_migrations` 自体もマスター owner で生成される。
> - Step 2 の `ALTER ROLE` を**省略するとパスワード不一致でアプリ起動失敗**（リスク表 §8 #14 ゲート条件）。
> - Step 2 の `GRANT CREATE, USAGE ON SCHEMA public TO expense_owner` は、PostgreSQL 15+ のデフォルト（`PUBLIC` から `CREATE` 権限剥奪）に対応するための保険。Step 4 で `expense_owner` が public スキーマに `CREATE TABLE` する権限を担保する。
> - **Step 2 の `ALTER TABLE schema_migrations OWNER TO expense_owner` を省略すると、Step 4 の `migrate up` が `permission denied for table schema_migrations` で失敗する**（F-115 残課題ゲート条件、リスク表 §8 #14）。Step 2 末尾の `\dt schema_migrations` で owner = `expense_owner` を確認すること。
> - **Step 3 の `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner` は省略禁止**。000002 のマスター実行版 default privileges は本番 9 テーブルに効かないため、Step 3 を欠くと 000003 以降のテーブルへの `expense_app` の DML が空になる（F-118 ゲート条件、リスク表 §8 #16）。
> - Step 4 の `migrate up`（無引数）は schema_migrations.version=2 から再開し、000003〜最終までを `expense_owner` 接続で実行する。これにより `CREATE TABLE` の owner は `expense_owner` になり、Step 3 の default privileges 経由で `expense_app` の DML 権限も同時に自動付与される。
> - Step 4 の owner 確認 SQL (`pg_tables.tableowner`) が **「全テーブル `expense_owner`」（schema_migrations 含む）を返さない場合は工程失敗**。マスターで `ALTER TABLE ... OWNER TO expense_owner` で個別移管するか、`terraform destroy` + RDS 再作成からやり直す。
> - **Step 5 の `GRANT ... ON ALL TABLES IN SCHEMA public TO expense_app` は二重ガード**。Step 3 が万一漏れても、ここで本番 9 テーブル全件への DML 権限を明示的に埋める。`information_schema.role_table_grants` の確認 SQL で `tenants` への 4 種権限が揃っていることをゲートする。
> - シーケンス使用の有無は **000003〜000011 を全件 grep 済み（全テーブル UUID 主キー、`SERIAL` / `IDENTITY` / `nextval` / `CREATE SEQUENCE` 不在）** のため、`USAGE ON ALL SEQUENCES` は将来用の保険のみで現状は対象 0 件である。
> - Step 6 の `expense_app` 検証は **(a) `has_table_privilege()` メタデータ検証 + (b) RLS 非対象 `users` への live `SELECT count(*)`** の 2 段階（F-118 受け入れ条件 #4 ゲート条件、F-119 対応）。`tenants` / `expense_reports` 等 RLS 対象テーブルへの直接 SELECT は `app.current_tenant` 未設定で false negative（権限ありでも 0 行返却が permission denied と区別できない）になるため使用しない。本物のテナント分離検証は Phase 7 のスモークテスト（実 API 経由）で行う。
> - Step 1 の version 確認で `dirty=true` が出た場合は `migrate force` ではなく RDS スナップショットからリストアして再試行する（Step 1 失敗時のみ。Step 4 で `dirty=true` なら 000003 以降を down → up し直す）。

##### 5.7.4 マイグレーション SQL の EC2 持ち込み（Q1 案ごと）

`migrate up` には EC2 上に `db/migrations/` の SQL ファイル群が必要。Dockerfile（2026-05-08 確認）は **`db/migrations/` を image に同梱していない**（runtime stage が `server` / `seed` バイナリのみコピーするため）。したがって Q1 のいずれの案でも、**migration SQL の持ち込みは別途必要**。

| Q1 案 | migration SQL の持ち込み方法 |
|---|---|
| 案A（GHCR） | (a) `git clone https://github.com/<user>/expense-saas.git ~/expense-saas`（EC2 上で migration 用に clone のみ。docker build はしない）、または (b) ローカルから `scp -i ~/.ssh/expense-saas-portfolio.pem -r expense-saas/db/migrations ec2-user@<ec2_public_ip>:~/migrations/` で送り `migrate -path ~/migrations -database "$MIGRATE_DB_URL" up`。**(a) を推奨**（git で改ざん検知可、checkout タグを runbook 上で固定できる） |
| 案B（EC2 build） | `git clone` 済みなので `cd ~/expense-saas && migrate -path db/migrations ...` のままでよい |
| 案C（ECR） | 案A と同様。`git clone` または `scp` |

> **note**: 将来 Dockerfile の runtime stage に `COPY --from=builder /build/db/migrations ./db/migrations` を追加すれば `docker run --rm --entrypoint sh expense-saas:portfolio -c 'cat /app/db/migrations/...'` 形式で取り出せるが、image サイズが増えるため MVP では未対応とする。

> **注**: env_config.md §5.2 では prod の DB 認証情報は Secrets Manager 管理を想定しているが、本計画 §4.1 で「Secrets Manager は無料枠外」のため不採用とし、tfvars + EC2 ファイル配置（`/etc/expense-saas/app.env`）で代替する。env_config.md prod 構成との差分は §4.2 末尾および 11-F 引き継ぎに明記する。

#### 5.8 ロールバック手順

| 失敗箇所 | 手順 |
|---|---|
| `terraform apply` 失敗 | エラー内容確認後 `terraform destroy` で全削除、または部分的に `terraform state rm` してリトライ |
| EC2 起動成功・アプリ起動失敗 | `docker logs expense-saas` でログ確認。`systemctl restart expense-saas` |
| マイグレーション失敗 | `migrate -database "$DATABASE_URL" down 1`、ダメなら RDS スナップショットからリストア |
| 全体的な切り戻し | `terraform destroy` → state は残るので再度 apply で再現可能 |

---

### 6. 検証手順

#### 6.1 ヘルスチェック疎通（指揮役）

> ※TLS 構成は ADR-0007（CloudFront 前段化）を正とする。エンドユーザー向けの公開エンドポイントは CloudFront ドメイン（`https://<cloudfront_domain_name>/`）であり、ALB DNS への直アクセスは B-1-b により遮断され 403 を返す。`<cloudfront_domain_name>` / `<alb_dns_name>` は `terraform output` で取得する output 名（apply 後に確定）。

```bash
# CloudFront 経由（エンドユーザーと同一経路。これが正規の疎通確認）
curl -i https://<cloudfront_domain_name>/health
# 期待: HTTP/2 200, body {"status":"ok",...}

# ALB DNS 直アクセス（B-1-b 閉域確認。CloudFront のカスタムヘッダ X-Origin-Verify 不在のため遮断される）
curl -i http://<alb_dns_name>/health
# 期待: HTTP/1.1 403 Forbidden（ALB リスナーのデフォルトアクション）
# ※200 が返った場合は B-1-b の閉域化が機能していない。restrict_alb_to_cloudfront / カスタムヘッダ検証の設定を確認する

# EC2 直接（ALB をバイパスして切り分け）
curl -i http://<ec2_public_ip>:8080/health
# ※ EC2 SG が ALB SG のみ許可なら外からは 8080 直叩きは届かない（正しい）

# ssh で EC2 内から
ssh ec2-user@<ec2_public_ip> 'curl -s http://localhost:8080/health'
```

#### 6.2 11-D CONDITIONAL 4 条件の最終確認タイミング

| # | 条件 | タイミング | 実施方法 | 状態 |
|---|---|---|---|---|
| 1 | test DB + Go 統合テスト | Phase 0（事前消化） | 指揮役 `/test` スキル。ローカル `docker compose --profile test run --rm migrate-test && go test -tags integration ./...` | **完了（2026-05-13、§実施ログ - Phase 0 詳細参照）** |
| 2 | `npm run e2e` 10件 PASS | Phase 0（ローカルで事前消化） | 指揮役 root で `docker compose up -d && npm run e2e`、10 件 PASS をログに記録 | **完了（2026-05-13、10/10 PASS 26.7s、§実施ログ - Phase 0 詳細参照）** |
| 3 | デプロイ環境の時刻ドリフト smoke | **Phase 7（実環境必須）** | EC2: `timedatectl` で chrony 動作確認、`date -u` を JWT iat / exp 境界で確認。RDS: `SHOW timezone; SELECT NOW();` | 未実施（Phase 7） |
| 4 | frontend build をクリーン環境で実行 | Phase 0（事前消化） | 指揮役 `cd frontend && rm -rf dist && npm ci && npm run build` | **完了（2026-05-13、3.86s ビルド成功、§実施ログ - Phase 0 詳細参照）** |

#### 6.3 スモークテスト（申請 → 承認 → 支払）

11-D で確定した E2E `payment_flow.spec.ts`（CRS-055）と同等の操作を、デプロイ済み環境で **手動で** 実行する。テストデータは `seed` コマンドで投入する。

##### 6.3.1 シードデータ投入

`expense-saas/Dockerfile`（2026-05-08 確認）の構成:

- builder stage で `cmd/server` と `cmd/seed` の 2 バイナリをビルド
- runtime stage で両方を `/app/` 配下にコピー（`COPY --from=builder /build/server .` / `COPY --from=builder /build/seed .`）
- **`db/migrations/` は image に同梱されていない**（migration は §5.7.4 の手順で別途 EC2 上に持ち込む）
- デフォルト `CMD ["./server"]`、seed は `--entrypoint /app/seed` で起動可能（W-04 検証 OK）

> **F-116 対応**: 以下の `expense-saas:portfolio` という image 名は §5.6 で 3 案いずれでも `docker tag` により統一済みのため、案A/B/C どれを採っても同じコマンドで動作する。

```bash
# EC2 上で seed を実行
ssh ec2-user@<ec2_public_ip>
cd ~/expense-saas
sudo docker run --rm \
  --env-file /etc/expense-saas/app.env \
  --entrypoint /app/seed \
  expense-saas:portfolio
```

##### 6.3.2 手動スモーク手順

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | ブラウザで `https://<cloudfront_domain_name>/` を開く（エンドユーザーと同一の公開 URL。ALB DNS 直アクセスは B-1-b で 403 となるため使わない） | HTTPS でログイン画面が表示される |
| 2 | Member（`test-member@example.com` / seed 既定パスワード `TestPass1!`、出典: `expense-saas/internal/seed/seed.go:174` `cmd/seed/main.go:89`）でログイン | ダッシュボードに遷移する |
| 3 | レポート作成 → 明細追加（金額・カテゴリ・日付） → 添付（PNG/JPG 1ファイル） → 保存 → 提出 | レポートが submitted 状態になる |
| 4 | ログアウト → Approver（`test-approver@example.com` / `TestPass1!`、出典: `seed.go:173`）でログイン → 承認待ち一覧 → 該当レポートを承認 | レポートが approved 状態になる |
| 5 | ログアウト → Accounting（`test-accounting@example.com` / `TestPass1!`、出典: `seed.go:175`）でログイン → 支払待ち一覧 → 該当レポートを支払完了 | レポートが paid 状態になる |
| 6 | 添付ダウンロード（署名付き URL）を Member 側で確認 | 添付ファイルがダウンロードできる |

**seed 投入アカウント一覧（テナント A：`test-admin/approver/member/accounting/approver2@example.com`、テナント B：`test-member-b/member-empty/approver-b@example.com`、全 8 アカウント、全て `TestPass1!`、出典: `internal/seed/seed.go:171-182`）**

##### 6.3.3 確認観点（11-E 完了条件）

- [ ] 第三者が公開 URL（`https://<cloudfront_domain_name>/`）にブラウザから到達でき、HTTPS で配信される
- [ ] `https://<cloudfront_domain_name>/health` が 200 を返す
- [ ] ALB DNS 直アクセス（`http://<alb_dns_name>/health`）が 403 を返す（B-1-b 完全閉域の確認）
- [ ] 申請 → 承認 → 支払のフローが画面操作で完走する

---

### 7. コスト見積り

#### 7.1 12ヶ月間（無料枠内）

| リソース | 想定月額 |
|---|---|
| EC2 t3.micro | $0（無料枠 750h/月） |
| EBS gp3 8GB | $0（無料枠 30GB） |
| RDS db.t3.micro Single-AZ + 20GB | $0（無料枠 750h + 20GB） |
| ALB | $0（無料枠 750h + 15GB） |
| S3 5GB 以下 | $0（無料枠） |
| データ転送 100GB 以下 | $0（無料枠） |
| DynamoDB（state lock） | $0（PAY_PER_REQUEST、ほぼ呼ばれない） |
| **合計** | **$0/月** |

#### 7.2 12ヶ月超過後（最大想定）

| リソース | 月額（USD） |
|---|---|
| EC2 t3.micro 24h × 30d | ~$8 |
| EBS gp3 8GB | ~$1 |
| RDS db.t3.micro Single-AZ | ~$15 |
| RDS Storage 20GB gp3 | ~$2.5 |
| RDS Backup 7d（DB容量分は無料、超過分のみ） | ~$0 |
| **ALB（最大支出ドライバ）** | **~$20**（無料枠 12ヶ月のみ。超過後も最大 750h/月課金が続き、destroy しない限り恒常 $20。§11 Q5 の判断根拠の中心） |
| S3 5-10GB | ~$0.5 |
| データ転送 | ~$1-5 |
| **合計** | **~$48/月** |

UAT 期間（1-2ヶ月）終了後は `terraform destroy` で全停止し、$0/月に戻すのを runbook 化する。

#### 7.3 ALB を使わない選択肢（参考）

- ALB を使わず EC2 に Elastic IP + Route53 + Caddy/Nginx で TLS 終端する案も技術的には可能（ALB 約 $20/月の節約）
- ただし MVP 期間は無料枠内、ポートフォリオ品質では ALB 構成（ADR 整合）を選ぶ
- §11 ユーザー判断事項として記載

---

### 8. リスクと緩和策

| # | リスク | 影響度 | 緩和策 |
|---|---|---|---|
| 1 | 無料枠誤超過（特に NAT Gateway / Elastic IP / Multi-AZ） | 高 | NAT Gateway を絶対に作らない、EIP を未アタッチで残さない、Budget アラート $5/月、`terraform plan` を必ず確認 |
| 2 | Terraform state 破損 | 中 | **state バケット（`expense-saas-tfstate-*`）は §5.3 で versioning Enabled で作成**（領収書バケットの versioning とは別物。F-117 で領収書側を無効化したことと混同しないこと）、DynamoDB lock、`terraform.tfstate` をコミットしない（.gitignore） |
| 3 | RDS db.t3.micro メモリ不足（1GB） | 中 | UAT 程度の同時接続数なら問題なし。pgxpool の MaxConns を絞る |
| 4 | EC2 t3.micro メモリ不足（1GB）で docker build OOM | 中 | swap 1-2GB を user_data で確保、または GHCR で配布（案A） |
| 5 | デプロイ失敗時のロールバック | 中 | RDS スナップショットを apply 前後で取得、`terraform destroy` 手順を §5.8 に明記 |
| 6 | IAM 権限過多（Terraform 用 AdministratorAccess） | 低（ポートフォリオなら受容） | アクセスキーを使い終わったら無効化。本番運用想定では権限を絞る |
| 7 | 公開 URL のセキュリティ（パスワード総当たり等） | 中 | レート制限はデフォルト値（本番）で運用、`UNAUTH_RATE_LIMIT_PER_MINUTE` を未設定にする |
| 8 | JWT 鍵の漏洩 | 高 | EC2 上 `/etc/expense-saas/keys/` を root:600、ssh 鍵をローカル `~/.ssh/`、tfvars をコミットしない |
| 9 | RDS public access | 高 | `publicly_accessible = false` を明示、SG で EC2 SG のみ許可 |
| 10 | S3 バケット public 化 | 高 | `aws_s3_bucket_public_access_block` で全ブロック、署名付き URL のみ許可 |
| 11 | HTTPS 化忘れ / 案1 採用時の中間者攻撃リスク | 中 | §11 Q2 案1（HTTP のみ）採用時は ADR-0004 / env_config.md prod と乖離するため UAT を社内 / 自分一人 / 親しい開発者レビューに限定。第三者（採用面接評価者等）に見せる場合は案2（Route53 + ACM）を採る（B-02 対応） |
| 12 | 実環境時刻ドリフト | 低 | EC2 は AL2023 デフォルトで chrony が動く。RDS も AWS 管理。Phase 7 で `timedatectl` 確認 |
| 13 | `expense_app` ロールの再現漏れ | 中 | RLS ポリシーは `migrate up` で完全に反映される（B-01 検証済み、§5.7.1）。残るリスクは `expense_app` ロール作成漏れのみ。§5.7.3 の psql ヒアドキュメントで担保し、Phase 4-4 タスクで完了確認 |
| 14 | **migration `000002_create_roles.up.sql` の `localdev` パスワードが prod に残置 + テーブル owner 取り違え + `schema_migrations` 移管漏れ** | 高 | (a) `expense_owner` / `expense_app` の両ロールは migration により `PASSWORD 'localdev'` で作成されるため、**§5.7.3 Step 2 で RDS マスターユーザーから `ALTER ROLE ... WITH PASSWORD '${...}'` を必ず実行**して prod 値で上書きすること。(b) **`migrate up` を全段マスターで実行するとテーブル owner がマスターになる**（前回 commit d714373 の副作用）。`expense_owner` で接続するアプリ・seed が ALTER TABLE / ロック取得等で権限不足を起こすため、§5.7.3 では **000001/000002 のみマスター、000003 以降は `expense_owner`** で実行する c 案を採用。(c) c 案の Step 1 で `schema_migrations` がマスター owner として作られるため、**Step 2 で `ALTER TABLE schema_migrations OWNER TO expense_owner` を実行しないと Step 4 の migrate 継続が `permission denied` で失敗する**（F-115 残課題、再々修正で追加）。Step 4 の `pg_tables.tableowner` 確認 SQL（全行 `expense_owner`、`schema_migrations` 含む）と Step 6 の両ロール疎通確認 `SELECT 1` + `expense_app` での実テーブル `SELECT count(*)` を完了ゲートとする。`IF NOT EXISTS` 付き `CREATE ROLE` は NOOP になるため `CREATE ROLE` 文は runbook から削除済み（F-115 + 再修正 + 再々修正） |
| 15 | Q1 案ごとの runbook 不整合（image 名・migration SQL 持ち込み） | 中 | §5.5.1 / §5.5.2 / §5.6 / §5.7.4 で 3 案を分岐し、終点（image 名 `expense-saas:portfolio`、`/health` 200、seed、手動スモーク）を統一。Phase 1 着手前に Q1 を確定する（F-116 対応） |
| 16 | **`ALTER DEFAULT PRIVILEGES` の実行ロール限定挙動による `expense_app` の本番テーブル DML 権限欠落（F-118）** | 高 | `000002_create_roles.up.sql` の `ALTER DEFAULT PRIVILEGES IN SCHEMA public ... TO expense_app` は **「実行したロールが今後作成するテーブル」にのみ効く** PostgreSQL 仕様。c 案では 000002 がマスター実行のため、000003 以降を `expense_owner` が作成するシナリオでは `expense_app` への自動付与が成立しない。結果として `APP_DATABASE_URL` で接続するアプリの通常 API リポジトリは `permission denied for table tenants` で全失敗する。**緩和策（二重ガード）**: §5.7.3 Step 3 で `expense_owner` 接続から `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO expense_app` を仕込み、Step 5 で既存テーブル全件への保険 `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO expense_app` を投入する。Step 6 では `has_table_privilege()` で 9 テーブル × 4 権限のメタデータ検証 + RLS 非対象 `users` への live `SELECT count(*)` を完了ゲートとし、`SELECT 1` だけでは検出できない権限不足を実地検証する（F-119 対応で RLS 対象テーブルへの直接 SELECT は使わない）。シーケンスは MVP では未使用（全テーブル UUID 主キー、全件 grep 確認済み）だが、将来導入時の保険として `USAGE ON SEQUENCES` も併せて投入（実行時は対象 0 件、害なし）（F-118 対応） |

---

### 9. 作業順序（タスクリスト）と担当割り

#### 9.1 Phase 0: 事前確認（並列可）

| # | タスク | 担当 | 並列 |
|---|---|---|---|
| 0-1 | ~~PR #144（issue #175 deploy.yml 整合）レビュー & merge~~ **PR #144 マージ済み（commit `6c99001`、2026-05-08）** — 基盤反映済みのため対応不要。確認のみ | 指揮役 | 完了 |
| 0-2 | 11-D CONDITIONAL #1（Go 統合テスト再実行） | 指揮役（`/test`） | **完了（2026-05-13、§実施ログ - Phase 0 詳細参照）** |
| 0-3 | 11-D CONDITIONAL #2（`npm run e2e` 10件 PASS、ローカル） | 指揮役 | **完了（2026-05-13、10/10 PASS 26.7s）** |
| 0-4 | 11-D CONDITIONAL #4（frontend クリーンビルド） | 指揮役 | **完了（2026-05-13、3.86s ビルド成功）** |
| 0-5 | AWS アカウント / 支払い情報 / Budget アラート $5 設定 | USER | 並列2 |
| 0-6 | IAM ユーザー `terraform-deployer` 作成 + アクセスキー | USER | 並列2 |
| 0-7 | キーペア `expense-saas-portfolio` 作成 | USER | 並列2 |
| 0-8 | ローカル Terraform >= 1.6 インストール | USER | 並列2 |
| 0-9 | `scripts/init-db.sh` 確認、prod 版マイグレーション手順確定（§5.7） | 指揮役（read-only） | 並列1 |

#### 9.2 Phase 1: Terraform IaC 作成（サブエージェント委譲可）

| # | タスク | 担当 | 出力 | 状態 |
|---|---|---|---|---|
| 1-1 | `expense-saas/infra/terraform/` 配下のファイル骨子作成 | platform-builder | §3.1 のディレクトリ構成 | **完了（PR #145 / `f3381cd`、2026-05-13）** |
| 1-2 | network.tf / security_groups.tf | platform-builder | VPC + 4 subnet + 3 SG | **完了** |
| 1-3 | ec2.tf / iam.tf | platform-builder | EC2 + instance profile + S3 IAM | **完了** |
| 1-4 | rds.tf | platform-builder | DB subnet group + RDS instance | **完了** |
| 1-5 | s3.tf / alb.tf | platform-builder | 領収書バケット + ALB + target group | **完了** |
| 1-6 | variables.tf / outputs.tf / terraform.tfvars.example | platform-builder | 入出力定義 | **完了** |
| 1-7 | `terraform fmt -recursive` の検証 | platform-builder | エラー 0 | **完了（reviewer + codex 3 ラウンドで PASS）** |
| 1-8 | README.md（apply 手順、本書 §5 を抜粋） | platform-builder | runbook | **完了（DB bootstrap §5.7.3 c 案 Step 1〜7 詳細含む）** |

サブエージェントは `terraform plan` 等の AWS 認証コマンドは実行しない（feedback_subagent_permissions.md）。**Phase 1 の検証ゲートは `terraform fmt -recursive` のみ**：

- `terraform init` / `terraform validate` は provider DL のため `registry.terraform.io` / `releases.hashicorp.com` への egress が必要だが、devcontainer の egress allowlist には未許可（issue #060/#061 の最小 egress 方針に従う）
- 結果として lock ファイル（`.terraform.lock.hcl`）の生成・コミットは USER の Windows ホストで Terraform CLI install 後（Phase 0-8 完了後）に実施する
- `terraform validate` 相当の構造チェックは Phase 2-3 の `terraform plan` で実質的に行われるため、Phase 1 では構文（fmt）と HCL ファイル責務カバレッジ（reviewer による）でゲートする
- サブエージェントは事前承認なくツール（Terraform / OpenTofu 等）のインストールを行わない（memory `feedback_no_silent_tool_install.md` 参照）。devcontainer に install された Terraform CLI は `.devcontainer/Dockerfile` で正規にビルド時 install される

#### 9.3 Phase 2-7: AWS apply 〜 検証（USER 主導）

| # | タスク | 担当 | 備考 |
|---|---|---|---|
| 2-1 | state バケット + DynamoDB lock 作成（§5.3） | USER | 一回限り |
| 2-2 | `terraform init` | USER | |
| 3-1 | `terraform plan -out=tfplan` | USER | 指揮役が出力をレビュー |
| 3-2 | `terraform apply tfplan` | USER | 10-15 分 |
| 3-3 | outputs 取得（ALB DNS、RDS endpoint、S3 bucket） | USER | 指揮役に共有 |
| 4-1 | EC2 ssh 疎通確認 | USER | |
| 4-2 | RDS 疎通確認（EC2 から psql） | USER | |
| 4-3 | migrate up 実行（§5.7） | USER + 指揮役支援 | |
| 4-4 | expense_app ロール / RLS 投入 | USER + 指揮役支援 | |
| 5-1 | Docker イメージ配布（§3.4 で選択した方式） | USER | |
| 6-1 | systemd unit 配置 + 起動 | USER | |
| 6-2 | `docker logs` でアプリ起動成功確認 | USER + 指揮役 | |
| 7-1 | CloudFront 経由 `/health` 200 OK ＋ ALB DNS 直アクセス 403 確認 | 指揮役 | curl（§6.1） |
| 7-2 | 11-D CONDITIONAL #3 時刻ドリフト smoke | USER + 指揮役 | `timedatectl`、`SELECT NOW()` |
| 7-3 | seed 投入 | USER | |
| 7-4 | 手動スモーク（申請→承認→支払） | USER（操作者） + 指揮役（観察・記録） | §6.3 |
| 7-5 | 11-E チケット §「実施ログ」記録 | 指揮役 | |

#### 9.4 Phase 9: 11-F 引き継ぎ準備（指揮役）

| # | タスク | 担当 |
|---|---|---|
| 9-1 | 公開 URL / UAT アカウント / デプロイ日時 / コミット SHA を 11-E チケットに記録 | 指揮役 |
| 9-2 | 既知非ブロッカー（11-D N-01〜N-05、post-MVP issue 一覧）を引き継ぎリストに整理 | 指揮役 |
| 9-3 | 11-F でユーザーが確認すべき重点箇所を列挙（11-D §7 ベース） | 指揮役 |

#### 9.5 並列化可能な箇所

- Phase 0 の指揮役タスク（0-1, 0-2, 0-3, 0-4, 0-9）と USER の AWS 準備（0-5〜0-8）は完全並列
- Phase 1 内では 1-2 と 1-4（network と RDS は依存少ない）を別エージェントで並列も可能だが、ファイル数が少ないので 1 エージェントで直列の方がコスト効率は良い

---

### 10. 完了判定（11-E チケット完了条件との対応）

> ※TLS / 公開 URL は ADR-0007（CloudFront 前段化）を正とする。公開 URL はエンドユーザー向けの CloudFront ドメイン（`https://<cloudfront_domain_name>/`）であり、ALB DNS への直アクセスは B-1-b により遮断される。`<cloudfront_domain_name>` / `<alb_dns_name>` は apply 後に確定する `terraform output` 名であり、ドキュメント上はプレースホルダ表記とする。

| 11-E 完了条件 | 本書での確認手段 |
|---|---|
| デプロイ済み環境で第三者がアクセスできる | §6.3.2 手順1 で `https://<cloudfront_domain_name>/` 経由でログイン画面に HTTPS 到達できる（CloudFront ドメインが公開 URL） |
| ヘルスチェックエンドポイントが応答する | §6.1 `curl -i https://<cloudfront_domain_name>/health` が HTTP/2 200 ＋ body `{"status":"ok"}` を返す |
| ALB DNS 直アクセスが遮断されている（B-1-b 完全閉域の確認） | §6.1 `curl -i http://<alb_dns_name>/health` が `403 Forbidden` を返す（カスタムヘッダ `X-Origin-Verify` 不在のため ALB リスナーのデフォルトアクションで遮断。200 が返る場合は閉域化が機能していない＝未完了） |
| 主要フロー1本（申請→承認→支払）が通る | §6.3.2 手動スモーク 6 ステップ全て期待結果通り |
| 11-F 引き継ぎ表が 11-E チケット §「実施ログ」に記録済み | 公開 URL（CloudFront ドメイン `https://<cloudfront_domain_name>/`） / UAT アカウント（メール + パスワード `TestPass1!`） / デプロイ日時 / コミット SHA / 既知非ブロッカー（11-D N-01〜N-05 + ADR-0007 で受容した逸脱 B-3 / B-5 + Secrets Manager 不採用）が全て埋まっていること（I-01 対応） |

加えて、11-D CONDITIONAL の 4 条件全てを Phase 0 + Phase 7 で消化済みであること。

---

### 11. ユーザー判断が必要な事項

以下は本計画で「方針提示のみ・最終決定は USER」とした事項。**2026-05-08 にユーザー判断完了、全件 architect 推奨案で確定済み**。

| # | 判断事項 | 選択肢 | 推奨 | **確定（2026-05-08）** |
|---|---|---|---|---|
| Q1 | Docker イメージ配布方式 | 案A: GHCR / 案B: EC2 上 build / 案C: ECR | 初回は案B、運用化で案A（§13 チェックリスト「Phase 1 着手前に Q1 確定」必須） | **案 B: EC2 上で docker build**（image_tag 変数は default = "" の optional として運用） |
| Q2 | TLS / カスタムドメイン（Q7 を統合） | 後述 3 案 | 採用面接の評価者等に見せないなら案1。見せるなら案2 | **【現行】ADR-0007（CloudFront 前段化で HTTPS 化）が正本**。当初は案1（HTTP のみ、ALB DNS）採用だったが、issue #185 / ADR-0007（2026-05-20）で C 案へ方針変更。詳細は後述「Q2 詳細」参照 |
| Q3 | Multi-AZ | Single-AZ（無料枠） / Multi-AZ（有料） | Single-AZ（ADR-0004 で受容） | **Single-AZ** |
| Q4 | Terraform 用 IAM 権限 | AdministratorAccess（簡易） / 必要権限のみ（厳密） | AdministratorAccess（ポートフォリオ簡略化） | **AdministratorAccess**（IAM ユーザー使い切り後に削除） |
| Q5 | UAT 後の停止運用 | 維持 / `terraform destroy` で停止 | 13ヶ月目以降は destroy。**ALB が最大支出ドライバ（$20/月恒常）** のため UAT 期間明けは destroy 必須 | **13 ヶ月目以降に terraform destroy** |
| Q6 | deploy.yml の if: false 解除 | 11-E で実施 / 11-E 完了後に別チケット | 別チケット（11-E スコープ外） | **11-E スコープ外、別チケット**（参考デモとしてリポジトリに残置） |
| ~~Q7~~ | （Q2 に統合済み） | - | - | （Q2 で確定済み） |

#### Q1 案 B 採用に伴う追加準備

- §5.7.4 で **案 B 用の migration SQL 持ち込み手段（git clone 経由）** を採用
- §3.3 で **`image_tag` 変数を default = "" の optional** として宣言、user_data で `docker pull` をスキップする分岐を入れる
- platform-builder への Phase 1 指示は「Q1 案 B 確定済み」を前提に進められる

##### Q2 詳細: TLS / カスタムドメイン

このセクションは「現行構成（ADR-0007 を正本とする確定仕様）」と「判断履歴（案1 採用当時の記録、参照のみ）」の 2 ブロックに分かれる。**11-E の手順・完了判定・11-F 引き継ぎは下記【現行構成】に従うこと。**【判断履歴】は当時の経緯を残すためのもので、現行の作業根拠としては使わない。

###### 【現行構成】ADR-0007（CloudFront 前段化で HTTPS 化）— 2026-05-20 確定・正本

issue #185 / ADR-0007（`dev-journal/deliverables/docs/30_arch/adr/0007-cloudfront-https.md`）により、当初の案1（HTTP のみ、ALB DNS 直利用）から **C 案（ALB 前段に CloudFront を配置し `*.cloudfront.net` デフォルト証明書で HTTPS 化）** へ方針変更した。11-E の TLS 構成・公開 URL・完了判定は本ブロックと ADR-0007 を正とする。

- **リクエスト経路**: `Browser → CloudFront → ALB → EC2（Go API + SPA）`
- **公開 URL**: エンドユーザーは CloudFront ドメイン（`https://<cloudfront_domain_name>/`）経由でのみ到達する。`<cloudfront_domain_name>` は apply 後に確定する `terraform output`（`expense-saas/infra/terraform/outputs.tf`）名であり、ドキュメント上はプレースホルダ表記とする。
- **ALB 直アクセス**: B-1-b（カスタムヘッダ `X-Origin-Verify` 検証 ＋ SG プレフィックスリスト限定）により ALB DNS への直接 HTTP アクセスは遮断され `403` を返す。これは閉域化が機能している証跡であり、完了判定（§10）・ヘルスチェック手順（§6.1）の独立項目として確認する。
- **CORS**: prod の `CORS_ALLOWED_ORIGINS` は CloudFront ドメイン（`https://<cloudfront_domain_name>`）。`env_config.md` §3.1 / §4.5 を正とする。
- **受容する逸脱**: ADR-0007 B-3（CloudFront〜ALB 間 HTTP）/ B-5（viewer 側 TLS 最小バージョン TLSv1 固定）を ADR-0007 に「受容する逸脱」として記録済み。`security.md` §11 / §441 本文は不変。11-F 引き継ぎの「既知差分」は本 B-3 / B-5 を参照する（旧案1 の HTTP 運用差分ではない）。
- **クライアント IP**: CloudFront 追加でプロキシ段が 2 段になるため `TRUSTED_PROXY_COUNT`（prod=2 / dev=0）を導入（issue #185 T1）。

###### 【判断履歴】案1（HTTP のみ、ALB DNS 直利用）採用当時の記録 — 参照のみ・現行作業の根拠にしない

> 以下は ADR-0007 による方針変更**前**の判断記録である。**現行の手順・完了判定は上記【現行構成】が正本**であり、本ブロックの「案1 採用」「ALB DNS 直利用」「HTTP 運用」前提は現行作業に適用しない。経緯把握のために残す。

当時、`security.md` §11 / `env_config.md` §3.1 prod が定める prod の TLS（ACM 終端）・CORS（`https://...` 固定）に対し、本計画は差分を許容する代わりに選定理由と差分を 11-F 引き継ぎに記録する方針だった。3 案を以下のとおり比較し、案1 を採用していた。

| 案 | 構成 | 月額追加コスト | メリット | デメリット |
|---|---|---|---|---|
| **案1（当時採用, ポートフォリオ簡易）** | HTTP のみ。ALB DNS をそのまま使う | $0 | 設定がない（ACM 不要）/ ALB DNS でアクセス可能 | ADR-0004 / env_config.md prod の TLS 要件と乖離 / JWT 平文送受信のため中間者リスクあり / 採用面接の評価者等に見せると印象悪 |
| **案2（当時の代替案, セキュリティ整合）** | Route53 ドメイン取得（年 $12）+ ホストゾーン（$0.5/月）+ ACM（無料、DNS 検証）+ ALB 443 リスナー | 年 $12 + 月 $0.5（destroy しても Route53 ドメインは年契約継続） | ADR-0004 整合 / HTTPS で JWT 安全 / UAT を社外に見せられる | 初期セットアップ +30 分（ドメイン取得 + ホストゾーン作成 + ACM 検証 CNAME 投入 + ALB リスナー差し替え） |
| **案3（不採用）** | nip.io / sslip.io 等の wildcard DNS + ACM | $0 | ドメイン費不要 | ACM は wildcard DNS の TXT 検証を通せない（ドメイン検証ポリシー上不可）/ Let's Encrypt + EC2 終端なら可能だが ALB 構成と両立しない |

案1 採用の分岐基準は「UAT を社内デモ / 自分一人 / 親しい開発者レビュー限定で済ませる場合」だった。その後、本プロジェクトは「ポートフォリオ兼デモ」で第三者に URL を共有する前提のため、issue #185 で HTTPS 化が必要と判断され、案2 のコスト（独自ドメイン年契約）を避けつつ HTTPS を達成する C 案（CloudFront）が ADR-0007 で採用された。これにより案1 / 案2 はいずれも現行の採用案ではなくなった。

> なお、案1 採用当時に存在した整合記述「Q2 案 1 採用に伴う §4.2 / §3.2 / §7.2 / リスク表の整合」（`CORS_ALLOWED_ORIGINS=http://<alb-dns>` 採用、alb.tf 80 のみ、リスク表 #11 を HTTP 運用受容として記録 等）は ADR-0007 への方針変更に伴い現行の根拠ではなくなったため削除した。CORS・TLS・公開 URL の現行整合は上記【現行構成】および ADR-0007・`env_config.md` を正とする。なお §3.2 alb.tf・§4.2 `CORS_ALLOWED_ORIGINS` 等の他セクションには案1（HTTP / ALB DNS 直）当時の記述が残存しており（finding 121 §注記の他ファイル波及確認の対象外＝同一ファイル内）、本 finding のスコープ（§6 / §10 / §11 Q2 / 11-F 引き継ぎ）には含めていない。これらの現行整合は ADR-0007 反映時の別タスク（issue #185 T5 / T4）に委ねる。

---

### 12. 工数見積り

| カテゴリ | 内訳 | 想定時間 |
|---|---|---|
| サブエージェント時間（platform-builder） | Phase 1: Terraform 雛形作成・静的検証 | 1.5-2.5h |
| 指揮役時間（Phase 0 事前消化） | Go 統合テスト / e2e ローカル / FE クリーンビルド / scripts 確認 / PR #144 レビュー | 1.5-2.5h |
| 指揮役時間（Phase 3-7 支援） | plan レビュー、ログ確認、スモーク観察、チケット記録 | 1.5-2h |
| ユーザー手動時間（Phase 0 AWS 準備） | アカウント / IAM / キーペア / Terraform install | 0.5-1h |
| ユーザー手動時間（Phase 2-3 apply） | state 作成 / init / plan / apply | 0.5-1h（apply 待ちは 10-15 分） |
| ユーザー手動時間（Phase 4-6 デプロイ） | ssh / migrate / role 作成 / Docker / systemd / 起動確認 | 1.5-2.5h |
| ユーザー手動時間（Phase 7 スモーク） | seed / 手動 6 ステップ / 不具合切り分け | 0.5-1.5h |
| バッファ（不具合切り分け・リトライ） | apply 失敗 / マイグレ失敗 / Docker run 失敗 等 | 1-3h |
| **合計** | | **約 8-16h** |

#### 12.1 クリティカルパス

```
[CP] Phase 1 Terraform 雛形 (1.5-2.5h)
   -> Phase 2 state init (0.2h)
   -> Phase 3 apply (10-15 分の待ち + 監視)
   -> Phase 4 migrate (0.3-0.5h)
   -> Phase 5-6 Docker 配布 + 起動 (0.5-1.5h、案B build なら +0.5h)
   -> Phase 7 スモーク (0.5-1.5h)
```

ボトルネック:

1. **Phase 3 apply の待ち時間（特に RDS 作成 10 分前後）** — 並行作業不可、ただし長時間ブロックではない
2. **Phase 5 案B（EC2 build）の場合の docker build** — t3.micro メモリ 1GB で OOM リスク。swap 設定が必要
3. **Phase 7 不具合切り分け** — RDS 接続失敗 / RLS 設定漏れ / SG 設定漏れ等が起きると一気に+2-3h

---

### 13. このあと指揮役が確認すべき事項（チェックリスト）

- [ ] **§11 Q1（Docker イメージ配布方式）を Phase 1 着手前に確定**（未確定のまま platform-builder に着手させる場合は §3.3 image_tag 変数を default = "" の optional として渡す。W-01 対応）
- [ ] §11 Q2（TLS / カスタムドメイン、Q7 統合）の判断をユーザーに確認 — 案1 / 案2 のどちらを採るか（B-02 対応）
- [ ] §11 Q3〜Q6 の判断をユーザーに確認
- [x] ~~PR #144（issue #175）のステータス確認、Phase 0 と並行で merge する~~ **PR #144 はマージ済み（commit `6c99001`、2026-05-08）。基盤反映済み**（I-03 対応）
- [x] ~~`expense-saas/scripts/init-db.sh` を読み、prod 用マイグレーション + role 作成手順を §5.7 にフィードバック~~ **本計画 §5.7.1 / 5.7.2 / 5.7.3 で確定済み**（B-01 対応）
- [ ] **Dockerfile の seed エントリポイント検証は §6.3.1 で完了（`/app/seed` でビルド済み確認）**。Phase 0 で再確認する場合は `docker run --entrypoint /app/seed expense-saas:portfolio --help` を素振り（W-04 対応）
- [ ] 11-E チケットを Phase 別の小チケット（11-E-1 Terraform 雛形 / 11-E-2 AWS apply / 11-E-3 デプロイ + 検証）に分割するかを判断
- [ ] Phase 0 の 11-D CONDITIONAL 事前消化を即時着手
- [x] **F-115 対応（再々修正 2026-05-08）**: `000002_create_roles.up.sql` の localdev パスワード残置 + commit d714373 のテーブル owner 取り違え + commit 21521f2 で残った `schema_migrations` 移管漏れを §5.7.1 / §5.7.3 / §8 #14 / §4.2 末尾差分表に反映済み。**§5.7.3 c 案（000001/000002 はマスター → ALTER ROLE + `ALTER TABLE schema_migrations OWNER TO expense_owner` → `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner` → 000003 以降は `expense_owner` → `GRANT ON ALL TABLES`）** を採用し、Phase 4-4 で **Step 2 ALTER ROLE + schema_migrations 移管 / Step 3 default privileges / Step 4 owner 確認 SQL（全テーブル expense_owner、schema_migrations 含む）/ Step 5 既存テーブル一括 GRANT / Step 6 両ロール SELECT 1 + `has_table_privilege()` メタデータ検証 + `users` への live SELECT** の 5 つを完了ゲートとする
- [x] **F-118 対応（新規 blocker、2026-05-08）**: `ALTER DEFAULT PRIVILEGES` の実行ロール限定挙動により、c 案では 000002 のマスター実行版 default privileges が本番 9 テーブルへの `expense_app` DML を自動付与できないことを §5.7.1 / §5.7.3 Step 3 + Step 5 / §8 #16 / §4.2 末尾差分表に反映済み。**Step 3（default privileges、`expense_owner` 接続）+ Step 5（既存テーブル一括 GRANT、保険）の二重ガード**を採用。シーケンス使用の有無は **000003〜000011 全件 grep で UUID 主キーのみ・SERIAL/IDENTITY/sequence 不在を確認**（§5.7.1）。`USAGE ON SEQUENCES` は将来用の保険として runbook に含む（実行時は対象 0 件）。完了ゲートは §5.7.3 Step 6 の `has_table_privilege()` 9 テーブル × 4 権限メタデータ検証 + `users` live SELECT
- [x] **F-119 対応（2026-05-08）**: Step 6 の `expense_app` 検証 SQL が RLS 対象テーブル（`tenants` / `expense_reports`）への直接 SELECT で `app.current_tenant` 未設定のため false negative になる問題。**`has_table_privilege()` メタデータ検証 + RLS 非対象 `users` への live SELECT** の 2 段階に再構成。RLS 対象テーブルへの直接 SELECT は使わず、本物のテナント分離検証は Phase 7 スモーク（実 API 経由）に委ねる方針を §5.7.3 Step 6 / §4.2 表 / §8 #16 / §13 に反映済み
- [x] **F-116 対応**: §5.5.1 / §5.5.2 / §5.6 / §5.7.4 で Q1 3 案ごとに image 取得 / retag / migration SQL 持ち込みを分岐済み。Dockerfile に `db/migrations/` は同梱されていないことを確認済み（image 名は `expense-saas:portfolio` で 3 案統一）
- [x] **F-117 対応**: 領収書バケット S3 versioning は **a 案（上流追従、無効）** を採用。§3.2 s3.tf 行 / §4.2 末尾差分表に反映。b 案（versioning 有効）を後から採るための条件は §4.2 注で明示

---

### 14. 参考リンク

- 11-E チケット: `dev-journal/progress-management/tickets/step11/11-E-deploy.md`
- 11-D 再レビューレポート: `dev-journal/progress-management/step11-d-review-report.md`
- ADR-0004（インフラ選定 + ポートフォリオ対応）: `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md`
- リリース手順: `dev-journal/deliverables/docs/70_operations/release.md`
- 環境設定: `dev-journal/deliverables/docs/70_operations/env_config.md`
- アーキテクチャ: `dev-journal/deliverables/docs/30_arch/architecture.md`
- deploy.yml: `expense-saas/.github/workflows/deploy.yml`
- issue #175: `dev-journal/issues/open/175-deploy-yml-e2e-job-misalignment.md`
- seed 実装（既定パスワード `TestPass1!` の出典、W-05 対応）:
  - `expense-saas/cmd/seed/main.go`（エントリポイント、`main.go:89` に password "TestPass1!"）
  - `expense-saas/internal/seed/seed.go`（実装、`seed.go:108` に `hasher.HashPassword("TestPass1!")`）
- Dockerfile（W-04 検証根拠）: `expense-saas/Dockerfile`
- migrations の RLS 定義（B-01 検証根拠）: `expense-saas/db/migrations/00000{3,5,6,7,8,9}_*.up.sql`
- localdev 用 init-db.sh（B-01 比較根拠）: `expense-saas/scripts/init-db.sh`

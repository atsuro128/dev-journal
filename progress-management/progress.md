# プロジェクト進捗管理

## マイルストーン

| # | マイルストーン | 状態 | 完了日 | ガイド |
|---|--------------|------|--------|--------|
| 0 | 事前準備（プロジェクトの土台づくり） | 完了 | 2026-03-04 | `ai-dev-framework/guide/work-breakdown/step0-preparation/` |
| 1 | 要件定義（業務理解 → ユースケース化） | 完了 | 2026-03-09 | `ai-dev-framework/guide/work-breakdown/step1-requirements/` |
| 2 | ドメイン設計（データとルールの核） | 完了 | 2026-03-14 | `ai-dev-framework/guide/work-breakdown/step2-domain/` |
| 3 | アーキテクチャ設計（技術選定・構成決定） | 完了 | 2026-03-16 | `ai-dev-framework/guide/work-breakdown/step3-architecture/` |
| 4 | 基本設計（画面一覧・画面遷移） | 完了 | 2026-03-22 | `ai-dev-framework/guide/work-breakdown/step4-basic-design/` |
| 5 | 詳細設計（API・DB・認可・セキュリティ） | 完了 | 2026-03-23 | `ai-dev-framework/guide/work-breakdown/step5-detail-design/` |
| 5.5 | UI コンポーネント設計 | 完了 | 2026-04-06 | `ai-dev-framework/guide/work-breakdown/step5.5-ui-component/` |
| 6 | テスト設計 | 完了 | 2026-04-12 | `ai-dev-framework/guide/work-breakdown/step6-testing/` |
| 7 | 運用設計 | 完了 | 2026-03-26 | `ai-dev-framework/guide/work-breakdown/step7-operations/` |
| 8 | 基盤構築 | 完了 | 2026-03-31（初回完了）/ 2026-04-12（8-11 追加・再完了） | `ai-dev-framework/guide/work-breakdown/step8-foundation/` |
| 9 | テストコード実装 | 完了 | 2026-04-08 | `ai-dev-framework/guide/work-breakdown/step9-test-implementation/` |
| 10 | 機能実装 | 完了 | 2026-04-11 | `ai-dev-framework/guide/work-breakdown/step10-feature-implementation/` |
| 11 | システムテスト・UAT | 進行中 | - | `ai-dev-framework/guide/work-breakdown/step11-system-test/` |

## タスク状態定義

| 状態 | 意味 | 遷移条件 |
|------|------|----------|
| 未着手 | 未着手 | — |
| 作業中 | サブエージェントが作業中 | 依存先が全て `完了` |
| レビュー待ち | 成果物完成、レビュー未実施 | サブエージェント作業完了 |
| 修正中 | レビュー指摘を対応中 | レビューで FIX 判定 |
| 完了 | レビュー LGTM 済み | 品質ゲート PASS |

## 完了 Step のチケット一覧

Step 5.5 / 6（追加）/ 8 / 9 / 10 のチケット一覧はアーカイブ済み（`archives/progress/steps.md` 参照）。

## Step 11: システムテスト・UAT — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 11-A | ローカル動作確認 | ユーザー + 指揮役 | Step 10 全完了、Step 8-11 完了 | 完了（2026-05-05: 全 62 SMK PASS（FAIL 0）、副次 #170 起票） | `tickets/step11/11-A-local-verification.md` |
| 11-B | 横断テスト（Go） | test-implementer | 11-A | 完了（2026-05-06: PR #139 マージ済み、cross_cutting_test.go + rate_limit_test.go 1363 行追加、副次 #172 起票・解消） | `tickets/step11/11-B-cross-cutting-test.md` |
| 11-C | E2E テスト（Playwright） | test-implementer | 11-A | 完了（2026-05-07: PR #138 マージ済み、E2E 10/10 PASS。CRS-066 真因 = Docker クロックドリフト 18 秒巻戻りで JWT iat 未来扱い 401 → JWT leeway 60s 追加 PR #141 で解消、副次 #173 起票・解消） | `tickets/step11/11-C-e2e-test.md` |
| 11-D | 横断レビュー | reviewer (codex) | 11-B, 11-C | 完了（2026-05-07: codex 11-D 横断レビュー CONDITIONAL PASS。初回 FAIL のブロッカー B-01 (issue #173 対応漏れ middleware 経路) + M-03 (rate limit テストケース乖離) + M-04 (npm run e2e config 未指定) を全て解消し再レビュー PASS。CONDITIONAL の 4 条件は 11-E 進入条件として `tickets/step11/11-D-cross-review.md` §実施結果 §6 に明記、2026-05-08 に独立ファイル `step11-d-review-report.md` から統合） | `tickets/step11/11-D-cross-review.md` |
| 11-E | デプロイ・スモークテスト | platform-builder | 11-D | 作業中（2026-05-13: Phase 0 + Phase 1 完了 / 2026-05-18-19: Phase 2 + Phase 3 完了、26 リソース AWS apply 済み、ALB DNS `expense-saas-portfolio-alb-290554356.ap-northeast-1.elb.amazonaws.com` 取得、副次 #180 / #181 起票 / 2026-05-20: Phase 4 完了、DB bootstrap 全 Step 1-7 PASS、F-115/F-118/F-119 完了ゲート全通過、app.env 確定値書き込み済み / 2026-05-20: Phase 5（旧 backend-only イメージ）+ Phase 6 ヘルスチェック実施後、SPA 配信未実装の設計乖離（issue #183）が発覚 → PR #148 で go:embed 方式を実装・マージ。SPA embed 入り新イメージで Phase 5 を再実施中。Phase 5-7 USER 主導待ち） | `tickets/step11/11-E-deploy.md` |
| 11-F | UAT | ユーザー | 11-E | 未着手 | `tickets/step11/11-F-uat.md` |

## 課題・ブロッカー

### 解決済み（2026-04-12 〜 2026-04-16）

アーカイブ済み（`archives/progress/issues.md` 参照）。

### 残存 issue（Step 11 関連）
| ID | タイトル | 起票日 | 状態 |
|----|---------|-------|------|
| 133 | ログ・エラー出力の言語ポリシー整理（FE/BE 共通、日本語メッセージの棚卸しと削減） | 2026-04-21 | 別セッションで対応予定 |
| 145 | ReportWorkflowInfo セクション見出し追加（post-MVP） | 2026-04-24 | 起票のみ（post-MVP） |
| 146 | 大規模クロステナント環境での性能テスト計画（post-MVP） | 2026-04-25 | 起票のみ（post-MVP） |
| 151 | パスワードリセットのメール送信が未実装（要件 AUTH-F06 乖離、post-MVP） | 2026-04-28 | 起票のみ（post-MVP） |
| 165 | マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える | 2026-05-02 | resolved（PR #135 + #136 + #137 マージ済み 2026-05-06、レイアウト統一 + 改行位置調整 + デグレ修正） |
| 167 | DataGrid 列の自動幅 minWidth と手動リサイズ minWidth の分離 | 2026-05-03 | 起票のみ（post-MVP） |
| 170 | 「保存して続けて追加」ボタン押下で明細が 2 件登録される（データ整合性 blocker） | 2026-05-05 | resolved（PR #132 マージ済み 2026-05-05） |
| 171 | 認証フォームのメールアドレス欄に autocomplete 属性が欠落しており、ブラウザ・パスワードマネージャーがサジェスト候補を出さない | 2026-05-05 | resolved（PR #133 + PR #134 マージ済み 2026-05-05、Edge での手動確認 PASS） |
| 172 | ログインエンドポイントのレート制限値を環境変数で上書き可能にする（11-C E2E テストブロッカー） | 2026-05-06 | resolved（PR #140 マージ済み 2026-05-06、本番値 5/20 維持・dev/E2E 100/200 緩和） |
| 173 | JWT 検証にクロックスキュー許容（leeway）を追加（CRS-066 真因 = Docker クロックドリフト） | 2026-05-07 | resolved（PR #141 マージ済み 2026-05-07 で domain.JWTVerifier に leeway 適用、PR #143 マージ済み 2026-05-07 で pkg/jwt.Verifier (middleware 経路) にも leeway 適用、両系統を 11-D 再レビューで確認） |
| 174 | Step 11-C E2E テスト品質改善（CRS-074 アサーション強化 + カテゴリセレクタ data-testid 化） | 2026-05-07 | 起票のみ（post-MVP） |
| 175 | deploy.yml の e2e ジョブが Playwright プロジェクト構成と不整合（5 件、11-E 着手前修正） | 2026-05-08 | resolved（PR #144 マージ済み 2026-05-08、5 件不整合 + healthz→health bug の 2 コミットで対応、codex 再レビュー PASS） |
| 176 | frontend dev 依存の既知 CVE 棚卸し（esbuild + postcss、post-MVP） | 2026-05-13 | 起票のみ（post-MVP。Phase 0 着手時の npm ci で検出、本番影響なし、vitest 4.x 移行と合わせて対応予定） |
| 177 | frontend バンドルの chunk size 500 kB 超過警告（code splitting 未適用、post-MVP） | 2026-05-13 | 起票のみ（post-MVP。Phase 0 着手時の npm run build で検出、機能影響なし、code splitting + manualChunks で対応予定） |
| 178 | Go バージョン定義の環境間不整合（devcontainer / production build / test-be、post-MVP） | 2026-05-13 | 起票のみ（post-MVP。Phase 0 着手時に検出、現実害なし、test.Dockerfile 新規作成で 1.24.1 完全固定する予定） |
| 179 | devcontainer egress allowlist の github.com 全許可によるツール install リスク（issue #060 派生、post-MVP） | 2026-05-13 | 起票のみ（post-MVP。Phase 1 で platform-builder が OpenTofu を silent install した事案を契機に検出、memory + Dockerfile 正規 install で即時補完済み、allowlist 粒度化は post-MVP 検討） |
| 180 | backend.tf の dynamodb_table が Terraform 1.10+ で deprecated（use_lockfile に移行） | 2026-05-18 | 起票のみ（post-MVP。Phase 2 terraform init で deprecation warning 検出、動作支障なし、Step 11-F 完了後 or destroy 直前に整理予定） |
| 181 | RDS backup_retention_period が Free Tier 制約により NFR-AVAIL-003 から逸脱（7 日 → 1 日） | 2026-05-19 | resolved（PR #146 + PR #147 マージ済み 2026-05-19、設計修正 dev-journal f10f88a + 解決移動 d175c83 で NFR を portfolio 仕様の 1 日保持に緩和、案 3 採用、残置事項なし） |
| 182 | devcontainer egress allowlist に registry.terraform.io が含まれず terraform validate 実行不可（issue #179 系統） | 2026-05-19 | 起票のみ（post-MVP。devcontainer 内で terraform validate / plan / apply が `Forbidden` で失敗、fmt のみ利用可能。allowlist 追加 / provider 事前バンドル / fmt 限定明文化のいずれかで post-MVP 解決） |
| 183 | SPA embed 配信が未実装（architecture.md §4.0 と実装の乖離、Phase 7 ブロッカー） | 2026-05-20 | resolved（PR #148 マージ済み 2026-05-20、A 案 go:embed 方式で実装。Dockerfile に node build stage 追加 + cmd/server/embed.go + internal/spa パッケージ。内部レビュー 3 ラウンドで blocker 3 件解消 + codex レビュー PASS。新 Dockerfile はホストで docker build → image を EC2 に持ち込み 2026-05-20、Phase 5/6 やり直し完了・SPA 配信実地確認済み） |
| 184 | SPA fallback ハンドラが HEAD リクエストで 405 を返す（GET のみ登録） | 2026-05-20 | resolved（PR #150 マージ済み 2026-05-20、master dd480b5。SPA fallback を GET+HEAD 登録、HEAD /health はヘルスハンドラ経由化し index.html 誤返却を回避、回帰テスト 5 件追加。内部レビュー + codex PASS） |
| 185 | ALB が HTTP 平文で HTTPS 未対応（security.md §11 と乖離） | 2026-05-20 | 対応中（2026-05-20: architect 実装計画 v2 を reviewer 再レビュー PASS。設計判断 B-1-b 完全閉域 / B-2-c TRUSTED_PROXY_COUNT / B-3 受容 / B-4 追補ノート を確定、T1〜T6 + #184 に分解。**T1（remoteIP）PR #149 / #184（SPA HEAD）PR #150 / T3（CloudFront + B-1-b 完全閉域）PR #151 マージ済み（master 4812a01、各 内部レビュー + codex PASS）**。設計判断 B-5（CloudFront デフォルト証明書の viewer TLS 最小 TLSv1 を §11 受容逸脱として ADR-0007 記録）を追加。**T5（設計書整合・ADR-0007）完了**（内部レビュー + codex レビュー PASS、review-finding 121 対応・resolved、dev-journal commit 1925081 / 2155854。W-1 は副次 #186 起票）。残: T4（CORS/env 値反映）・T6（疎通検証）は実 AWS デプロイ時に実施） |
| 186 | 設計書のコンピュート層表記が ECS Fargate のまま（実装は EC2 t3.micro 単一） | 2026-05-20 | resolved（2026-05-25: 案D「EC2 を正本、Fargate は ADR-0004 だけに残す」で対応完了。Phase 0 設計確定 + Phase 1-4 designer 5 並列 + Phase 4 diagrams シリアル + codex 2 ラウンド経て PASS。11 ファイル EC2 化、ユーザー判断 UD-1〜7 確定、副次 issue #187（SSM Terraform 実装、起票のみ）/ #188（monitoring ロググループ命名残置、起票のみ）。コミット: db53b03 / 80dc4f4 / 92650b5 / 本コミット） |
| 187 | terraform/ec2.tf: SSM Session Manager + SSM Parameter Store 対応（issue #186 設計確定に追従） | 2026-05-24 | resolved（2026-05-25: PR #152 マージ済み squash `445240b`。architect 計画書 v2 で reviewer PASS → 内部レビュー 2 ラウンド（B-04 S3_BUCKET 名 / W-01 PEM permission 修正後 PASS）→ codex 2 ラウンド（B-05 kms:Decrypt alias ARN 無効 / B-06 IAM depends_on 修正後 PASS）。確定ユーザー判断 P-0=A / P-1=B / P-2=A / P-3=A / P-4=B / P-5=A / P-6=B / P-8=A。AWS 実 apply 系の受け入れ基準 (#1/#5/#7) は Step 11-E 実 apply セッションで消化。dev-journal コミット: e11f90e / ac9e476 / db79487 / 本コミット） |
| 188 | 監視設計のロググループ命名残置をリネーム（monitoring.md + 実装側 docker awslogs-group 設定の同期更新） | 2026-05-24 | 起票のみ（issue #186 reviewer レビュー I-01 で検出。monitoring.md L624 周辺の `/ecs/expense-saas/api` を中立名（推奨: `/expense-saas/api`）にリネーム + 実装側 Terraform `aws_cloudwatch_log_group` / `docker --log-opt awslogs-group` 同期更新 + AWS 上の新 LogGroup 作成。post-MVP、Step 11-E 実デプロイ時 or #187 と同セッションで実施想定） |

### pending-review issue

全件 resolved 移動済み（archives/progress/issues.md 参照）

### 残存 issue（運用・基盤系）
| ID | タイトル |
|----|---------|
| ops-055 | work-breakdown テンプレート不整合 |
| 060 | devcontainer egress allowlist の厳格化と根拠の欠如 |
| 061 | devcontainer マウントとシークレット露出の最小化 |
| ops-062 | ワークフロースキルの粒度 |
| 064 | MCP ジョブログが proxy allowlist でブロック |
| ops-080 | Post-MVP スコープ管理方法 |
| 081 | Post-MVP テストカバレッジ項目 |
| 084 | Post-MVP HttpOnly Cookie 移行 |
| 104 | UI 層の表示・振る舞いカバレッジ監査（ロール別表示制御 + レスポンシブ） |
| 122 | Post-MVP セッション期限切れ時のリダイレクト UX 改善（サイレント → インラインアラート + return_to） |

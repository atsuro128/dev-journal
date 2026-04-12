# Step 8 でローカル開発環境統合の配線が漏れている（シード・migration 自動適用・MinIO・起動手順書）

## 発見日
2026-04-12

## カテゴリ
infrastructure

## 影響度
高

## 発見経緯
proactive

## 関連ステップ
Step 8（基盤構築）、Step 11（システムテスト・UAT）

## ブロッカー
なし

## 問題

Step 11-A（ローカル動作確認）の実施準備として `expense-saas/` を調査した結果、ローカル開発環境としてアプリ全体を起動する配線が以下 4 点で漏れていることが判明した。これらはいずれも Step 8（基盤構築）の責務範囲内であるが、Step 8 完了時のレビューで検出されなかった。

### 1. シードデータ投入手段が存在しない

- `internal/testutil/fixture.go` に `SeedFixtures()` 関数があり、`test_strategy.md §4.2/4.3` と完全一致する固定 UUID・メールアドレス・パスワード（`TestPass1!` を Argon2id ハッシュ化）でテナント A/B・4 ロールのユーザー・6 種のレポート・6 種のカテゴリを投入できる
- しかし呼び出しは `*_test.go` からの手動呼び出しのみで、本番用 DB（docker-compose の `db` サービス）に投入するスタンドアロンツールや docker-compose での自動実行経路が存在しない
- `scripts/init-db.sh` は `expense_app` ロール作成のみで、シード投入もマイグレーション適用も行わない
- Makefile にシード投入コマンドが存在しない

結果として、`docker compose up` 直後の DB は空で、ログインできるアカウントが存在しない。smoke_check.md の 54 項目はすべて `userMember` / `userApprover` / `userAccounting` / `userAdmin` および `reportDraft` / `reportSubmitted` / `reportApproved` 等のフィクスチャを前提としているため、シードなしでは 1 項目も実施不可。

### 2. マイグレーションが自動適用されない

- `db/migrations/` に 11 本の SQL マイグレーション（000001〜000011）が存在
- `go.mod` に `github.com/golang-migrate/migrate/v4` 依存あり
- しかし Dockerfile に migrate コマンドが組み込まれておらず、docker-compose に migration 適用用の init コンテナや sidecar が存在しない
- ローカル開発では `make migrate-up` を手動実行する前提（README にも記載なし）

結果として、`docker compose up` 直後の DB にはテーブルが存在せず、API が起動しても SQL エラーで機能しない。

### 3. MinIO が docker-compose.yml に存在しない

- ADR-0004（`30_arch/adr/0004-infra.md`）の環境一覧で「ローカル: Docker Compose（Go + PostgreSQL + **MinIO**）」と明記され、決定理由にも「ローカル開発は Docker Compose で本番と同等の構成を再現する。S3 の代替として MinIO を使用」とある
- `30_arch/diagrams.md` のローカル環境構成図に `MinIO :9000 （S3 互換）` ノードが描画され、API → MinIO の接続線も存在
- `50_detail_design/files.md` §S3 互換サービスで「MinIO（Docker Compose）」と明記
- `70_operations/env_config.md` で環境変数 `S3_ENDPOINT` / `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` が MinIO 用として定義され、`.env.example` のサンプル値まで記載（`AWS_ACCESS_KEY_ID=minioadmin`）
- `60_test/test_strategy.md` で「添付ファイルテストには MinIO を使用する」と明記
- `internal/pkg/s3/client.go` は AWS SDK v2 の S3 クライアント実装で、`S3_ENDPOINT` 環境変数により MinIO エンドポイントを使用可能
- しかし `expense-saas/docker-compose.yml` に `minio` サービスが定義されておらず、`.env.example` にも S3 関連の環境変数が存在しない

結果として、smoke_check.md の添付ファイル関連チェック項目（SMK-030〜038 の 9 項目）が実施不可。添付ファイル機能は S3 必須でローカル FS フォールバックを持たないため、MinIO なしでは 501/500 エラーとなる。

### 4. ローカル開発の起動手順書が存在しない

- `expense-saas/README.md` は 1 行のみ（実質空白）
- 起動手順（`generate-keys.sh` 実行 → `docker compose up` → マイグレーション適用 → シード投入）を記述したドキュメントがない
- 将来の開発者やレビューアがローカル環境を立ち上げる際の再現性が担保されていない

## 影響

- **Step 11-A（ローカル動作確認）の実施が不可**: smoke_check.md の全 54 項目がシード・migration・MinIO のいずれかに依存しており、現状では 1 項目も実施できない。11-A のブロッカー
- **11-A 以降の 11-B/C/D/E/F が全て未着手のまま遅延**: 依存グラフ上、11-A 完了が全後続タスクの前提
- **設計実装乖離の温存**: ADR-0004・env_config.md・files.md・test_strategy.md に明記された「ローカル = Docker Compose（Go + PostgreSQL + MinIO）」という要件が実装で満たされていない状態が残る。設計書と実装の信頼関係を損なう
- **Step 8 品質ゲートの再発リスク**: Step 8 のレビュー観点に「ローカル環境でアプリ全体が起動して開発者がブラウザで触れること」が欠けていた。同様の漏れが他 Step でも起きうる

## 提案

Step 8 に新規タスク **8-11「ローカル開発環境統合」** を割り込ませ、現 8-11「整理」を **8-12** にリネームする（選択肢 Y）。

### 8-11 の実装スコープ

| # | 作業内容 | 根拠 |
|---|---------|------|
| 1 | `cmd/seed/main.go`（スタンドアロンシード CLI）を新規作成し、`testutil/fixture.go` の `SeedFixtures()` を再利用してテナント A/B・4 ロール・6 レポート・6 カテゴリを投入できるようにする | test_strategy.md §4.2/4.3、smoke_check.md 全項目の前提 |
| 2 | docker-compose.yml にマイグレーション自動適用の仕組みを追加（init コンテナ or migrate 専用サービス + depends_on で順序制御） | db/migrations/ の 11 ファイル |
| 3 | docker-compose.yml に `minio` サービスを追加（ポート 9000/9001、初期バケット作成、AWS SDK 用環境変数を api サービスに注入） | ADR-0004、env_config.md、files.md §6 |
| 4 | `.env.example` に S3 関連の環境変数を追加（`S3_ENDPOINT`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_BUCKET_NAME` 等） | env_config.md §環境変数表 |
| 5 | `expense-saas/README.md` にローカル開発の起動手順を記載（generate-keys → docker compose up → 自動 migration → 自動 seed の正規シーケンス、停止・リセット手順、トラブルシュート最低限） | 再現性確保 |
| 6 | Step 8 work-breakdown（`step8-foundation/main.md`, `review.md`）に 8-11 を追加、旧 8-11「整理」を 8-12 に rename | work-breakdown の更新 |

### スコープ外（本 issue では扱わない）

- Step 8 のレビュー観点強化（「ローカル起動確認」の必須化）は別 issue で検討
- CI ワークフローへの MinIO 追加は既存テストが `inmemory.go` モックで動作しているため 11-B/C の範囲で必要に応じて対応

### 着手順序

1. 本 issue の合意
2. Step 8 work-breakdown に 8-11 追加・旧 8-11 → 8-12 リネーム
3. 8-11 実装チケット起票（`tickets/step8/8-11-local-dev-integration.md`）
4. PR フローで実装（platform-builder）→ レビュー → マージ
5. progress.md 更新、本 issue を resolved に移動
6. 11-A 着手

### 影響範囲（リネーム対象）

- **要更新**: `progress.md` L45, `tickets/step8/8-11-cleanup.md`（→ `8-12-cleanup.md` にリネーム）, `ai-dev-framework/guide/work-breakdown/step8-foundation/main.md` L109,110,118,119,131,138,293, `step8-foundation/review.md` L43,82
- **更新しない（歴史保持）**: `dev-journal/archives/**`, `dev-journal/issues/resolved/**`, `dev-journal/review-findings/resolved/**` — 旧 8-11「整理」の記録はそのまま保持

---

## 解決内容

PR #44（Step 8-11「ローカル開発環境統合」）として対応完了。2026-04-12 に master へスカッシュマージ（merge commit `82497f1`）。

### 実装した配線

1. **シード投入 CLI**: `cmd/seed/main.go` + `internal/seed/seed.go` を新設。`testutil/fixture.go` の `SeedFixtures()` を `seed.Run()` のラッパーに縮退させ、本番用 DB と test 用 DB の両方から同一ロジックを呼ぶ構造に統一
2. **マイグレーション自動適用**: `docker-compose.yml` に `migrate` init コンテナ（`migrate/migrate:v4.17.0`）を追加し、`depends_on.service_completed_successfully` で api/seed の起動順序を制御
3. **MinIO 統合**: `minio` サービス + `minio-init` mc サイドカー（`expense-saas-receipts-dev` バケット作成）を追加。`.env.example` と docker-compose.yml に S3 関連環境変数を注入
4. **README 起動手順**: 正規シーケンス（`generate-keys.sh` → `docker compose up -d` → `make seed`）、サービス表、停止/リセット、トラブルシュートを記載
5. **seed サービスの Compose 統合**（codex 3 回目レビュー指摘対応）: `seed` サービスを `profiles: [seed]` で新設し、Dockerfile のマルチステージに seed バイナリビルドを追加。`make seed` を `docker compose --profile seed run --rm seed` に変更してホスト Go 非依存化
6. **添付ファイルフィクスチャ**: `reportSubmitted` と `reportDraft` の両方に添付レコードと MinIO オブジェクトを seed 投入し、smoke_check.md の SMK-030〜038（特に SMK-037/038）を seed 直後に独立実施可能に

### work-breakdown 更新

- Step 8 main.md / review.md を更新
- 新タスク 8-11「ローカル開発環境統合」を割り込ませ、旧 8-11「整理」を 8-12 にリネーム
- 歴史ファイル（`dev-journal/archives/**`, `issues/resolved/**`, `review-findings/resolved/**`）の旧 8-11 参照は保持

### レビュー履歴

- reviewer: 5 回（1 回目 FIX → 2〜5 回目 PASS）
- codex: 4 回（1〜3 回目 FIX → 4 回目 PASS）
- CI: 全 PASS（ECONNRESET 一時障害でリラン 1 回）

### 派生 issue

本 PR のレビューで顕在化した設計書・実装乖離を別 issue として分離:

- **issue 077**: `S3_PRESIGNED_URL_EXPIRY` が設計書定義されているが実装コードで環境変数参照されていない
- **issue 078**: `S3_REGION`（設計書）vs `AWS_REGION`（実装）の変数名乖離

## 解決日
2026-04-12

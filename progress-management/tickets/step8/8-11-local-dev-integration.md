# ローカル開発環境統合

- 担当: platform-builder
- 依存: 8-1（db/migrations/・docker-compose 基盤）、8-2（バックエンド env 読み込み）、8-3（フロントエンド設定）、8-8（testutil/fixture.go の SeedFixtures()）
- ブランチ: step8/8-11-local-dev-integration
- 出力先: expense-saas/ (docker-compose.yml, cmd/seed/, .env.example, README.md, Makefile)
- テンプレート: なし

## 背景

Step 11-A（ローカル動作確認）着手準備で以下 4 点の配線漏れが発覚。issue 076 で起票済み（`dev-journal/issues/open/076-step8-local-dev-environment-integration-missing.md`）。本チケットは issue 076 の対応として Step 8 に遡及追加したもの。

1. シード投入手段が存在しない（`SeedFixtures()` は test 専用）
2. マイグレーションが `docker compose up` で自動適用されない
3. MinIO サービスが `docker-compose.yml` に存在しない
4. README にローカル開発の起動手順が記載されていない

根拠: ADR-0004・env_config.md・files.md・test_strategy.md で「ローカル = Docker Compose（Go + PostgreSQL + MinIO）」と明記されているが、Step 8 実装時にその配線が漏れた。

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| インフラ ADR | deliverables/docs/30_arch/adr/0004-infra.md | ローカル構成方針「Docker Compose（Go + PostgreSQL + MinIO）」 |
| 環境構成 | deliverables/docs/70_operations/env_config.md | 環境変数表・.env.example サンプル（S3_ENDPOINT / AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / S3_BUCKET_NAME） |
| ファイル設計 | deliverables/docs/50_detail_design/files.md | §6 S3 互換サービス = MinIO |
| テスト戦略 | deliverables/docs/60_test/test_strategy.md | §4.2 ユーザー構成・§4.3 レポートフィクスチャ・§4.4 カテゴリフィクスチャ |
| スモークチェックリスト | deliverables/docs/60_test/manual_checklists/smoke_check.md | 前提フィクスチャ（userMember 等）と添付系チェック（SMK-030〜038） |
| 既存シード関数 | expense-saas/internal/testutil/fixture.go | `SeedFixtures()` — 再利用元 |
| 既存 S3 クライアント | expense-saas/internal/pkg/s3/client.go | AWS SDK v2、`S3_ENDPOINT` で MinIO 切替対応済み |
| 既存 docker-compose | expense-saas/docker-compose.yml | 統合対象 |
| 既存マイグレーション | expense-saas/db/migrations/000001〜000011 | 自動適用対象 |
| 既存起動スクリプト | expense-saas/scripts/init-db.sh, generate-keys.sh | 参考 |

## 責務

### 1. シード投入 CLI

- `cmd/seed/main.go` を新規作成
- `internal/testutil/fixture.go` の `SeedFixtures()` を再利用してテナント A/B・4 ロールのユーザー・6 レポート・6 カテゴリを投入
- `DATABASE_URL` を環境変数から読み込み
- 再実行時の冪等性を担保（既存データがあれば削除 → 再投入、または INSERT ... ON CONFLICT）
- Makefile に `make seed` エイリアスを追加
- docker-compose の seed 専用サービスまたは `docker compose run --rm seed` で実行可能にする

### 2. マイグレーション自動適用

- `docker-compose.yml` にマイグレーション実行の仕組みを追加
- 方式の選択肢（判断ポイント参照）から1つを採用し、`docker compose up` 直後に DB スキーマが適用される状態にする
- db サービス healthy 後・api サービス起動前にマイグレーションが完了する順序制御

### 3. MinIO 追加

- `docker-compose.yml` に `minio` サービスを追加
  - イメージ: `minio/minio:latest`（または pinned tag）
  - ポート: 9000（API）、9001（コンソール）
  - 認証: `MINIO_ROOT_USER=minioadmin` / `MINIO_ROOT_PASSWORD=minioadmin`
  - ヘルスチェック: `mc ready local` または `/minio/health/live`
  - ボリューム: 永続化用の named volume
- 初期バケット作成の仕組み（entrypoint script / mc サイドカー / 初回投入サービスのいずれか）
- api サービスに S3 関連環境変数を注入:
  - `S3_ENDPOINT=http://minio:9000`
  - `AWS_ACCESS_KEY_ID=minioadmin`
  - `AWS_SECRET_ACCESS_KEY=minioadmin`
  - `S3_BUCKET_NAME=expense-attachments`（env_config.md に合わせる）
  - `AWS_REGION` が必要なら `us-east-1` 等のダミー値
- api サービスの `depends_on` に minio を追加（healthy 条件）

### 4. .env.example 更新

- S3 関連環境変数を追記（env_config.md §環境変数表と一致させる）
- 既存変数との整合性を保つ

### 5. README 起動手順

- `expense-saas/README.md` に以下を記載:
  - **前提**: Docker / Docker Compose / Go（テスト実行用に必要であれば）
  - **初回セットアップ**: `./scripts/generate-keys.sh` を実行して JWT 鍵を生成
  - **起動**: `docker compose up -d` で全サービス（db, api, frontend, minio）を起動。db のマイグレーションが自動適用される
  - **シード投入**: `make seed`（または `docker compose run --rm seed`）で開発用フィクスチャを投入
  - **アクセス**:
    - フロントエンド: http://localhost:5173
    - API: http://localhost:8080
    - MinIO コンソール: http://localhost:9001
  - **ログイン情報**: `test_strategy.md §4.2` のテストアカウント一覧を参照（リンクを張る）
  - **停止・リセット**: `docker compose down -v` で完全リセット
  - **トラブルシュート**: 最低限（port 競合、鍵未生成、DB ヘルスチェック失敗）

### 含めない

- CI ワークフローへの MinIO 追加（既存テストは `inmemory.go` モックで動作するため、本チケットでは扱わない）
- Step 8 のレビュー観点強化（「ローカル起動確認」の必須化は別途検討）
- 本番デプロイ手順（Step 11-E の責務）

## 判断ポイント

### マイグレーション実行方式
| 方式 | メリット | デメリット |
|------|---------|-----------|
| init コンテナ（ワンショット） | depends_on で順序制御しやすい、api 本体は migration 知識不要 | サービス数が増える |
| migrate 専用サービス（exit 0 で終了） | init コンテナとほぼ同等、profile 分離可 | 上と同様 |
| api サービス起動時の内部呼び出し | 追加サービス不要、シンプル | api のライフサイクルに migration が混入、起動時間増 |

**推奨**: init コンテナ方式（golang-migrate の公式イメージ `migrate/migrate` を使用）。api 本体は migration 非依存のまま。

### シード自動実行タイミング
| 方式 | メリット | デメリット |
|------|---------|-----------|
| 初回起動時自動実行 | 開発者が手順を忘れない | DB リセット時に毎回自動投入で開発ワークフローを阻害することも |
| 明示実行（`make seed`） | 開発者が意図的にタイミングを選べる | 手順書通りに叩かないと詰まる |

**推奨**: 明示実行（`make seed`）。README で正規シーケンスとして明記する。

### MinIO 初期バケット作成
| 方式 | メリット | デメリット |
|------|---------|-----------|
| minio サービスの entrypoint script | サービス1つで完結 | カスタム entrypoint が必要 |
| mc サイドカー（`minio/mc`） | 公式ツール、宣言的 | サービスが増える |
| Go コード側で起動時に CreateBucket | アプリで完結 | 起動時に副作用、権限設計の複雑化 |

**推奨**: mc サイドカー（ワンショット実行で exit）。公式で宣言的。

## 完了条件

- `./scripts/generate-keys.sh` 実行後、`docker compose up -d` のみで全サービス（db, api, frontend, minio）が起動する
- `docker compose up` 完了時点で DB マイグレーションが自動適用され、`expense_reports` 等のテーブルが存在する
- `make seed`（または同等コマンド）で `test_strategy.md §4.2/4.3` のフィクスチャが投入される
- 投入後、ブラウザから http://localhost:5173 にアクセスし、`userMember` のメールアドレスとパスワード（`TestPass1!`）でログイン成功する
- 添付ファイル機能（明細への領収書添付）が MinIO 経由で動作し、アップロード・ダウンロード・削除が完了する
- `.env.example` に S3 関連環境変数（`S3_ENDPOINT`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_BUCKET_NAME`）が記載されている
- `expense-saas/README.md` に正規起動シーケンス・停止/リセット手順が記載されている
- 既存の CI（lint / test / build）が引き続き通る
- 既存テストコード（`inmemory.go` モックを使用）の動作に影響を与えていない

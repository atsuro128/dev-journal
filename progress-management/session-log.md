# 引き継ぎメモ

## セッション: 2026-05-20 01:25

### ゴール

- EC2 user_data の env ファイル確認（前回セッション持ち越し）
- Step 11-E Phase 4（DB bootstrap）完走
- Phase 5-7 は次セッション以降

→ **両方達成。Phase 4 完了**。

### 作業ログ

#### 1. EC2 user_data env 確認（Task #1）

- SSH 接続: 秘密鍵 `~/.ssh/expense-saas-portfolio.pem` がホスト Windows 側に未配置だった
  - `Desktop\AWS\expense-saas-portfolio.pem` に保存されていることを発見
  - PowerShell の `Move-Item` + `icacls` で `~/.ssh/` に配置 + 権限を本人のみアクセス可に絞る
- EC2 (`13.158.141.63`) に ED25519 fingerprint 認証 → ログイン成功
- 確認結果: `app.env` の `CHANGEME_AFTER_MIGRATE` / `CHANGEME_RDS_HOST` / `CHANGEME_S3_BUCKET` / `http://placeholder.invalid` は user_data.sh.tpl の **意図通りプレースホルダ**（Phase 4 Step 7 で更新する設計）
- JWT 鍵 (`/etc/expense-saas/keys/*.pem` root:600) / systemd unit / Docker 25.0.14 / swap 2Gi / git すべて配置済み確認

#### 2. Phase 4 マイグレーション資材の持ち込み（PAT 方式）

- expense-saas が GitHub private リポジトリのため、EC2 から git clone するには認証が必要
- **GitHub fine-grained PAT** を発行（name: `expense-saas-ec2-deploy` / Contents Read-only / 7 days / expense-saas のみ）
- EC2 で `read -s` でトークン入力 → `git clone https://${GH_PAT}@github.com/atsuro128/expense-saas.git` → `unset GH_PAT`
- `~/expense-saas/db/migrations/` に 11 本のマイグレーション SQL 配置確認

#### 3. Phase 4 Step 1: 000001/000002 をマスターで migrate up 2（Task #2）

- EC2 に `postgresql16` + `golang-migrate v4.17.0` をインストール
- terraform.tfvars から 3 種類のパスワード（`db_password` / `expense_owner_db_password` / `expense_app_db_password`）をクリップボード経由で SSH に転送
- `terraform output -raw rds_endpoint` で RDS エンドポイント取得（初回 `terraform init -backend-config="bucket=expense-saas-tfstate-oc0sqjmb"` 再実行が必要だった）
- `RDS_HOST` 入力時に `:5432` ポート付きで入ったため `${RDS_HOST%:5432}` で除去
- `migrate up 2` 実行 → `schema_migrations.version=2 / dirty=f` 確認

#### 4. Phase 4 Step 2: マスター接続でロール / 権限基盤整備（Task #3）

3 つを一括投入:
- `ALTER ROLE expense_owner / expense_app WITH PASSWORD '...'`（migration の localdev → prod 値上書き）
- `GRANT CREATE, USAGE ON SCHEMA public TO expense_owner`（PG15+ の public schema 制限対応）
- `ALTER TABLE schema_migrations OWNER TO expense_owner`（**F-115 残課題対応**）

`\dt schema_migrations` で Owner = `expense_owner` 確認。

#### 5. Phase 4 Step 3: expense_owner で default privileges 仕込み（Task #4）

接続を `expense_owner` に切替 → `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner` で expense_app に TABLES / SEQUENCES の権限を投入（**F-118 一次対策**）。

初回 SEQUENCES を打ち忘れて再投入。`\ddp` で 3 行（owner 所有 2 行 + master 所有 1 行）確認。

#### 6. Phase 4 Step 4: 000003 以降を expense_owner で migrate（Task #5）

- `migrate up`（無指定 = 残り全部）で 000003〜000011 を実行
- 全 10 テーブルの owner = `expense_owner` を `pg_tables` で確認
- `schema_migrations.version=11 / dirty=f` 確認

#### 7. Phase 4 Step 5: expense_app への保険 GRANT（Task #6）

- `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO expense_app`（**F-118 二重ガード**）
- `GRANT USAGE ON ALL SEQUENCES`（実行時対象 0 件、将来用保険）
- `\dp tenants` で `expense_app=arwd/expense_owner` + RLS ポリシー 2 種（`tenant_isolation_select` / `update`）確認

#### 8. Phase 4 Step 6: 疎通確認（完了ゲート、Task #7）

- `expense_owner` で `SELECT 1` → 1 ✅
- `expense_app` で `SELECT 1` → 1 ✅
- `has_table_privilege()` 9 テーブル × 4 権限（SELECT/INSERT/UPDATE/DELETE）= 全て `t` ✅（**F-118 検証**）
- `expense_app` で `SELECT count(*) FROM users` → 0（RLS 非対象テーブル、エラーなし＝権限あり、**F-119 対応**）

#### 9. Phase 4 Step 7: app.env 更新（Task #8）

- `terraform output -raw s3_bucket_name` で `expense-saas-portfolio-receipts-d8ed055a` 取得
- CORS は ALB DNS `http://expense-saas-portfolio-alb-290554356.ap-northeast-1.elb.amazonaws.com` を採用
- `sudo tee` のヒアドキュメントで `/etc/expense-saas/app.env` を全面上書き
- **トラブル**: 初回 `cat -A` で先頭スペース混入 + パスワード平文露出を検出
  - 先頭スペース: `sudo sed -i 's/^[[:space:]]\+//'` で除去（端末の PS2 継続プロンプト混入が原因の可能性）
  - パスワード露出: ユーザー判断「portfolio 本番データなし、対応不要」でリスク許容
- 確定値の全 9 行が `^VAR=VALUE$` 形式で書き込まれていることをマスク版で確認
- systemd 再起動は **docker image 未ビルド** のため Phase 5 に持ち越し

### 完成した成果

| 項目 | 結果 |
|---|---|
| EC2 user_data 確認 | ✅ 想定動作通り（プレースホルダは意図通り） |
| GitHub PAT 認証経路の確立 | ✅ EC2 で git clone 成功 |
| RDS スキーマ初期化 | ✅ 10 テーブル + schema_migrations、version=11 |
| ロール / 権限設定 | ✅ expense_owner = テーブル所有、expense_app = DML 権限 |
| F-115 / F-118 / F-119 完了ゲート | ✅ 全項目 PASS |
| app.env 確定値書き込み | ✅ 全 placeholder 解消 |

### 未完了

- **Phase 4 完了済み、未完了タスクなし**
- ただし以下は **次セッション開始時に確認** が必要:
  - SSH セッション内の機密環境変数（OWNER_DB_PW / APP_DB_PW / MASTER_PW）クリア状況
  - EC2 シェル履歴（`history`）に残ったパスワード入力痕跡（`read -s` で入れたので痕跡なしのはず）

### ブロッカー

なし

### 次にやること

#### 優先度 1: Step 11-E Phase 5（Docker build + systemd 起動）

11-E チケット §5.6 / §11 Q1 案 B（EC2 上で docker build）に従う。

主要ステップ（USER 主導、工数見積 2-4h、t3.micro メモリ 1GB + swap 2GB で OOM 注意）:
1. `cd ~/expense-saas && docker build -t expense-saas:portfolio .`
2. `sudo systemctl enable --now expense-saas`
3. `sudo systemctl status expense-saas` で active 確認
4. `docker logs expense-saas` で起動成功 + DB 接続成功確認

#### 優先度 2: Step 11-E Phase 6（ヘルスチェック）

- EC2 上 `curl localhost:8080/healthz` → 200 OK
- ALB 経由 `curl http://expense-saas-portfolio-alb-290554356.ap-northeast-1.elb.amazonaws.com/healthz` → 200 OK
- ALB Target Group が healthy 状態になることを確認

#### 優先度 3: Step 11-E Phase 7（スモーク + 時刻ドリフト確認）

- ブラウザで ALB DNS にアクセス → ログイン画面表示確認
- テストアカウント作成 → 経費登録の golden path をブラウザで動作確認
- JWT leeway（issue #173）が時刻ドリフト下で機能するかも確認
- 工数見積: 1-2h

#### 優先度 4: Step 11-F（UAT、ユーザー主導）

- `dev-journal/deliverables/docs/manual_checklists/uat_check.md` に従い、画面別チェック
- 工数見積: 2-4h

### 学び・気づき

#### user_data の「意図的プレースホルダ」設計

`/etc/expense-saas/app.env` が `CHANGEME_AFTER_MIGRATE` / `placeholder.invalid` で生成されているのは **user_data.sh.tpl の意図通り**。DB マスターパスワードを user_data に流す経路を作らないため、migration 完了後に手動更新する 2 段階運用。

→ 初見では「user_data がバグっている」と誤読しがち。設計意図を確認してから「直す / 直さない」判断する。

#### SSH 秘密鍵の所在管理

AWS コンソールで作成した EC2 キーペアの `.pem` ファイルは **作成時 1 回のみダウンロード可能**。今回 `Desktop\AWS\` に保存されていたが `~/.ssh/` には未配置だった。`ssh-keygen` で作り直すには **新しいキーペア作成 + terraform apply 再実行** が必要なため、`.pem` 紛失は重大事故。

→ AWS で作成した `.pem` は **発行直後に `~/.ssh/` に配置 + `icacls`（Windows）/ `chmod 400`（Linux）で権限絞り**。Desktop / Downloads には残さない（バックアップは別途暗号化保管）。

#### PostgreSQL 15+ の public スキーマ CREATE 制限

PG15 から `public` スキーマへの `CREATE` 権限が default で剥奪された。`expense_owner` で 000003 以降の `CREATE TABLE` を実行する前に `GRANT CREATE, USAGE ON SCHEMA public TO expense_owner` が必須。

→ migration ファイル設計時、`public` スキーマ前提のテーブル作成は PG15+ で動かないことがある。bootstrap 手順に GRANT を含める。

#### heredoc 経由 sudo tee の先頭スペース混入

`sudo tee <<APPENV ... APPENV` で書き出した `/etc/expense-saas/app.env` の 2 行目以降に先頭 2 スペースが混入。原因は端末の PS2（継続プロンプト）が `>  ` だったため、コピペ時に取り込まれた可能性。Docker `--env-file` は厳密で `^VAR=` 期待のため、これは Phase 5 起動時の env 認識ミスを誘発する。

→ heredoc 書き出し後は必ず `cat -A` で `$` 直前と行頭を確認する。混入していたら `sed -i 's/^[[:space:]]\+//' ...` で除去。

#### GitHub PAT vs gh CLI device flow の選択

private repo を EC2 から clone する手段として PAT / gh CLI / SSH deploy key の 3 通りを提示。Fine-grained PAT は **CLI からは発行できない**（GitHub のセキュリティ仕様）。今回は手動 PAT で進めたが、`gh auth login` の device flow なら手動 PAT 作成を省略できる。

→ EC2 から private repo を扱う場面では、`gh CLI device flow` が PAT より自動化に向く（ただし対話 1 回必要）。

#### チャット履歴へのパスワード平文露出

`cat -A` で出力した app.env に DB パスワード（OWNER_DB_PW / APP_DB_PW）が平文表示され、チャット履歴に残った。ユーザー判断「portfolio 本番データなし、対応不要」でリスク許容したが、業務プロジェクトでは即時ローテーション必須。

→ 機密値の確認には `sed -E 's|:[^@]+@|:***REDACTED***@|g'` 等のマスクを **常に通す**。生 `cat` / `cat -A` を機密ファイルに使わない。

### 意思決定ログ

#### EC2 へのマイグレーション資材持ち込み手段: GitHub Fine-grained PAT

選択肢:
- A: Fine-grained PAT（Web UI 発行、Contents Read-only、7 days）
- B: scp でローカルから migrations のみ転送
- C: SSH deploy key

採用 A、理由: Phase 5 で docker build に repo 全体が必要、PAT なら一括カバー。CLI で PAT 発行不可な代わりに `gh CLI device flow` も候補に挙がったが、ユーザー判断で「最初の案（Web UI PAT）で良い」と即決。

#### CORS_ALLOWED_ORIGINS の値: ALB DNS の http のみ

選択肢:
- A: ALB DNS (`http://expense-saas-portfolio-alb-...amazonaws.com`)
- B: ALB DNS + localhost:5173 両方
- C: placeholder.invalid のまま Phase 5 で確定

採用 A、理由: portfolio MVP かつ独自ドメイン未取得、Phase 5 で frontend を docker image に同梱 + ALB 経由配信する前提。

#### app.env 更新範囲: 全 placeholder を一気に確定

選択肢:
- A: DB 関連のみ（DATABASE_URL / APP_DATABASE_URL）→ Phase 5 で S3 / CORS 確定
- B: 全 4 項目を一気に確定

採用 B、理由: ユーザー所感「S3_BUCKET を忘れてしまいそう」。Step 7 のスコープを少し広げて作業の二重手間を回避。

#### パスワード露出への対応: 対応不要

`cat -A` 出力に DB パスワードが平文露出した件、ユーザー判断「AWS アカウントは個人所有、RDS は portfolio 加入のみ、chat 履歴は anthropic とユーザーに限るためリスク許容」で **ローテーション見送り**。

#### Phase 4 完了後の進め方: セッション終了 + session-log

選択肢:
- A: session-log で一区切り、Phase 5-7 は次セッション
- B: 同セッションで Phase 5 着手（docker build に 30-60 分かかる見込み）
- C: progress.md 反映のみして停止

採用 A、理由: ユーザー判断「セッション終了（session-log）」。docker build の長時間ブロックを避け、Phase 5 着手は別セッションのリフレッシュ状態で始める方が安全。

### PR / コミット要約

**expense-saas**: 変更なし（master 直 push なし、PR なし）

**dev-journal**: progress.md 更新 + session-log 移行（本セッション末で実施予定）

**root-project / ai-dev-framework**: 変更なし

**起票 issue**: なし

**解決 issue**: なし

### Phase 状態サマリ（次セッション着手時の前提）

| Phase | 状態 | 担当 |
|-------|------|------|
| Phase 0-3 | **完了**（既存セッション） | - |
| NFR-AVAIL-003 緩和 | **完了**（2026-05-19） | - |
| Phase 4 DB bootstrap | **完了**（今セッション） | - |
| Phase 5 Docker build + systemd | 未着手（次優先） | USER |
| Phase 6 ヘルスチェック | 未着手 | 指揮役 + USER |
| Phase 7 スモーク + 時刻ドリフト | 未着手 | USER（操作）+ 指揮役（観察）|

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-19.md`（2026-05-19 夜: NFR-AVAIL-003 緩和の正式化 PR #146 / #147、設計修正 14 箇所、issue #181 resolved、issue #182 起票）

# 引き継ぎメモ

## セッション: 2026-05-25 〜 2026-05-26

（前回セッション直後、05-25 朝〜26 朝にかけて連続実施）

### ゴール

- 「次の作業は？」から /session-start で前回引き継ぎを確認。issue #187（SSM Terraform 実装）→ #188（CloudWatch Logs awslogs 実装）を連続対応する。
- → **両方完全クローズ達成。PR #152（#187）+ PR #153（#188）マージ済み。**

### 作業ログ

#### Phase 1: issue #187（SSM Session Manager + SSM Parameter Store Terraform 実装）

1. **計画策定（2026-05-25 朝）**:
   - architect 計画書 v1 → reviewer CONDITIONAL PASS（Blocker 3 + Warning 5）→ v2 で全反映 → reviewer PASS。
   - 主要 Blocker: B-01 PEM 多行値 awk 取り出し破綻（jq + --output json に切替）/ B-02 IAM Action 整合 / B-03 /var/log/user-data.log 平文残存リスク。
   - 新規ユーザー判断 P-8（env_config.md §5.3.3/§5.3.5 を同 PR に含める）追加。
   - ユーザー判断 8 項目（P-0=A / P-1=B / P-2=A / P-3=A / P-4=B / P-5=A / P-6=B / P-8=A）全推奨案で確定。

2. **実装 + レビューサイクル（2026-05-25 昼〜夕）**:
   - platform-builder で T1-T8 + env_config.md 実装、PR #152 作成（4 コミット、+341/-221、8 ファイル変更）。
   - 内部レビュー初回 FAIL: B-04（S3_BUCKET 名不一致、`expense-saas-receipts-portfolio` ↔ 実際 `expense-saas-portfolio-receipts-<8hex>` で `NoSuchBucket` リグレッション）+ W-01（PEM 一時 permission 0644 リスク）→ 修正コミット `2a3e81b` → 内部レビュー PASS。
   - codex 初回 FAIL: B-05（`kms:Decrypt` の Resource に alias ARN は無効、AWS managed key の自動 grant で代替可）+ B-06（EC2 が IAM policy 作成完了に `depends_on` していない）→ 修正コミット `862a3a4`（ec2_kms_decrypt 削除 + depends_on 追加 + ssm get-parameters 5 回 retry）→ codex 再レビュー PASS。

3. **マージ + クローズ（2026-05-25 夜）**:
   - PR #152 squash マージ commit `445240b`
   - issue #187 §解決内容 / §解決日 記入 → resolved 移動 → progress.md 更新 → コミット `b6d4a40`
   - worktree 4 個 + 独自ブランチ 3 個クリーンアップ（step11/issue-187 ブランチは保持）

#### Phase 2: issue #188（CloudWatch Logs awslogs ドライバ完全実装）

4. **スコープ拡大判定（2026-05-25 夜）**:
   - 当初想定「monitoring.md L624 リネームのみ」→ 現状調査で **設計 vs 実装の重大な乖離** を発見:
     - 設計: monitoring.md §1 で UD-2=B「Docker awslogs ドライバで CloudWatch Logs 自動転送」と明記
     - 実装: user_data.sh.tpl の docker run に `--log-driver=awslogs` も `--log-opt awslogs-group=...` も **一切なし**（json-file ドライバ = ホスト内ログのみ、CloudWatch 転送ゼロ）
     - Terraform: `aws_cloudwatch_log_group` リソース **未定義**、IAM `logs:*` 権限 **なし**
   - issue 受け入れ基準を満たすため **awslogs 完全実装** にスコープ拡大（ユーザー承認）

5. **計画策定（2026-05-26 朝）**:
   - architect 計画書 v1 → reviewer CONDITIONAL PASS（Blocker 2 + Warning 4 + Info 5）→ v2 で全反映 → reviewer PASS。
   - 主要 Blocker: B-01 分岐元コミット記述ミス（`619a702` は root-project HEAD、expense-saas は `445240b`）/ B-02 systemd `%i` 使用不可（template identifier）→ heredoc 方式 X（`<<'SYSTEMD_EOF'` → `<<SYSTEMD_EOF` でクォート除去 + bash 展開）確定 + 副作用検査済み。
   - ユーザー判断 7 項目（P-0=c / P-1=b / P-2=a / P-3=a / P-4=b / P-5=a / P-6=a）全推奨案で確定。
   - **P-1=b は issue 起票案 B (`/expense-saas/api`、env なし) から逸脱**（SSM パス `/expense-saas/{env}/...` と階層整合させるため `/expense-saas/{env}/api` 採用）。

6. **実装 + レビューサイクル（2026-05-26 朝）**:
   - platform-builder で T1-T4 + T10 実装、PR #153 作成（3 コミット、+ dev-journal 5 ファイル別途修正）。
   - 内部レビュー: **1 ラウンド PASS**（Blocker 0 / Warning 0 / Info 1）
   - codex レビュー: **1 ラウンド APPROVE 相当**（Blocker 0）

7. **マージ + クローズ（2026-05-26 朝）**:
   - PR #153 squash マージ commit `420f0ea`
   - issue #188 §解決内容 / §解決日 記入 → resolved 移動 → progress.md 更新 → コミット `86228e5`
   - worktree 2 個 + 独自ブランチ 1 個クリーンアップ（step11/issue-188 ブランチは保持）

### 未完了

- **issue #185 T4**（`CORS_ALLOWED_ORIGINS` を CloudFront ドメインに、`TRUSTED_PROXY_COUNT=2` を prod に投入）— AWS 実 apply 待ち
- **issue #185 T6**（CloudFront 経由疎通・CORS・HSTS・レート制限検証）— AWS 実 apply 待ち
- **issue #187 / #188 受け入れ基準の AWS 実環境疎通系**（合計 9 項目程度）— Step 11-E 実 apply 時に消化
  - #187: SSM Session Manager 接続 / SSH 不可確認 / SecureString 取得 / CloudTrail / describe-instance-attribute / アプリ起動疎通
  - #188: 新 LogGroup へのログ到達確認 / 旧 LogGroup `/ecs/expense-saas/api` の存在確認 → 削除 or 放置判断
- **Step 11-E Phase 7 スモークテスト**（seed 投入 + 申請→承認→支払 golden path）— AWS 実 apply 後
- **Step 11-F UAT 着手**（MVP 完成判定）

### ブロッカー

なし

### 次にやること

優先順位:

1. **Step 11-E 実 apply セッション（ユーザー主導 AWS 操作）**:
   - 事前に SSM Parameter Store 投入（DATABASE_URL / APP_DATABASE_URL / JWT 秘密鍵 / JWT 公開鍵 の 4 件、`aws ssm put-parameter --type SecureString`）
   - `terraform apply` で SSM + CloudWatch Logs リソース作成、EC2 再作成（`user_data_replace_on_change = true`）
   - issue #187 受け入れ基準 6 項目 + issue #188 受け入れ基準 #3/#4 を消化
   - issue #185 T4 (env 反映) + T6 (疎通検証)
2. **Step 11-E Phase 7 スモークテスト**（seed 投入 + 申請→承認→支払 golden path）
3. **Step 11-F UAT 着手**（MVP 完成判定）
4. **post-MVP issues 整理**: 060 / 061 / 064 / 081 / 084 / 104 / 122 / 133 / 145 / 146 / 151 / 167 / 174 / 176 / 177 / 178 / 179 / 180 / 182 + ops-055 / 062 / 080

### 学び・気づき

- **内部レビューと codex の役割分担**: PR #152 では内部レビュアーが B-04（S3_BUCKET 名）を発見、codex が B-05 / B-06（KMS alias ARN 無効 + IAM eventual consistency）を発見。**コード論理整合は内部レビュー、外部仕様（AWS / Docker 等）整合は codex** という分業が機能した。memory `feedback_critical_review_of_codex` の運用例として正解パターンの再確認。
- **計画書レビュー → ユーザー判断提示の順序が効いた**: feedback_review_before_user_judgment に従い、両 issue とも architect 計画書 → reviewer PASS → ユーザー判断項目提示の順で進めた結果、ユーザーに提示する判断項目が確実に「品質チェック済みの選択肢」になり、判断負荷が減った。
- **PR #152 → PR #153 へのパターン継承が機能**: PR #152 で確立した「depends_on パターン / IAM eventual consistency retry / 機密区間 /dev/null リダイレクト / atomic PEM permission」が PR #153 でも踏襲され、内部レビュー・codex とも **1 ラウンドで PASS**。同じ問題を 2 度起こさない仕組みとして有効。
- **issue スコープの再評価が必要だった例**: issue #188 は「リネームのみ」想定で起票されたが、現状調査で「設計と実装の完全乖離」が判明し、本来の意図を満たすには awslogs 実装そのものが必要だった。**issue 着手時に必ず「設計通り動いているか」を grep で確認する習慣** を継続すべき。
- **Agent ツールの isolation: worktree 挙動**: `isolation: "worktree"` で起動した Agent は新規 worktree を作る一方、既存 worktree を再利用させる手段が標準にない。今回は「worktree path を明示し isolation 指定なしで Agent 起動」で既存 worktree を使わせたが、その場合でも Agent が自動的に新規 worktree を作ってしまうケースがあった（agent-a209034 / agent-ac8c36 が空のまま残置）。これは harness 側の仕様。後で worktree 散乱掃除が必要になるので、起動時にユーザーに明示しておくと良い。
- **dev-journal 修正の worktree 外編集**: expense-saas worktree で作業中に dev-journal を直接編集することは許容するが、コミットは指揮役が責任を持つ。今回は env_config.md 修正を 2 回（B-01/B-02 と B-05）に分けて反映したが、それぞれ平文 vs 暗号化注記 / KMS 削除注記と独立した内容で、別コミットにする判断が妥当だった。

### 意思決定ログ

#### issue #187 確定したユーザー判断（計画書 v2 §3）

- **P-0=A**: JWT PEM 2 件のみ tfvars から削除、DB password 3 件は Terraform `master_password` 必須のため残置
- **P-1=B**: SSM Parameter は手動 `aws ssm put-parameter` 投入（IaC 一貫性より機密分離優先、`ssm.tf` 作成しない）
- **P-2=A**: KMS は AWS managed key `alias/aws/ssm`
- **P-3=A**: 既存 EC2 は `user_data_replace_on_change = true` + terraform apply で再作成
- **P-4=B**: AWS 実環境疎通系の受け入れ基準は Step 11-E 切り出し
- **P-5=A**: SSH 関連変数完全削除
- **P-6=B**: 非機密項目（CORS / proxy_count）は Terraform variable のまま
- **P-8=A**: env_config.md §5.3.3 (awk → jq) / §5.3.5 (IAM Resource 差分注記) を同 PR スコープで同時修正
- **codex 修正で確定**: KMS managed key `alias/aws/ssm` は SSM SecureString 復号で自動 grant が付与されるため、明示的な `kms:Decrypt` ポリシーは不要（B-05 → ec2_kms_decrypt 削除採用）

#### issue #188 確定したユーザー判断（計画書 v2 §2）

- **P-0=c**: 旧 LogGroup `/ecs/expense-saas/api` は無視（Step 11-E 実 apply 直前に AWS console 確認 → 手動判断）
- **P-1=b**: ロググループ命名 `/expense-saas/${var.environment}/api`（env 込み、SSM パス命名と階層整合）— issue 起票案 B (`/expense-saas/api`) から逸脱
- **P-2=a**: awslogs-stream は `${INSTANCE_ID}/api`（IMDSv2 取得 + bash 展開、systemd `%i` は instance template identifier のため使用不可）
- **P-3=a**: `logs:CreateLogGroup` 不要、Terraform で LogGroup を先行作成
- **P-4=b**: AWS 実環境疎通検証は Step 11-E 切り出し
- **P-5=a**: nginx 等の追加 LogGroup は本 issue では扱わない
- **P-6=a**: env_config.md への新規変数追加なし

#### heredoc 方式 X 採用根拠

- user_data.sh.tpl L129-147 の systemd unit 中身に `$` を含む箇所はゼロ（事前検査済み）→ `<<'SYSTEMD_EOF'` → `<<SYSTEMD_EOF` への切替で副作用なし
- templatefile レイヤと bash 展開レイヤを `${var}` vs `$${VAR}` で責務分離

### PR / コミット要約

#### expense-saas（PR 2 件マージ）

- **PR #152** `infra(terraform): migrate EC2 access + secrets to SSM (#187)` — squash `445240b`
  - 5 コミット（feat IAM / refactor user_data / chore SSH 削除 / style fmt / fix B-04 + W-01 / fix B-05 + B-06）
  - 8 ファイル変更
- **PR #153** `infra(terraform): enable Docker awslogs driver + CloudWatch Log Group (#188)` — squash `420f0ea`
  - 3 コミット（feat IAM + LogGroup / feat user_data awslogs / docs README）
  - 5 ファイル変更（cloudwatch.tf 新規 + iam.tf / ec2.tf / user_data.sh.tpl / README.md）

#### dev-journal（7 コミット）

- `e11f90e` env_config.md §5.3.3 awk → jq + §5.3.5 IAM 差分注記（#187 B-01/B-02）
- `ac9e476` #187 計画書 v2 アーカイブ
- `db79487` env_config.md §5.3.5 kms:Decrypt 削除注記（#187 B-05）
- `b6d4a40` issue #187 クローズ
- `1d1352b` monitoring.md / env_config.md / runbook.md / release.md / architecture.md awslogs 同期更新（#188 T5-T9）
- `2e6c327` #188 計画書 v2 アーカイブ
- `86228e5` issue #188 クローズ

#### 起票・解決した issue / review-findings

- **クローズ済み**: issue #187 / #188（両方とも resolved）
- **新規起票**: なし
- **review-finding**: 内部レビュー指摘 / codex 指摘とも全て PR 内コメントで処理（review-findings ファイル化なし）

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-25.md`（2026-05-22 開始〜05-25 にかけて: issue #186 設計書 ECS→EC2 化を完全クローズ、連動 issue #187 / #188 を起票）

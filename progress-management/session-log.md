# 引き継ぎメモ

## セッション: 2026-05-19 (主に夜、~20:59)

### ゴール

- Step 11-E Phase 3 で発生した RDS backup_retention 緊急 fix の正式化（PR 化）
- NFR-AVAIL-003 の設計修正（実装値「1 日保持」と整合）
- issue #181 を resolved まで進める
- Phase 4 着手は次セッションへ持ち越し

実際は **Step 1 + Step 3（設計修正） + 派生 issue #182 起票**まで完走。Phase 4-7 は未着手。

### 作業ログ

#### 1. Step 1: 緊急 fix の正式化（expense-saas PR #146）

- 前回セッションで未コミットだった `rds.tf` (`backup_retention_period = 7 → 1`) と `.terraform.lock.hcl` (前回 init 後コミット漏れ) を機能ブランチ `step11/181-rds-backup-retention-fix` で正式化
- 軽量化案で進行: チケット起票 / implement エージェント / `/commit` スキルを省略、指揮役が直接 feature ブランチで生 git commit
- ローカル CI: `terraform fmt -check -recursive` PASS。`validate` は devcontainer egress allowlist で `registry.terraform.io` がブロックされ実行不可（後に issue #182 として起票）
- 内部 reviewer: **PASS**（blocker 0、warning 0、info 1）。info（PR Summary に設計修正別 PR 予定を明記）を取り込み
- codex レビュー: **COMMENT（blocker なし）**、`gh pr review --comment` で投稿
- squash マージ完了: merge commit `fee0a2c`

#### 2. Step 3: NFR-AVAIL-003 設計修正（dev-journal）

ユーザー判断:
- **「本番運用予定がない」**→「7 日に戻す前提は不要」
- 設計修正方針 **「NFR 自体を 1 日保持に書き換え」**（issue #181 提案案 3）

修正対象を grep → 5 ファイル / 8 行を初回特定 → designer エージェントに 8 箇所修正を依頼:
- `requirements.md:444`（NFR 定義の正本）
- `architecture.md:488`（トレーサビリティ）
- `adr/0004-infra.md:42, 94`（RDS 選定根拠）
- `traceability.md:234`（テスト紐付け）
- `backup_restore.md:37, 106, 143, 157`（運用設計）

designer から **指示外 7 箇所の「7 日間」残存** を指摘される（私の初回 grep が「7日間」単独表記を取りこぼし）。残存箇所を再 grep:
- `adr/0004-infra.md:71`（決定表）
- `backup_restore.md:17, 59, 142, 144, 167`
- `env_config.md:60`

**ユーザー判断: 案 Z（同一文書内整合を優先）** を採用:
- ADR 決定表 + backup_restore.md の自動バックアップ系（17/59/142/167）→ 1 日に修正（誤読防止）
- `backup_restore.md:144`（手動スナップショット）+ `env_config.md:60`（stg/prod）→ 業務 SaaS 想定として **7 日のまま維持**

合計 14 箇所修正 + issue #181「解決内容」セクション記入完了。

#### 3. 内部 reviewer + codex レビュー

- 内部 reviewer: **PASS**（blocker 0、warning 2、info 2）
  - W-01 `env_config.md:60` 文脈注記不足
  - W-02 `backup_restore.md:144` 別概念明示不足
  - I-01 `adr/0004-infra.md:94` 表現微調整
  - I-02 issue #181 解決内容 → 指摘なし
- W-01 / W-02 / I-01 全部対応（各 1 行追記 / 微調整）
- 軽量化案で再 reviewer をスキップ、codex に直接渡す
- dev-journal コミット `f10f88a`
- codex レビュー: **FIX 判定** → review-finding 120 起票
  - 指摘内容: `rds.tf:55-57` のコメントがまだ「7 日間保持からの逸脱」「post-MVP に追跡」のままで、NFR が 1 日に緩和された解決後の正本と矛盾

#### 4. PR #147: rds.tf コメント追従更新

- 機能ブランチ `step11/120-rds-comment-update` で `rds.tf:55-56` コメントを「portfolio 仕様として 1 日保持に緩和済み（issue #181 解決済み）」に更新
- 軽量化: 内部 reviewer スキップ、codex のみで再レビュー
- codex 再レビュー: **APPROVE 相当**（self-approve 不可で `--comment` 投稿）
- squash マージ完了: merge commit `b53905e`

#### 5. dev-journal 後処理

- `issues/open/181-*.md` → `resolved/` 移動 + 解決内容に PR #147 追記
- `review-findings/open/120-*.md` → `resolved/` 移動
- dev-journal コミット `d175c83`

#### 6. 派生 issue #182 起票 + progress.md 更新

- `terraform validate` 実行不可問題を **issue #182** として起票
  - devcontainer egress allowlist に `registry.terraform.io` が含まれず Forbidden
  - 影響度 低、post-MVP、issue #179 系統
- `progress.md` 更新: issue #181 → resolved、issue #182 追加
- dev-journal コミット `a789b87`

### 完成した PR / 設計修正

| PR / Commit | 内容 |
|---|---|
| PR #146 (`fee0a2c`) | rds.tf 7→1 + .terraform.lock.hcl |
| PR #147 (`b53905e`) | rds.tf コメント追従更新 |
| dev-journal `f10f88a` | 設計修正 14 箇所 + 注記 3 箇所 |
| dev-journal `d175c83` | issue #181 / finding 120 resolved 移動 |
| dev-journal `a789b87` | progress.md 更新 + issue #182 起票 |

### 未完了

- **EC2 user_data の env ファイル更新**（前回セッション持ち越し、未着手）
  - `13.158.141.63` に SSH ログイン → `sudo grep -r "placeholder.invalid\|CORS" /etc/expense-saas/ /opt/expense-saas/` で確認
  - 古い値が残っていれば手動更新 or `terraform taint aws_instance.app` + apply で強制再作成
- **Phase 4-7**（DB bootstrap / Docker / systemd / smoke）全て未着手

### ブロッカー

なし

### 次にやること

#### 優先度 1: EC2 user_data 環境変数の確認・更新（前回セッションから持ち越し）

```bash
ssh -i ~/.ssh/expense-saas-portfolio.pem ec2-user@13.158.141.63
sudo grep -r "placeholder.invalid\|CORS" /etc/expense-saas/ /opt/expense-saas/ 2>/dev/null
```

#### 優先度 2: Step 11-E Phase 4（DB bootstrap）

11-E チケット §5.7.3 の Step 1〜7 を実施（前回セッション引き継ぎから変わらず）

#### 優先度 3: Phase 5〜7（Docker / systemd / smoke）

工数見積: 5〜10 時間（前回見積から変わらず）

### 学び・気づき

#### 設計修正の grep は表記揺れを網羅する

NFR-AVAIL-003 関連の grep を `"NFR-AVAIL-003\|backup_retention\|バックアップ.*保持\|7.*日.*保持"` で実行 → 「7日間」単独表記 7 箇所を取りこぼした。designer エージェントが指示外の漏れを指摘してくれて発覚。

→ 設計書の用語修正では「対象用語の全パターン（半角/全角スペース、語幹単独、複合）」を grep する。grep パターンは最初に複数試して、件数差を見て網羅性を担保する。

#### 設計書には「業務 SaaS 想定」と「portfolio MVP 現実値」の二重構造がある

env_config.md の dev/stg/prod 列、backup_restore.md の手動スナップショット行など、「業務 SaaS の理想形」を残している箇所がある。今回 NFR を緩和する際、これらをすべて 1 日に統一する案（案 Y）と、同一文書内整合のみ取って業務想定箇所は残す案（案 Z）が分岐した。

→ 設計書は「業務 SaaS の要件」と「portfolio 実装の現実」の二重構造として保持し、矛盾する箇所には注記を追加する。完全統一は portfolio として将来性を捨てる選択になる。

#### codex は「実装側コメントとの整合性」までトレースする

dev-journal の設計修正に対する codex レビューが、`expense-saas/infra/terraform/rds.tf:55-57` のコメント（「逸脱」「post-MVP に追跡」）まで追跡し、設計の正本と矛盾していることを指摘。**14 箇所の設計修正自体は PASS**、コメントだけが不整合という細かい検出。

→ 設計修正と実装コメントは一緒に追従させる。「正式化 PR を切る」段階で旧コメントが残っていないか必ず確認する。

#### コミットスキルの「master のみ」ルールと feature ブランチ作業の摩擦

`commit` スキルには「master 以外のブランチではコミット中止」という安全装置がある。これは「指揮役は通常 master でコミット、feature ブランチは implement エージェント（worktree）の責務」という運用前提に基づく。今回のように指揮役が極小修正で feature ブランチを直接扱う場合、`/commit` スキルは使えない。

→ 指揮役が feature ブランチで直接 commit する場合は、生 git で実行する（過去ログで定着）。`commit` スキルの安全装置を頼らない場面では、Co-Authored-By フッターを忘れずに付ける。

#### Free Tier 制約問題は「実装 → 設計」の逆フローで解消

前回セッションでは「設計書 NFR 7 日 → 実装側を 1 日に緊急 fix」という逆向きの逸脱だった。本セッションでは「実装値（1 日）に合わせて設計を緩和」して整合性を回復。portfolio プロジェクトでは「実装で現実が判明 → 設計を追従」の方が現実的なケースがある。

→ 制約由来の逸脱は、当該プロジェクトの位置付け（業務向け / portfolio 向け）に応じて「実装側を直す」か「設計側を直す」かを判断する。portfolio なら設計緩和が自然。

### 意思決定ログ

#### NFR 修正方針: 案 3（NFR 自体を 1 日に書き換え）

- issue #181 の提案 3 案のうち、ユーザー判断 **「本番運用予定がない」** から案 3 を採用
- 案 1（AWS Support 申請）/ 案 2（有料アカウント化）は不採用
- 結果として「残置事項なし」で issue #181 を closed まで進められた

#### 設計書修正範囲: 案 Z（同一文書内整合を優先）

選択肢:
- 案 X: 直接引用箇所 9 のみ修正 → 同一ファイル内に矛盾発生（NG）
- 案 Y: 残存 7 箇所すべて 1 日に修正 → 業務 SaaS の理想形が消える
- **案 Z**: 同一ファイル内整合を取りつつ業務 SaaS 想定箇所（env_config.md stg/prod、backup_restore.md 手動スナップショット）を 7 日のまま維持
- 採用理由: 業務 SaaS としての設計書価値を保ちつつ、誤読リスクを注記で解消できる

#### 軽量化案の繰り返し採用

今セッションは以下を軽量化:
- Step 1（PR #146）: チケット起票 / implement / `/commit` スキルを省略、指揮役直接コミット
- Step 3 設計修正: architect の計画策定をスキップ、designer 直接依頼（14 箇所のメカニカル修正）
- 設計修正の reviewer 再レビュー: warning 対応後の再 reviewer を省略、codex に直接渡す
- PR #147: 内部 reviewer をスキップ、codex のみ
- 採用理由: 修正規模が極小（1 行 / 8 行 / コメント 1 行）、ユーザー方針として「正規フローの過剰適用を避ける」

#### review-finding 120 への対応: PR 化（軽量化はせず）

`rds.tf` コメント 1 行修正だが、PR フローで対応:
- 採用理由: master 直接コミットは workflow.md ルール違反、コメント修正もコードレビュー対象
- 結果: PR #147 で 1 行修正 + codex 再レビュー APPROVE → squash マージ

### PR / コミット要約

**expense-saas**（PR 2 件、master 更新済み）:
- PR #146 (`fee0a2c`): rds.tf 7→1 + .terraform.lock.hcl
- PR #147 (`b53905e`): rds.tf コメント追従更新

**dev-journal**（コミット 3 件）:
- `f10f88a`: 設計修正 14 箇所 + 注記 3 箇所
- `d175c83`: issue #181 / review-finding 120 resolved 移動
- `a789b87`: progress.md 更新 + issue #182 起票

**root-project / ai-dev-framework**: 変更なし

**起票 issue（全 post-MVP）**:
- #182 devcontainer egress allowlist に registry.terraform.io が含まれず terraform validate 実行不可

**解決 issue**:
- #181 RDS backup_retention_period が Free Tier 制約により NFR-AVAIL-003 から逸脱 → resolved

### Phase 状態サマリ（次セッション着手時の前提）

| Phase | 状態 | 担当 |
|-------|------|------|
| Phase 0-3 | **完了**（前セッションまで） | - |
| NFR-AVAIL-003 緩和（後追い設計修正） | **完了**（今セッション） | - |
| Phase 4 DB bootstrap | 未着手（user_data env 確認が前提） | USER + 指揮役支援 |
| Phase 5 Docker build + systemd | 未着手 | USER |
| Phase 6 ヘルスチェック | 未着手 | 指揮役 |
| Phase 7 スモーク + 時刻ドリフト | 未着手 | USER（操作）+ 指揮役（観察）|

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-18.md`（2026-05-18 〜 2026-05-19 00:30: Phase 2 + Phase 3 完了、26 リソース AWS apply、AWS アカウント作成 + 3DS トラブル、RDS Free Tier 制約による緊急 fix）

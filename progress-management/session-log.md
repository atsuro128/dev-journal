# 引き継ぎメモ

## セッション: 2026-06-05 12:57

### ゴール

issue #197（AWS公開デモの lean 化）を **方針再決定 → 実装 → 本番 apply → 障害 → 復旧** まで一気通貫で完遂する。「動作検証（料金含め）」まで実施。

### 作業ログ

#### 方針再決定（長い対話で確定）

- 前回（第1セッション）の「Go ミドルウェアで X-Origin-Verify 移設 + 手動 destroy/apply」から方針転換。
- 確定方針: **ALB除去 / CloudFront直結（HTTPS終端・独自ドメイン不要）/ オリジン保護を SG 一枚（CloudFront prefix list）に簡素化（X-Origin-Verify 廃止、安全網はアプリのレート制限 login 5/min 等）/ Go 不介入 / EventBridge 深夜 stop/start**。
- 対話の論点: 「常時公開 vs 見せる時だけ」「CDN/HTTPS/オリジン保護の一般性」「独自ドメイン有無」「stop/start と destroy の違い」「EIP は停止中も課金」等をユーザーと議論して確定。

#### 実装（設計成果物フロー + PR フロー）

- architect 計画 → reviewer 検証（FIX）→ ユーザー判断（security.md 本文更新 / 移行ダウンタイム許容）。
- platform-builder（Terraform・PR #159）と designer（dev-journal 設計成果物）を**並列**実行 → reviewer 両方 PASS。
- codex 指摘 3 件（systemd start limit / IAM EC2 ARN アカウント固定 / ADR-0004 旧750h記述）を妥当と判断し対応 → codex APPROVE。
- PR #159 マージ（32a8112）。dev-journal は 4 コミット（設計成果物・issue#197更新・ADR-0004修正・review-finding 127 resolved）。

#### 環境整備（Windows ネイティブ）

- worktree フックが devcontainer 用 `/root-project` ハードコードで動かず → **`$CLAUDE_PROJECT_DIR` 化 + 自己位置解決 + cygpath で Windows 形式変換**（root-project コミット 79c22d2）。
- `jq` / `terraform 1.9.8` / `aws cli`（pip）をローカル導入（~/bin・git 管理外）。

#### 本番 apply（障害 → 復旧）

- **1回目失敗**: IAM role description に日本語（非ASCII制約違反）/ EC2 SG が `create_before_destroy` 無しで `DependencyViolation`。→ description を ASCII 化、EC2 SG を `name_prefix + create_before_destroy` 化で修正。
- **2回目成功**（create_before_destroy で SG 置換順序が解決）。だが **EC2 が replace され docker イメージ消失 + user_data の `image_tag=""` 分岐でアプリ起動スキップ → CloudFront 502/504**。
- **復旧**: ローカルに残っていた `expense-saas:portfolio` イメージを `docker save`+gzip → 本番 S3 の `_temp/` へ → SSM `send-command` で EC2 が `docker load` + `systemctl enable --now` → **CloudFront 経由 200 復旧・DB 接続 OK・オリジン保護（直 EIP:8080 遮断）も確認**。

#### 追加修正

- scheduler の **cron timezone バグ**発見（`schedule_expression_timezone=Asia/Tokyo` なのに cron が UTC 前提で書かれ、意図と異なる時刻に動作）→ JST 直書きに修正。停止 **JST 00:00 / 起動 08:30** に変更（コミット 6dd814d）。
- README に「EC2 replace を伴う apply の後はアプリ再デプロイ必須」の注意を追記（7c3855c）。

#### コスト確認（料金含め）

- Cost Explorer 実データで **ALB が月 ~$13-16 かかっていた**ことを確認（今日削除で今後消える）。lean 稼働 ~$36/月、深夜 stop/start でさらに削減。**自腹はクレジット相殺で $0 継続**。実請求への反映は来月から。

### 未完了

- なし（issue #197 のコード・設計・apply・復旧・コスト確認まで完了）。ただし下記「次にやること」の明朝確認が残る。

### ブロッカー

なし（本番は lean 構成で 200 稼働中）。

### 次にやること

1. **明朝（JST 08:30 以降）に深夜 stop/start の初回サイクル後、CloudFront 経由 200 を確認**。EC2 の stop/start は電源 off/on でイメージ保持されるため自動復帰する想定だが、**初回は要確認**（万一起動しない場合は SSM で `systemctl restart expense-saas`）。
2. 公開URL: `https://djhmwtrr79jdq.cloudfront.net/`（lean 構成で稼働）。

### 学び・気づき（重要）

1. **EC2 replace を伴う apply は「アプリ再デプロイ」が必須**。今回 lean 移行で EC2 が再作成されイメージが消え本番停止。**計画（architect）・reviewer・codex すべてが見落とした**。`terraform plan` で `aws_instance.app` が `must be replaced` と出たら再デプロイ（`tickets/step11/11-E-deploy.md`：docker save→S3→SSM load）を想起する観点を持つ。デプロイ方式は ECR でなく S3 持ち込み。
2. **本番への破壊的 apply 前に影響を十分に洗い出すべきだった**。本番を長時間停止させた。plan の replace 行を軽視しないこと。
3. **issue 起票は「対応すべき作業」がある時に限る**。当初 #198 を影響度「高」で起票したが、デプロイ手順は既存（11-E-deploy.md）・復旧実証済み・発生稀のため過剰と判断しユーザーと合意の上で削除。代わりに README 1行注意 + 本 session-log の教訓で対応。
4. **機密情報をコミットや session-log に書かない**（ユーザー指示）。`terraform.tfvars.bak` のような機密バックアップが gitignore 外に残らないよう注意（今回 sed 編集時に作り、検知して削除）。
5. **timezone 指定時の cron はその timezone で書く**。PR #159・reviewer・codex 全てが cron timezone バグ（Asia/Tokyo 指定 + UTC 前提 cron）を見逃した。

### 意思決定ログ

#### オリジン保護を SG 一枚に簡素化（X-Origin-Verify 廃止）

- ALB 除去で X-Origin-Verify 検証（ALB リスナールール）が消える。Go 移設せず SG（CloudFront prefix list）一枚に簡素化。残存リスク「他人の CloudFront 踏み台」はダミーデータ・標的型限定・アプリのレート制限（login 5/min 等）で受容。判断は ADR-0007 に記録。

#### 深夜停止は EC2/RDS の電源 stop/start（destroy ではない）

- 「毎日やるなら destroy/apply（RDS 20分・データ消失）でなく stop/start（1-2分・データ保持）」と整理。EventBridge Scheduler で JST 00:00 停止 / 08:30 起動。ALB は stop 不可のため除去が前提（残すと停止しても課金継続）。

#### lean は維持しつつ保護は一般手法（ユーザーの「特有デザイン回避」意向）

- 「ポートフォリオ特有の構成にしたくない」意向に対し、オリジン保護（カスタムヘッダ検証）は CloudFront/Cloudflare 公式推奨の一般手法、ALB 除去はコスト都合の割り切り、と切り分け。lean 維持 + 保護は標準手法（SG + レート制限）で決着。

### 起票 issue

- なし（#198 は起票後に過剰と判断し削除。教訓は本 session-log + README 注記へ）

### PR / コミット要約

- **expense-saas**: PR #159 マージ（32a8112）/ fix 6dd814d（IAM description ASCII・create_before_destroy・rds fmt・cron JST修正）/ docs 7c3855c（README 再デプロイ注意）
- **dev-journal**: 設計成果物・issue#197更新・ADR-0004修正・review-finding 127 resolved の 4 コミット
- **root-project**: 環境修正 79c22d2（worktree フックの環境非依存化）push 済み

### AWS リソース変更（本番）

- ALB 一式削除 / EIP 新設 / CloudFront origin を EIP:8080 へ差替 / EC2 SG を CloudFront prefix list 限定 / EventBridge stop/start（JST 00:00 停止・08:30 起動）/ RDS skip_final_snapshot=true。
- **本番は lean 構成で 200 稼働中**。RDS データは保持。

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-06-04.md`（2026-06-04 16:04。issue #197 起票・方針決定・計画/レビューまで。実装は本セッションで実施）

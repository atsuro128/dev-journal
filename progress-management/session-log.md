# 引き継ぎメモ

## セッション: 2026-05-25 01:56

（作業は 2026-05-22 開始、05-23 〜 05-25 にかけて連続実施）

### ゴール

- 「次の作業は？」から /session-start で前回引き継ぎを確認。issue #186（設計書のコンピュート層 Fargate 表記が EC2 実装と乖離）の対応を進める。
- → **達成。issue #186 を完全クローズ（resolved）。連動して issue #187 / #188 を新規起票。**

### 作業ログ

1. **スコープ確定フェーズ（2026-05-22 〜 05-23）**:
   - issue #186 当初想定は「軽微 3 ファイル是正」だったが、ユーザーとの議論で 3 案（A: 注記統一 / B: 全面 EC2 化 / C: architect 委任）→ 最終案D「EC2 を正本、Fargate は ADR-0004 だけに残す」を採用。
   - 全出現箇所マップ調査で、3 ファイル想定を超えて 10 ファイル + 詳細メトリクス（monitoring）+ 詳細運用手順（runbook / release）への波及を確認。カテゴリ X/Y/Z に分類。
   - 「Fargate は実装しない（する予定もない）から、寧ろ EC2 の方が詳しく書くべき」というユーザー指摘で案A を案D に転換。
2. **Phase 0 設計確定（2026-05-24）**:
   - architect 実装計画 v1 策定 → reviewer CONDITIONAL PASS（blocker 1 + warning 6 + info 5）→ v2 で全反映 → reviewer 完全 PASS。
   - ユーザー判断 7 項目（UD-1〜UD-7）確定:
     - UD-1=A SSM Session Manager（自宅 Wi-Fi IP 変動への耐性で SSH→SSM に変更、別 issue #187 起票連動）
     - UD-2=B Docker awslogs ドライバ / UD-3=A CWAgent + AWS/EC2 併用 / UD-4=A EC2-StatusCheckFailed 追加
     - UD-5=B SSM Parameter Store / UD-6=A ECR pull / UD-7=A 並列実行
   - 連動 issue #187（SSM Session Manager + SSM Parameter Store Terraform 実装）を起票。
   - コミット: `db53b03 docs(issue-186): 案D 設計確定（Phase 0 ユーザー判断 7 項目）と issue #187 起票`
3. **Phase 1-4 設計成果物書き直し（2026-05-24）**:
   - designer 5 並列起動: Phase 1 シリアル束（T1/T2/T4/T5/T6/T11 = 6 ファイル）/ T7 monitoring / T8 env_config / T9 release / T10 runbook
   - Phase 4 シリアル: T3 diagrams 構成図差し替え（他成果物確定後に Mermaid 図更新）
   - reviewer 内部レビュー PASS（受け入れ基準 11/11 達成、横断整合 OK、副作用なし）
   - reviewer info I-01「monitoring.md ロググループ命名 `/ecs/expense-saas/api` 残置」→ 別 issue #188 起票
   - コミット: `80dc4f4 docs(issue-186): Phase 1-4 designer 完了 — 11 ファイルの ECS 記述を EC2 実装と整合（UD-1〜7 反映 + issue #188 起票）`
4. **Phase 5 codex レビュー対応（2026-05-25）**:
   - codex 初回レビューで finding #122/#123/#124（重大度 高 2 件 + 中 1 件）起票:
     - #122: 存在しない `expense-app` unit/container 名参照（実装は `expense-saas`）
     - #123: ECR pull 後の `docker tag` 抜けで systemd 起動不能（systemd unit は固定 local tag `expense-saas:portfolio` 参照）
     - #124: release.md §3.3 で `Secrets Manager`、§4.1 で `bastion`、runbook で未定義 `/expense-saas/db_password` 残置
   - designer 1 エージェントで一括対応 → user_data.sh.tpl と完全整合
   - コミット: `92650b5 fix(issue-186): codex finding #122/#123/#124 対応（user_data.sh.tpl と整合）`
   - codex 再レビュー PASS（Findings なし）→ 3 件とも resolved/ 移動
5. **クローズ（2026-05-25）**:
   - issue #186 §解決内容 / §解決日 記入 → `open/` → `resolved/` 移動
   - progress.md 状態更新（「対応中」→「resolved」）
   - コミット: `32c9f69 chore(issue-186): クローズ — codex 再レビュー PASS、resolved 移動 + progress 更新`

### 未完了

- **issue #187**（SSM Session Manager + SSM Parameter Store の Terraform 実装）— 起票のみ、着手は Step 11-E 実デプロイ時想定
- **issue #188**（monitoring.md ロググループ命名 `/ecs/expense-saas/api` リネーム）— 起票のみ、post-MVP（実装側 awslogs-group 設定の同期更新必要）
- **前回からの持ち越し（未着手）**:
  - issue #185 T4（`CORS_ALLOWED_ORIGINS` を CloudFront ドメインに、`TRUSTED_PROXY_COUNT=2` を prod に投入）
  - issue #185 T6（CloudFront 経由の疎通・CORS・HSTS・レート制限検証）
  - Step 11-E Phase 7（スモークテスト: seed 投入 + 申請→承認→支払 golden path）
  - issue #184 ファイルの housekeeping（前回 progress.md は resolved 反映済みだが、ファイル自体は前セッションで resolved/ 移動済み）
  - expense-saas の worktree クリーンアップ

### ブロッカー

なし

### 次にやること

優先順位:

1. **Step 11-E デプロイ継続作業**（実 AWS 操作、ユーザー主導）:
   - issue #185 T3 apply（CloudFront 2 段階 apply）→ T4 env 反映 → T6 疎通検証
   - issue #187 の Terraform 実装（SSM Session Manager + SSM Parameter Store）を同セッションで同期実施するのが効率的（#186 設計成果物が #187 の仕様書として機能する）
2. **Step 11-E Phase 7 スモークテスト**（seed 投入 + 申請→承認→支払 golden path）
3. **Step 11-F UAT 着手**（MVP 完成判定）
4. **post-MVP issues 整理**:
   - issue #188（monitoring ロググループ命名）
   - その他 post-MVP open issues（060, 061, 064, 081, 084, 104, 122, 145, 146, 151, 167, 174, 176-180, 182 + ops-055/062/080）

### 学び・気づき

- **codex レビューが reviewer の盲点を補完した実例**: reviewer 内部レビュー PASS 後に codex が「実装側 `user_data.sh.tpl` との具体整合（unit 名・コンテナ名・タグ付け方式・パラメータ名）」を 3 件指摘。reviewer は受け入れ基準 grep と用語置換は通したが、「実行可能性」までは踏み込まなかった。**memory `feedback_critical_review_of_codex` の運用例として、codex 指摘は形式ではなく実害を見て判定すべきという原則を再確認**（今回は 3 件とも妥当な指摘）。
- **designer プロンプトに「実装側ファイルを必ず読ませる」指示の重要性**: 私が初回 designer 起動時に `user_data.sh.tpl` の値（`expense-saas.service` / `--name expense-saas` / `expense-saas:portfolio`）を確認せず推測で `expense-app` と書いたため、3 件の codex 指摘が発生。**実装と整合する設計書を書かせるなら、designer に該当実装ファイルを Read させるプロンプトを徹底する**。
- **「読み替え注記」案A vs「正本書き換え」案D のトレードオフ**: 案A（注記統一）は ADR の書き分け美しさを保つが、「動かない手順書」が残る。ポートフォリオ用途では「動く手順書」の方が価値が高い（codex / 第三者レビューでの突っ込みも減る）。ユーザー指摘で案D に転換した判断は良かった。
- **AskUserQuestion の使い方**: 7 件のユーザー判断を 4 件 + 3 件に分けて提示したが、初回はユーザーが用語を理解できず「全部解説して」と返ってきた。**技術用語の判断項目を出す前に、文脈・選択肢の意味・実害を先に説明する手順を踏むべき**。
- **タスクトラッキング**: 17 タスク（#1〜#17、Phase 0-5 + クローズ）を依存関係付きで管理。並列起動・順序制御に有効だったが、サブタスク化のタイミング（特に findings 対応の #15/#16/#17 を後追いで追加）が遅れたのは反省。最初から「設計確定 → Phase 1-4 → codex → クローズ」の全フローを想定してタスク作成すべきだった。

### 意思決定ログ

確定した設計判断（issue #186 §ユーザー判断項目に詳細記録）:

- **案D 採用**: ADR-0004 を「Fargate 採用 + §ポートフォリオ対応で EC2 ピボット」の歴史記録として不変、下流 10 ファイルは EC2 を一次正本として書き直す。Fargate 言及は ADR-0004 への参照に集約。
- **UD-1 = A（SSM Session Manager）**: 自宅 Wi-Fi の IP 変動に対し SSH では SG 更新が常時発生するため、SSM 採用が合理的。連動して #187 起票。
- **UD-2 = B（Docker awslogs ドライバ）**: 追加エージェント不要、systemd unit の `docker run` オプションのみで完結。
- **UD-3 = A（CWAgent + AWS/EC2 併用）**: CWAgent はメトリクスのみモード（ログは awslogs 担当）。メモリ・ディスク使用率を取得し ALERT-W3（メモリ 80% 超）維持。
- **UD-4 = A（EC2-StatusCheckFailed 追加）**: 単一 EC2 構成のため早期検知価値高い。
- **UD-5 = B（SSM Parameter Store）**: KMS 暗号化・無料・ローテーション容易。#187 で UD-1 と統合実装。
- **UD-6 = A（ECR pull）**: Step 11-E Phase 5 で既にこの方式でデプロイ済み、実装実態と整合。t3.micro 上のビルドは過食リスク回避。
- **UD-7 = A（並列実行）**: Phase 1 のみシリアル、Phase 2-3 の重 4 タスク並列、Phase 4 最後にシリアル。

### PR / コミット要約

- **dev-journal**: 4 コミット
  - `db53b03` Phase 0 設計確定（ユーザー判断 + issue #187 起票）
  - `80dc4f4` Phase 1-4 designer 完了（11 ファイル + issue #188 起票）
  - `92650b5` codex finding #122/#123/#124 対応（user_data.sh.tpl と整合）
  - `32c9f69` クローズ（解決サマリ + resolved 移動 + progress 更新）
- **expense-saas**: 変更なし（参照のみ）
- **起票 issue**: #187（SSM Terraform 実装、open）/ #188（monitoring ロググループ命名、open）
- **review-finding**: 122 / 123 / 124（resolved）

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-21.md`（2026-05-20 開始〜05-21 にかけて: issue #185 ALB HTTPS 化対応で T1 / T3 / T5 + #184 を完了、T4 / T6 は実 AWS デプロイ時のアクティビティとして残置）

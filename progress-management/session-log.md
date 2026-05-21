# 引き継ぎメモ

## セッション: 2026-05-21 18:43

（作業は 2026-05-20 開始、05-21 にかけて実施）

### ゴール

- issue #185（ALB が HTTP 平文・HTTPS 未対応）の対応。あわせて #184（SPA fallback の HEAD 405）も同セッションで対応。
- → **達成。#185 の T1 / T3 / T5 と #184 を完了。T4 / T6 は実 AWS デプロイ時のアクティビティとして残置。**

### 作業ログ

1. **計画フェーズ**: architect が #185 実装計画 v1 → reviewer FIX（blocker 3）→ v2 → reviewer 再レビュー条件付き PASS。設計判断 B-1〜B-5 をユーザー確定（下記「意思決定ログ」）。
2. **T1（`remoteIP` の TRUSTED_PROXY_COUNT 化）** → PR #149。内部レビュー warning W-1（config テスト未追加）対応、codex blocker（`rate_limit_test.go` のシグネチャ追従漏れ＝integration テストのコンパイルエラー）対応 → スカッシュマージ（master `c16f9d7`）。
3. **#184（SPA fallback の HEAD 対応）** → PR #150。codex blocker（`r.Head("/*")` catch-all で `HEAD /health` が SPA fallback に流れ DB 異常時も index.html 200 を返す false positive）対応 = `HEAD /health` をヘルスハンドラ経由に。PR #148 由来の既存 lint（embed.go の staticcheck SA9009 誤検知）を drive-by 修正 → マージ（master `dd480b5`）。
4. **T3（CloudFront 構成 + B-1-b 完全閉域）** → PR #151。内部レビュー blocker 2（heredoc 退行 / 適用順序）→ codex blocker（2 段階 apply の `-target` 依存解決破綻 → `restrict_alb_to_cloudfront` 変数で SG 制限を段階制御）→ codex blocker（B-5: デフォルト証明書の viewer TLS 最小 TLSv1）→ 受容判断 → マージ（master `4812a01`）。
5. **T5（設計書整合）** → ADR-0007 起票、11-E チケット訂正（§3.5 参照切れ・ADR-0004 参照ミス・§11 Q2 追補ノート）、architecture.md / diagrams.md / env_config.md の CloudFront 反映。dev-journal commit `1925081`。内部レビュー PASS（warning W-1 → issue #186 起票）。codex 設計レビュー FIX（review-finding 121）→ 11-E 完了判定・11-F 引き継ぎを CloudFront ドメイン基準に修正 → commit `2155854` → codex 再レビュー PASS、finding 121 resolved。
6. ユーザーが Backend ローカル CI（BE: full test）を master `c16f9d7` で実行 → unit / integration 全 PASS、lint は embed.go 1 件（PR #150 の drive-by で修正済み）。

### 未完了

- **#185 T4**（`CORS_ALLOWED_ORIGINS` を CloudFront ドメインに、`TRUSTED_PROXY_COUNT=2` を prod に投入）— 実 AWS デプロイ時に実施
- **#185 T6**（CloudFront 経由の疎通・CORS・HSTS・レート制限検証）— 実 AWS デプロイ時に実施
- **Step 11-E Phase 7**（スモークテスト: seed 投入 + 申請→承認→支払 golden path）— 前回からの持ち越し、未着手
- issue #184 ファイルが `issues/open/` のまま（progress.md は resolved 反映済み）。次セッションで解決内容記入 → `resolved/` 移動の軽微な housekeeping
- expense-saas の worktree が複数残存（後述）

### ブロッカー

なし

### 次にやること

1. **#185 の実デプロイ（T3 apply + T4 + T6）= Step 11-E のデプロイ継続作業**:
   - CloudFront を **2 段階 apply**: `restrict_alb_to_cloudfront=false` で apply（CloudFront + ALB 作成、SG 開放だがカスタムヘッダで保護）→ CloudFront が `Deployed` になるのを確認 → `restrict_alb_to_cloudfront=true` で apply（SG を CloudFront プレフィックスリスト限定）
   - `cloudfront_origin_verify_secret`（sensitive）を tfvars に設定
   - apply 後、確定した CloudFront ドメインを `CORS_ALLOWED_ORIGINS` に反映（apply 後に実値設定する 2 段階デプロイ）、`TRUSTED_PROXY_COUNT=2` を prod に投入（T4）
   - T6 疎通検証は 11-E チケット §6（CloudFront 基準に更新済み）の手順に従う。ALB DNS 直アクセスは 403 が期待値
2. **Step 11-E Phase 7**（スモークテスト）
3. **issue #186**（architecture.md/diagrams.md/env_config.md のコンピュート層 Fargate 表記 → EC2 是正、post-MVP）
4. worktree クリーンアップ

### 学び・気づき

- **ローカル /test スキップのリスクが顕在化**: T1 でユーザー判断により /test をスキップしたが、codex が integration テスト（`rate_limit_test.go`、integration タグ付き）のシグネチャ追従漏れ＝コンパイルエラーを捕捉。`go build` / 個別パッケージテストでは検出できず、`go test -tags integration ./...`（BE: full test）のみが拾える類だった。/test が正規ゲートである理由の実例。
- **GitHub Actions は PR で走らない**（無料枠方針、`ci.yml` は存在するが `gh pr checks` は "no checks reported"）。「GitHub Actions を CI 証跡にする」という前提は誤り。テスト証跡はローカル /test が正本。
- **多層レビューが機能**: reviewer が見落とした構造的問題（B-5 の TLSv1 固定、T3 の `-target` 依存解決破綻）を codex が複数捕捉。reviewer → codex の二段が有効に働いた。
- worktree が多数残存。指揮役が修正対応で非 isolation エージェントを既存 worktree に向けて起動した際、ハーネスが空 worktree を別途生成するパターンを複数回踏んだ。

### 意思決定ログ

確定した設計判断（すべて ADR-0007 に記録）:

- **B-1 = B-1-b（完全閉域）**: ALB SG を CloudFront マネージドプレフィックスリスト限定 ＋ `X-Origin-Verify` カスタムヘッダを ALB リスナールールで検証（不一致 403）。他者の自前 CloudFront 経由も遮断。SG 制限は `restrict_alb_to_cloudfront` 変数で段階制御（CloudFront 作成完了前に SG を絞ると ALB 全遮断になるため。`-target` では依存リソースを巻き込み順序保証できないため変数方式を採用）。
- **B-2 = B-2-c**: `remoteIP` を `TRUSTED_PROXY_COUNT` 方式に。実クライアント IP = `XFF[len - TRUSTED_PROXY_COUNT]`、prod=2、dev=0。
- **B-3 = 受容**: CloudFront〜ALB 間が HTTP（security.md §441 乖離）。$0 維持のため受容、ADR-0007 に記録。
- **B-4 = 追補ノート**: 11-E §11 Q2 の「案1 採用」記述は履歴として残し、追補ノート + 【現行構成】/【判断履歴】の見出し分離で ADR-0007 を正本と明示。
- **B-5 = 受容**: CloudFront デフォルト証明書（`*.cloudfront.net`）は viewer TLS 最小バージョンが TLSv1 固定（AWS 仕様、引き上げ不可）。security.md §11「TLS 1.2 以上」と乖離するが $0 維持のため受容、ADR-0007 に記録。完全準拠には独自ドメイン + ACM（有料）が必要。

その他:
- security.md §11 / §441 本文は不変。受容逸脱（B-3 / B-5）は ADR-0007 に集約。
- issue #186 は T5 内部レビュー W-1 由来。architecture.md/diagrams.md/env_config.md のコンピュート層が ECS Fargate / 2 タスク表記だが実装実態は EC2 t3.micro 単一（ADR-0004 の EC2 ピボットが構成図に未反映）。T5 スコープ外として別 issue 化。

### PR / コミット要約

- **expense-saas**: PR #149（T1）/ #150（#184）/ #151（T3）スカッシュマージ済み。master `4812a01`。各 feature ブランチは保持。
- **dev-journal**: commit `1925081`（T5 設計成果物）/ `2155854`（finding 121 対応）。本セッション末で progress.md 等を追加コミット予定。
- **起票 issue**: #186（open, post-MVP）。
- **review-finding**: 121（resolved）。

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-20.md`（2026-05-20 の 2 セッション: 01:25 Step 11-E Phase 4 DB bootstrap 完走 / 16:27 Phase 5-6 完走・SPA 配信未実装の issue #183 を PR #148 で解決）

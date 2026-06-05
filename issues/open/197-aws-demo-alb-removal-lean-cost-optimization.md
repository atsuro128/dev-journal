# AWS公開デモの ALB 除去による lean 化（コスト最適化 + 手動 destroy/apply 運用）

## 発見日
2026-06-04

## カテゴリ
infra / cost

## 影響度
中（自腹は現状$0だが、クレジット枯渇＝2026年9月頃以降は月約$60の実費。放置不可の期限あり）

## 発見経緯
AWS予算アラート `monthly-5usd-alert`（$5上限・$4.25超過通知）の発報メール調査。Cost Explorer / Budgets / クレジット残を実データ確認した結果、下記コスト実態が判明。

## ブロッカー
なし（着手可能。実装は次セッションでホスト側 AWS 操作とあわせて実施予定）

## 背景（コスト実態 — 2026-06-04 実データ確定）

- AWSアカウント `<AWS_ACCOUNT_ID>`（ap-northeast-1）、**2026-05-16作成 = 新方式クレジット制 Free Tier**（旧来の「750h無料」が無く定価課金）。
- グロス稼働コスト **約$60/月**: RDS ~$20 / ALB ~$18 / 公開IPv4×3 ~$11 / EC2 ~$10 / ストレージ等。
- **クレジットが全額相殺中 → 現状の自腹は実質$0**。クレジット残 **$132.53**（4件・全て**2027-05-16失効**、消費は `AWS Free Tier $100` のみ $27.47）。~$60/月で **枯渇は2026年9月頃**（失効より枯渇が先）。
- 予算はグロス（クレジット適用前）を集計するため、デモ稼働中は毎月発報する（アラートメールは正規）。

## 方針（ユーザー合意済み — 2026-06-04 第2セッションで再決定）

ALB を除去し、CloudFront は **HTTPS 終端のために維持**（独自ドメイン不要・証明書無料）。CloudFront を EC2(EIP):8080 へ直結する。**オリジン保護は SG 一枚（CloudFront managed prefix list）に簡素化し、合言葉検証(X-Origin-Verify)は省く**。安全網はアプリ既存のレート制限（`RateLimitByIP` / `RateLimitByUser`）。運用は **深夜自動 stop/start（EventBridge Scheduler）** で日中常時公開・夜間節約、長期不在時用に手動 `destroy` を Makefile で用意する。

スケール対応設計（ALB+ASG）と「オリジン保護を多層防御に簡素化した判断」は ADR に記録して残す。

### 旧方針からの転換点（2026-06-04 第1セッション → 第2セッション）
> 第1セッションでは「Go ミドルウェアへ X-Origin-Verify 検証を移設して二重防御維持 + 手動 destroy/apply」で計画・reviewer 検証済みだった。第2セッションでユーザーと再議論し、以下に変更:
> - **Go ミドルウェア新設を取りやめ**（オリジン保護は SG + アプリのレート制限の多層で担保。総当たり/DoS はアプリのレート制限が安全網）。
> - **手動 destroy/apply → 深夜自動 stop/start を主運用に**（destroy は長期不在用に併設）。
> - 理由: ①Go 実装を避けたい ②独自ドメイン未保有のため CloudFront 維持が HTTPS 最安 ③ダミーデータ・標的型限定のため SG 一枚でも実害小 ④判断を ADR で言語化すれば多層防御の設計力アピールになる。

### 確定スペック
- **lean化**: ALB一式削除 → CloudFront を EC2(EIP):8080 直結 → EC2 SG を CloudFront managed prefix list 限定。
- **オリジン保護**: **SG 一枚**（CloudFront prefix list）。X-Origin-Verify 検証（ALB リスナールール）は廃止し、Go への移設も**行わない**。安全網はアプリのレート制限。
- **`trusted_proxy_count` 2→1**（CloudFront 1 段経由になるため。env のみ・Go 不介入）。
- **コスト運用**: EC2/RDS を EventBridge Scheduler で深夜 stop / 朝 start 自動化。`infra/terraform/Makefile` で apply/destroy/plan/seed をワンコマンド化（destroy は長期不在用）。
- **再現性**: デプロイ手順に seed 投入を組込み。添付サンプル2件は**本番S3へ投入**しデモ完全化。
- **設計記録**: ADR-0004/0007 を新構成に追記。

## タスク（実装フェーズ・依存順）

- [ ] **A. Terraform lean**: `alb.tf` 削除 / `eip.tf` 新設 / `cloudfront.tf` origin差替(EIP:8080) / `security_groups.tf`（ALB SG削除・EC2 SG prefix list化、**同時変更**） / `outputs.tf`（alb_dns_name削除・EIP出力へ） / `variables.tf`（X-Origin-Verify系・restrict_alb_to_cloudfront 整理） / `rds.tf`(skip_final_snapshot=true)。
- [ ] **B. 自動 stop/start**: EventBridge Scheduler で EC2/RDS の深夜 stop・朝 start。Terraform で定義（Lambda 不要・Go 不介入）。
- [ ] **C. user_data/env**: `TRUSTED_PROXY_COUNT=1` 反映（ORIGIN_VERIFY_SECRET は不要に）。
- [ ] **D. 運用**: `Makefile`（plan/apply/destroy/seed-remote）+ seed再現手順（owner DATABASE_URL は SSM、添付は S3_ENDPOINT=本番で投入）。
- [ ] **E. ADR/README/security.md**: ADR-0004/0007 追記（多層防御へ簡素化した判断を明記）、`infra/terraform/README.md` 改訂、`security.md` の XFF/オリジン保護記述更新。

> **取りやめ（旧計画フェーズ A）**: `internal/middleware/origin_verify.go` 新設・`config.go`/`main.go` 改修・`ORIGIN_VERIFY_SECRET`（SSM）は**実装しない**。Go コードは触らない。

### 実装上の必須注意（前回 reviewer 検証 + 新方針）
- (blocker) `outputs.tf` の `alb_dns_name` 削除（残すと plan エラー）。EIP 出力へ。
- (blocker) `cloudfront.tf` の `origin_id` / `target_origin_id` 整合 + `http_port=8080`。
- (blocker) ALB SG 削除と EC2 SG ingress 変更は不可分（順序誤ると EC2 完全閉塞）。
- (要検証) **SG 一枚に簡素化した場合のセキュリティ妥当性**（「他人の CloudFront 踏み台」リスクをアプリのレート制限で許容できるか）を reviewer で再評価する。
- (要検証) **EventBridge Scheduler の EC2/RDS stop/start 構成**（RDS の 7 日自動再起動・start 順序・CloudFront origin への影響）。
- (要検証) **EIP × CloudFront origin（public_dns）** の実現性（前回 reviewer で実現可能と確定済み・再確認）。
- (warning) `security.md` の XFF/オリジン保護記述更新（影響ドキュメント漏れ防止）。
- (warning) `trusted_proxy_count` 2→1 を origin 切替と同一 apply に。
- (warning) seed の owner `DATABASE_URL` 受け渡し（SSM）を手順明記。

## 受入基準
1. CloudFront経由 `/health` → 200 / prefix list 外からの `:8080` → SG 到達不可。
2. CloudFront経由でログイン〜レポートCRUD一連動作、**添付DLが200**（S3本体あり）。
3. `terraform plan` が ALB系0件・EIP1件・EventBridge Scheduler 系が定義どおり。
4. **destroy→apply→migrate→seed** 後、`test-admin-b@example.com`(TenantB Admin) 含む全 seed が再現。
5. 深夜 stop → 朝 start 後、CloudFront経由で正常応答（EIP 不変・CloudFront origin 再設定不要を確認）。
6. CloudWatchログのクライアントIPが正しい viewer IP（trusted_proxy_count=1）。
7. ローカル `docker compose up` が従来通り動作（Go 無改修のため影響なしを確認）。
8. `go vet ./... && go build ./...` 通過（Go 無改修の回帰確認）。

## プロダクト判断（確定済み）
- オリジン保護: **SG 一枚に簡素化**（合言葉検証は廃止、Go 移設もしない）。安全網はアプリのレート制限。判断を ADR に明記。
- 添付ファイル: **本番S3へ2ファイル投入**しデモ完全化。
- コスト運用: **深夜自動 stop/start**（主）+ 手動 destroy（長期不在用・併設）。
- 外形ヘルスチェック: 省略許容（ポートフォリオ用途）。
- ADR-0004/0007 更新: 実施。

## 関連
- 設計上流: ADR-0004（インフラ）/ ADR-0007（CloudFront）/ `50_detail_design/security.md`（XFF・レート制限 §4・オリジン保護）。
- レート制限実装: `expense-saas/internal/middleware/ratelimit.go`（`RateLimitByIP`=未認証IP単位 / `RateLimitByUser`=認証済ユーザー単位、トークンバケット・インメモリ）。SG 簡素化の安全網はこれ。
- 旧詳細計画（Go移設前提・一部失効）: ローカル非公開の作業資料に保存。新方針版は本 issue を正本とし、architect で計画を立て直す。
- 前提経緯: 前回 issue #109 で本番RDSへ Admin B を直接SQL投入。本 issue で seed をデプロイ手順に組込み、再現性を担保する。
- 補足: AWSアカウントID `<AWS_ACCOUNT_ID>` は公開 dev-journal の git 履歴に露出済みだが、アクセスキー等の機密漏れは無く非機密のため履歴書き換えは行わない判断（2026-06-04）。

## 状態
**計画確定（2026-06-04 第2セッション）・実装着手可**。architect 計画策定 → reviewer 検証（判定 FIX = 骨格健全・軽微修正）→ FIX 反映・ESCALATE 2件ユーザー判断済み（security.md 本文更新 / 移行ダウンタイム許容）。確定計画は作業資料 `private-materials/alb-lean-migration-plan-v2-2026-06-04.md`（reviewer 検証結果と確定事項を末尾に追記済み）。Go 実装は範囲外。次手順: ①expense-saas Terraform 実装（worktree・ブランチ `feat/197-terraform-lean`、ローカルゲートは fmt/validate、apply は AWS CLI 環境で別途）②dev-journal ADR/security.md 更新（ブランチ `docs/197-adr-security-update`）を並列。

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

## 方針（ユーザー合意済み）

ALB を除去して lean 構成（グロス**約$36/月**）にし、運用は**手動 `terraform destroy`/`apply`**（見せる時だけ apply、普段 destroy で $0）。自動 stop/start は作らない。スケール対応設計（ALB+ASG）は ADR に記録して残す（`enable_alb` フラグ化は見送り）。

### 確定スペック
- **lean化**: ALB一式削除 → CloudFront を EC2(EIP):8080 直結 → EC2 SG を CloudFront managed prefix list 限定。
- **オリジン保護**: X-Origin-Verify 検証を Go ミドルウェアへ移設（**本番フェイルクローズ / dev・CIフェイルオープン**、チェーン最外）。二重防御（SG + アプリ検証）維持。
- **`trusted_proxy_count` 2→1**（origin切替と同一applyで）。
- **運用**: `infra/terraform/Makefile` で apply/destroy/plan ワンコマンド化。
- **再現性**: デプロイ手順に seed 投入を組込み（現状 README は migrate のみ）。添付サンプル2件は**本番S3へ投入**しデモ完全化。
- **設計記録**: ADR-0004/0007 を lean に追記。

## タスク（実装フェーズ・依存順）

- [ ] **A. Goアプリ**: `internal/middleware/origin_verify.go`(+test) 新設、`config.go` / `main.go`（CORSより前に登録）。本番フェイルクローズ。
- [ ] **B. user_data/env**: `ORIGIN_VERIFY_SECRET`（SSM）・`TRUSTED_PROXY_COUNT` 反映。
- [ ] **C. Terraform lean**: `alb.tf` 削除 / `eip.tf` 新設 / `cloudfront.tf` origin差替(EIP:8080) / `security_groups.tf`（ALB SG削除・EC2 SG prefix list化、**同時変更**） / `outputs.tf`（alb_dns_name削除） / `variables.tf` / `rds.tf`(skip_final_snapshot=true)。
- [ ] **D. 運用**: `Makefile` + seed再現手順（owner DATABASE_URL は SSM、添付は S3_ENDPOINT=本番で投入）。
- [ ] **E. ADR/README**: ADR-0004/0007 追記、`infra/terraform/README.md` 改訂、`security.md:355` のXFF記述更新。

### reviewer 検証で確定した FIX（実装時必須）
- (blocker) `outputs.tf` の `alb_dns_name` 削除（残すと plan エラー）。
- (blocker) `cloudfront.tf` の `origin_id` / `target_origin_id`(:59,:72) 整合 + `http_port=8080`。
- (blocker) ALB SG 削除と EC2 SG ingress 変更は不可分（順序誤ると EC2 完全閉塞）。
- (warning) ミドルウェアは CORS より前・本番フェイルクローズ。
- (warning) `security.md:355` のXFF記述更新（影響ドキュメント漏れ）。
- (warning) `trusted_proxy_count` 2→1 を origin 切替と同一 apply に。
- (warning) seed の owner `DATABASE_URL` 受け渡し（SSM）を手順明記。

## 受入基準
1. CloudFront経由 `/health` → 200 / EIP直 `:8080`（ヘッダなし）→ 403 / prefix list外 `:8080` → SG到達不可。
2. CloudFront経由でログイン〜レポートCRUD一連動作、**添付DLが200**（S3本体あり）。
3. `terraform plan` が ALB系0件・EIP1件。
4. **destroy→apply→migrate→seed** 後、`test-admin-b@example.com`(TenantB Admin) 含む全 seed が再現。
5. CloudWatchログのクライアントIPが正しい viewer IP（trusted_proxy_count=1）。
6. ローカル `docker compose up` が env 未設定でも従来動作。
7. `go vet ./... && go build ./...` 通過。

## プロダクト判断（確定済み）
- 添付ファイル: **本番S3へ2ファイル投入**しデモ完全化。
- 検証ポリシー: **本番フェイルクローズ**。
- 外形ヘルスチェック: 省略許容（ポートフォリオ用途）。
- ADR-0004/0007 更新: 実施。

## 関連
- 設計上流: ADR-0004（インフラ）/ ADR-0007（CloudFront）/ `50_detail_design/security.md`（XFF）。
- 詳細計画・reviewer検証・apply順序: ローカル非公開の作業資料に保存（本 issue に要点を集約済み）。
- 前提経緯: 前回 issue #109 で本番RDSへ Admin B を直接SQL投入。本 issue で seed をデプロイ手順に組込み、再現性を担保する。
- 補足: AWSアカウントID `<AWS_ACCOUNT_ID>` は公開 dev-journal の git 履歴に露出済みだが、アクセスキー等の機密漏れは無く非機密のため履歴書き換えは行わない判断（2026-06-04）。

## 状態
起票のみ（計画・reviewer検証済み、実装未着手）。実装は次セッションでホスト側 AWS 操作とあわせて着手予定。

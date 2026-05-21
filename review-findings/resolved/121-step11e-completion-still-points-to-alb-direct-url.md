# 121: 11-E 完了判定が ALB 直アクセスを成功条件にしたまま

## 指摘概要

`progress-management/tickets/step11/11-E-deploy.md` の完了判定・11-F 引き継ぎが、issue #185 / ADR-0007 適用後も `ALB DNS` 直アクセスを公開 URL / `/health` 成功条件としている。

しかし ADR-0007 の B-1-b では ALB 直アクセスを遮断し、エンドユーザーは CloudFront ドメイン経由でのみ到達できる設計に変更されている。実装済み Terraform でも `cloudfront_domain_name` がエンドユーザー向け URL、`alb_dns_name` は直アクセス不可として出力説明が分かれている。

このままだと T6（CloudFront 経由の疎通・CORS・HSTS・レート制限検証）や 11-F UAT が、遮断されるべき ALB DNS を公開 URL / ヘルスチェック URL と誤認する。

## 根拠

- `dev-journal/progress-management/tickets/step11/11-E-deploy.md:1279`
  - `デプロイ済み環境で第三者がアクセスできる | §6.1 ALB DNS 経由で /health 200、§6.3 ログイン画面到達`
- `dev-journal/progress-management/tickets/step11/11-E-deploy.md:1280`
  - `ヘルスチェックエンドポイントが応答する | §6.1 curl http://<alb-dns>/health ...`
- `dev-journal/progress-management/tickets/step11/11-E-deploy.md:1282`
  - 11-F 引き継ぎの公開 URL が旧 TLS 案1の HTTP 運用差分を前提にしている。
- `dev-journal/deliverables/docs/30_arch/adr/0007-cloudfront-https.md:58-63`
  - B-1-b として CloudFront 経由を強制し、ヘッダ不一致・不在は ALB で 403 とする。
- `dev-journal/deliverables/docs/30_arch/adr/0007-cloudfront-https.md:98-99`
  - `CORS_ALLOWED_ORIGINS` は CloudFront ドメインになり、エンドユーザーは CloudFront ドメイン経由でのみ到達できる。
- `expense-saas/infra/terraform/outputs.tf:1-8`
  - `cloudfront_domain_name` はエンドユーザー向け URL、`alb_dns_name` は直アクセス不可として定義されている。

## 判定

重大度: 中

分類: 設計書整合 / 下流作業可能性

レビュー観点「上流資料との整合性」「成果物内部の整合性」「下流作業可能性」に反する。B-1-b のセキュリティ境界と、11-E / T6 / 11-F の検証入口が矛盾しているため、後続担当が追加解釈なしに正しい URL・期待結果を判断できない。

## 修正方針案

- `11-E-deploy.md` §10 の完了判定を CloudFront ドメイン基準に更新する。
  - 第三者アクセス: `https://<cloudfront_domain_name>/` 経由のログイン画面到達
  - ヘルスチェック: `https://<cloudfront_domain_name>/health` の応答確認
  - ALB DNS 直アクセス: `403` が期待結果（B-1-b の閉域確認）として別項目にする
- 11-F 引き継ぎの公開 URL / 既知差分から旧 TLS 案1（HTTP 運用）前提を外し、ADR-0007 の CloudFront 構成・B-3/B-5 受容逸脱へ差し替える。
- §11 Q2 付近の「案1 採用」履歴は残す場合でも、現行手順・完了判定が ADR-0007 を正とすることが読み取れるよう、履歴ブロックと現行ブロックを明確に分離する。

## 対応内容（2026-05-21）

`11-E-deploy.md` を修正:

- §1 Phase 7 / §6.1 / §6.3.2 / §6.3.3 / §9.3 / §10 完了判定 / 11-F 引き継ぎ: 公開 URL・ヘルスチェックを CloudFront ドメイン（`https://<cloudfront_domain_name>/`）基準に更新。
- §10 完了判定に「ALB DNS 直アクセスが 403 を返す（B-1-b 完全閉域の確認）」を独立完了項目として追加。
- §6.1 / §6.3.3 / §9.3 に「ALB DNS 直アクセス = 403 期待」の確認手順を明記。
- §11 Q2 を【現行構成】（ADR-0007 正本）と【判断履歴】（案1 採用当時の記録）の 2 ブロックに見出しレベルで分離。
- 残: §2/§3/§4 の planning セクションに案1（`http://<alb-dns>` CORS 等）当時の記述が残存。§11 Q2 履歴ブロック末尾に「現行整合の正本は ADR-0007 / env_config.md、CORS 値の最終反映は T4」と注記済み（正本は T5 で更新済みの env_config.md §4.5 のため別 issue は起票せず注記対応）。

## 再レビュー結果（2026-05-21）

PASS。`11-E-deploy.md` の §6/§10/§11 Q2/11-F 引き継ぎは CloudFront ドメイン基準に更新され、ALB DNS 直アクセスは 403 期待の閉域確認として扱われている。

- §6.1: `https://<cloudfront_domain_name>/health` を正規疎通確認、`http://<alb_dns_name>/health` を 403 期待として明記。
- §6.3.2 / §6.3.3: 公開 URL は `https://<cloudfront_domain_name>/`、ALB DNS 直アクセスは使わない/403 期待として明記。
- §10: 完了判定が CloudFront 経由のログイン画面到達・`/health` 200 を成功条件とし、ALB DNS 直アクセス 403 を独立完了項目に分離。
- §11 Q2 / 11-F 引き継ぎ: ADR-0007 の現行構成を正本とし、旧案1（HTTP / ALB DNS 直利用）は判断履歴として分離。既知差分も ADR-0007 B-3 / B-5 と Secrets Manager 不採用に更新済み。

下流 11-F / T6 が ALB DNS を公開 URL と誤認するリスクは解消したため、本指摘を resolved へ移動する。

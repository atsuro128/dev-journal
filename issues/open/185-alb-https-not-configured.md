# ALB が HTTP 平文で HTTPS 未対応（security.md §11 と乖離）

## 発見日
2026-05-20

## カテゴリ
infrastructure

## 影響度
中

## 発見経緯
proactive

## 関連ステップ
Step 11-E（デプロイ）/ Step 5（詳細設計・セキュリティ設計）

## ブロッカー
なし

## 問題

Step 11-E でデプロイした ALB は **HTTP:80 のみ**で、HTTPS:443 リスナー・ACM 証明書を持たない。エンドユーザーとの通信が全て平文。

一方、`security.md` §11「HTTPS / TLS」は以下を設計として定めている:

| 項目 | security.md §11 の設計 | 現状 |
|------|----------------------|------|
| TLS 終端 | ALB で終端 | **未対応（HTTP のみ）** |
| TLS バージョン | TLS 1.2 以上 | 該当なし |
| 証明書管理 | AWS Certificate Manager (ACM) | **証明書なし** |
| HTTP リダイレクト | ALB リスナールールで HTTP(80) → HTTPS(443) | **未設定** |
| HSTS | `Strict-Transport-Security: max-age=31536000; includeSubDomains` | アプリは HSTS ヘッダを返すが HTTPS 配信でないため矛盾 |

検出経緯: Step 11-E Phase 6 でブラウザから ALB DNS にアクセスした際、`http://` で接続されることを確認。ログイン画面のメール・パスワード送信も平文。

## 影響

- **認証情報が平文で流れる**: ログイン時のメールアドレス・パスワードが暗号化されない。中間者攻撃でクレデンシャルが窃取されうる。
- **HSTS ヘッダの矛盾**: アプリは `Strict-Transport-Security` ヘッダを返しているが、HTTPS で配信されていないため、このヘッダは無意味（HTTP レスポンスの HSTS はブラウザに無視される）。設計（security.md §460, §710）と実態が不整合。
- **security.md §441「本番環境では HTTP オリジンの許可は禁止（HTTPS のみ）」に違反**。
- portfolio のデモとして第三者にログインしてもらう場合、認証情報の平文送信は説明上も望ましくない。

## 背景（既知の経緯）

11-E チケット §11 Q2「TLS / カスタムドメイン」で案1（HTTP のみ、ALB DNS 直利用）/ 案2（Route53 + ACM + HTTPS）が併記され、**案1（HTTP のみ）が採用**されている。Terraform（`alb.tf` / `security_groups.tf`）も「§11 Q2 案1 採用のため HTTP のみ、443/ACM は作らない」とコメント付きで案1 に忠実に実装されている。

このため、本件は「チケットが設計と矛盾していた」のではなく「設計逸脱を案1/案2 の選択肢として用意し、案1 が選ばれた」結果である。ただし以下の不備がある:

1. **チケット内部のロジック不整合**: §11 Q2 の案1 採用理由は「採用面接の評価者等に見せないなら案1。見せるなら案2」。本プロジェクトは「ポートフォリオ兼デモ」で評価者に見せる前提のため、チケット自身の基準では案2 を採るべきだった。
2. **参照切れ**: チケット §2.1 / §2.2 は「§3.5 で判断」と参照するが §3.5 は存在しない。実際の判断は §11 Q2 にある（ドキュメントバグ）。
3. **ADR 参照ミス**: チケット §11 Q2（line 1317）は「ADR-0004 §環境構成 prod 行は TLS: ACM」と記すが、ADR-0004 に TLS の記述は実在しない。HTTPS を定めているのは security.md §11 と env_config.md §3.1 prod。

本 issue は案1 採用判断を覆すものではなく、security.md §11 との乖離を明示記録し、HTTPS 化（CloudFront 案）の対応方針を次セッションで詰めるためのもの。

## 提案

対応方針の選択肢（post-MVP で判断）:

- **A 案: 独自ドメイン取得 + ACM 証明書**
  - Route53 Hosted Zone（$0.50/月）でドメイン管理 → ACM で証明書発行（DNS 検証）→ ALB に HTTPS:443 リスナー + HTTP→HTTPS リダイレクト追加
  - security.md §11 に完全準拠。コストは Hosted Zone 分のみ
- **B 案: HTTP のまま運用（portfolio リスク受容）**
  - 独自ドメイン・証明書を持たず HTTP 継続。security.md §11 からの逸脱を ADR で正式に記録
  - その場合、アプリの HSTS ヘッダは外す（HTTP で無意味かつ矛盾のため）か、逸脱理由を明記
- **C 案: CloudFront 経由（CloudFront のデフォルト証明書で HTTPS 化）**
  - CloudFront ディストリビューションを ALB の前段に置けば、独自ドメインなしで `*.cloudfront.net` の HTTPS が使える
  - インフラ構成が 1 段増える

### 採用方針（2026-05-20 ユーザー判断）

**C 案（CloudFront）を採用**。理由: 独自ドメイン取得・ドメイン代が不要で実質 $0（CloudFront 永久無料枠内）、既存 ALB 構成を壊さず前段に追加できる。

次セッションで architect に以下を含む実装計画を立てさせる:
- CloudFront ディストリビューションの Terraform 構成（オリジン = ALB、`/api/*` 非キャッシュのビヘイビア分け、`redirect-to-https`、全 HTTP メソッド許可）
- **アプリのクライアント IP 取得ロジック（`X-Forwarded-For` 解釈）の調査** — CloudFront 経由でプロキシ段が増えるとレート制限が壊れるリスクがあるため必須
- CloudFront バイパス（ALB 直 HTTP アクセス）を塞ぐか否かの方針（SG を CloudFront IP レンジ限定 / カスタムヘッダ検証 / 妥協）
- TLS 終端が ALB→CloudFront に変わる差分の ADR 起票
- issue #184（SPA ハンドラ HEAD 405）を CloudFront 化の前に解消するか判断
- `CORS_ALLOWED_ORIGINS` の CloudFront ドメインへの変更、`architecture.md` 構成図・11-E チケット §11 Q2 の更新

影響度は中（認証情報平文 + 設計乖離）だが、ポートフォリオ MVP の段階では即時ブロッカーではないため次セッションで対応。対応方針決定時は security.md §11・11-E チケット §11 Q2 と整合させること。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

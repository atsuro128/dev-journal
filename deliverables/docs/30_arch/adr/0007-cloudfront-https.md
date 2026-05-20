# ADR-0007: ALB 前段 CloudFront による HTTPS 化

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | エンドユーザー〜サーバー間の TLS 終端を ALB から CloudFront へ変更する判断根拠を記録する |
| 正本情報 | TLS 終端方式の検討（独自ドメイン+ACM / HTTP 継続 / CloudFront 経由）、決定、受容する逸脱、理由、トレードオフ |
| 扱わない内容 | インフラ選定全体（ADR-0004 が正本）、`X-Forwarded-For` 解釈の実装詳細（`security.md` / 11-E チケットに委譲） |
| 主な参照元 | `../../../issues/`（issue #185）, `./0004-infra.md`, `../../50_detail_design/security.md` §11, §441 |
| 主な参照先 | `../architecture.md` §2, §6.1, `../diagrams.md` §1, §2, `../../70_operations/env_config.md` §3.1 |

## 背景

- Step 11-E でデプロイした ALB は **HTTP:80 のみ**で HTTPS:443 リスナー・ACM 証明書を持たず、エンドユーザーとの通信が全て平文だった（issue #185）。ログイン時のメールアドレス・パスワードが暗号化されず、中間者攻撃でクレデンシャルが窃取されうる。
- 一方 `security.md` §11「HTTPS / TLS」は TLS 終端・TLS 1.2 以上・ACM 証明書・HTTP→HTTPS リダイレクト・HSTS を設計として定めており、デプロイ実態と乖離していた。アプリは HSTS ヘッダ（`security.md` §460, §710）を返すが HTTPS 配信でないため無意味になっていた。
- 11-E チケット §11 Q2「TLS / カスタムドメイン」では案1（HTTP のみ、ALB DNS 直利用）/ 案2（Route53 + ACM + HTTPS）が併記され、案1 が採用されていた。本件はその設計逸脱を是正するものである。
- 本プロジェクトはポートフォリオ兼デモ用途であり、運用コストを実質 $0 に抑える方針（AWS 無料枠内）を採っている。独自ドメインの取得・維持（Route53 Hosted Zone $0.50/月 + ドメイン年契約）はこの方針と相容れないため採用しない。

## 検討した選択肢

### 案A: 独自ドメイン取得 + ACM 証明書

- Route53 Hosted Zone でドメインを管理し、ACM で証明書を DNS 検証で発行、ALB に HTTPS:443 リスナー + HTTP→HTTPS リダイレクトを追加する。
- **利点**: `security.md` §11 に完全準拠。viewer 側 TLS 1.2+ を強制でき、CloudFront〜オリジン間も HTTPS にできる。
- **不採用理由**: Route53 Hosted Zone（$0.50/月）+ ドメイン年契約費が恒常的に発生し、コスト $0 方針に反する。ポートフォリオ MVP の段階では費用対効果が見合わない。

### 案B: HTTP のまま運用（リスク受容）

- 独自ドメイン・証明書を持たず HTTP を継続し、`security.md` §11 からの逸脱を ADR で記録する。
- **利点**: 追加構成が一切不要。
- **不採用理由**: 認証情報の平文送信という主要リスクが解消されない。ポートフォリオとして第三者にログインしてもらうデモ用途で説明上も望ましくない。issue #185 の本来の目的（HTTPS 化）を達成しない。

### 案C: ALB 前段に CloudFront を置く（CloudFront デフォルト証明書で HTTPS 化）

- CloudFront ディストリビューションを ALB の前段に配置する。`*.cloudfront.net` のデフォルト証明書により、独自ドメイン・ACM なしでエンドユーザー〜CloudFront 間を HTTPS 化できる。
- **利点**: 独自ドメイン取得・維持費が不要で実質 $0（CloudFront 永久無料枠内）。既存 ALB 構成を壊さず前段に追加でき、アプリケーションコードへの影響が小さい。
- **欠点**: インフラ構成が 1 段増える。CloudFront〜ALB 間が HTTP のまま残る（後述 B-3 の受容逸脱）。デフォルト証明書では viewer 側 TLS 最小バージョンを引き上げられない（後述 B-5 の受容逸脱）。プロキシ段が増えるためアプリのクライアント IP 取得ロジック（`X-Forwarded-For` 解釈）の見直しが必要。

## 決定

**案C を採用する。** TLS 終端を ALB から CloudFront へ変更し、`*.cloudfront.net` デフォルト証明書でエンドユーザー〜CloudFront 間を HTTPS 化する。リクエスト経路は `Browser → CloudFront → ALB → EC2（Go API + SPA）` となる。

### CloudFront ディストリビューション構成

| 項目 | 構成 |
|------|------|
| オリジン | 既存 ALB（HTTP:80） |
| viewer プロトコルポリシー | `redirect-to-https`（HTTP アクセスを HTTPS へリダイレクト） |
| 証明書 | CloudFront デフォルト証明書（`*.cloudfront.net`、独自ドメイン・ACM 不要） |
| デフォルトビヘイビア | SPA 静的配信。`GET`/`HEAD`、`Managed-CachingOptimized` でエッジキャッシュ |
| `/api/*` ビヘイビア | API リクエスト。全 HTTP メソッド許可、`Managed-CachingDisabled` で**非キャッシュ**、`Managed-AllViewerExceptHostHeader` で `Authorization`・Cookie・クエリをオリジンへ転送 |
| `/health` の扱い | `/api/*` パターンに合致せずデフォルトビヘイビアに流れるが、機密情報を含まないためエッジにキャッシュされても実害なし（W-A） |
| 価格クラス | `PriceClass_200`（北米・欧州・アジアパシフィック、日本含む。無料枠維持） |

### B-1-b: ALB の完全閉域化（CloudFront 経由の強制）

CloudFront を前段に置いても ALB の DNS に直接 HTTP アクセスできてしまうと閉域化が不完全になる。これを 2 層防御で塞ぐ（B-1-b、W-B）。

1. **1 層目（常時有効）— カスタムヘッダ検証**: CloudFront がオリジンリクエストにカスタムヘッダ `X-Origin-Verify`（秘密値）を付与し、ALB リスナールールでこの値を検証する。一致時のみターゲットグループへ転送し、不一致・ヘッダ不在は ALB リスナーのデフォルトアクションで **403** を返す。他者が自前の CloudFront を ALB に向けても、この秘密値を知らなければ到達できない。
2. **2 層目（可変）— SG プレフィックスリスト限定**: ALB セキュリティグループの inbound 80 を CloudFront マネージドプレフィックスリスト `com.amazonaws.global.cloudfront.origin-facing` 限定に絞る。

2 層目は CloudFront ディストリビューションが `Deployed` になってから有効化する必要があるため、Terraform 変数 `restrict_alb_to_cloudfront` で段階適用する（初回 apply は `false`、CloudFront Deployed 後に `true` で再 apply）。1 層目は常時有効なため、`false` の間（SG が `0.0.0.0/0`）もカスタムヘッダ検証で保護されセキュリティギャップは生じない。

### 受容する逸脱

本決定に伴い、`security.md` の以下 2 点との乖離を**受容する逸脱**として記録する。`security.md` 本文（§11 / §441）は改変せず、逸脱の正本は本 ADR とする。

#### B-3: CloudFront〜ALB 間が HTTP（`security.md` §441 との乖離）

- **逸脱内容**: CloudFront〜ALB 間（オリジン通信）は HTTP のまま残る。`security.md` §441「本番環境では HTTP オリジンの許可は禁止（HTTPS のみ）」と乖離する。
- **受容理由**: CloudFront〜ALB 間を HTTPS にするには ALB に独自ドメイン+ACM 証明書が必要となり、コスト $0 方針に反する（案A 相当のコストが発生する）。一方、本件の主目的である「エンドユーザー〜サーバー間の平文通信の解消」は、エンドユーザー〜CloudFront 間が HTTPS 化されることで達成される。オリジン通信は AWS のネットワーク内（CloudFront エッジ〜ap-northeast-1 の ALB）に閉じており、かつ B-1-b でカスタムヘッダ検証 + SG プレフィックスリストにより CloudFront 経由以外の到達を遮断しているため、ポートフォリオ用途ではこの逸脱を受容する。
- **残存リスク（W-B）**: オリジン通信が傍受された場合、`X-Origin-Verify` 秘密値とリクエスト内容が露出しうる。ただし AWS バックボーン内通信であり、ポートフォリオ MVP の脅威モデルでは許容範囲とする。独自ドメイン+ACM へ移行する際にオリジンも HTTPS 化することで解消できる。

#### B-5: viewer 側 TLS 最小バージョンが TLSv1 固定（`security.md` §11 との乖離）

- **逸脱内容**: CloudFront デフォルト証明書（`*.cloudfront.net`）を使う場合、viewer 側 TLS の最小バージョンは `TLSv1`（1.0）に固定され、`minimum_protocol_version` で引き上げられない（AWS 仕様）。`security.md` §11「TLS バージョン: TLS 1.2 以上」と乖離する。
- **受容理由**: viewer 側 TLS 1.2+ を実際に強制するには独自ドメイン+ACM 証明書が必要でコストが発生する（案A 相当）。現代の主要ブラウザは全て TLS 1.2/1.3 で接続するため、TLSv1 が最小として許可されていても実際に TLS 1.0/1.1 でハンドシェイクするクライアントはほぼ存在せず、実害は事実上ゼロである。コスト $0 維持を優先しこの逸脱を受容する。
- **残存リスク**: TLS 1.0/1.1 のみ対応の旧クライアントが接続した場合、弱い TLS で通信しうる。ポートフォリオのデモ用途では当該クライアントは想定外であり、許容範囲とする。

## 理由

1. **コスト $0 方針との両立**: 案C は独自ドメイン取得・維持費が不要で、CloudFront 永久無料枠（毎月のデータ転送量・リクエスト数が無料枠内）に収まる。案A は恒常的なコストが発生し、案B は主目的を達成しない。
2. **主目的の達成**: エンドユーザー〜CloudFront 間が HTTPS 化されることで、認証情報の平文送信という issue #185 の主要リスクが解消される。HSTS ヘッダも HTTPS 配信下で意味を持つようになる。
3. **既存構成への影響が小さい**: 案C は ALB の前段に CloudFront を追加するだけで、ALB・EC2・RDS・S3 の構成は変更しない。アプリケーションコードへの影響はプロキシ段増加に伴う `X-Forwarded-For` 解釈の修正（`TRUSTED_PROXY_COUNT` 方式、issue #185 T1）に限られる。
4. **逸脱の限定と記録**: 残存する逸脱（B-3 / B-5）はいずれも「独自ドメイン+ACM があれば解消できる」性質のもので、移行パスが明確である。本 ADR に受容理由・残存リスク・解消条件を記録することで、将来の判断材料を残す。

## ADR-0004 との関係

- ADR-0004「インフラ選定」は本システムのインフラ構成（コンピュート・DB・ストレージ・IaC・ロードバランサ）の選定根拠を定める**正本**である。
- ADR-0004 §SPA 配信方式は「S3 + CloudFront」を**不採用**としているが、これは SPA 静的ファイルの配信方式（Go embed で同一コンテナ配信を採用）に関する判断であり、本 ADR の「TLS 終端のための CloudFront 前段配置」とは目的・対象が異なる。両者は矛盾しない（SPA は引き続き Go embed で配信し、CloudFront はその前段で TLS 終端とエッジ配信を担う）。
- 本 ADR は ADR-0004 に対する**差分追補**である。ADR-0004 本文は改変しない。インフラ構成の正本性は ADR-0004 に残し、CloudFront 前段化の判断は本 ADR を参照すること。

## 影響・結果

- **TLS 終端の移動**: TLS 終端が ALB から CloudFront へ移る。`security.md` §11 が定める ALB での TLS 終端・ACM 証明書・ALB での HTTP→HTTPS リダイレクトは、本 ADR では CloudFront 側（`redirect-to-https` + デフォルト証明書）で代替される。
- **クライアント IP 取得**: CloudFront 追加によりプロキシ段が 1 段増える（CloudFront + ALB の 2 段）。アプリのレート制限はクライアント IP をキーにするため、`X-Forwarded-For` の解釈を `TRUSTED_PROXY_COUNT` 方式へ修正する（prod=2、dev=0。issue #185 T1 / B-2-c）。修正しないとレート制限が誤動作する。
- **CORS**: prod の `CORS_ALLOWED_ORIGINS` は ALB DNS ではなく CloudFront ドメイン（`https://<cloudfront_domain_name>`）になる。`env_config.md` §3.1 / §4.5 に反映する。
- **ALB 直アクセスの遮断**: B-1-b により ALB DNS への直接 HTTP アクセスは 403 となり、エンドユーザーは CloudFront ドメイン経由でのみ到達できる。
- **コスト**: CloudFront は永久無料枠内のため実質 $0。ALB・EC2・RDS・S3 の費用は ADR-0004 のポートフォリオ対応のまま変わらない。
- **将来の独自ドメイン移行**: 独自ドメイン+ACM を導入する場合、CloudFront の代替ドメイン名（CNAME）と ACM 証明書（us-east-1）を設定し viewer 側 TLS 1.2+ を強制（B-5 解消）、ALB にも独自ドメイン+ACM を導入してオリジン通信を HTTPS 化（B-3 解消）できる。

## 反映先

| 反映先文書 | 反映内容 |
|-----------|---------|
| `../architecture.md` §2 | システム全体構成: リクエスト経路に CloudFront を ALB 前段として追記（`Browser → CloudFront → ALB → Task`） |
| `../architecture.md` §6.1 | 多層防御 [1] ネットワーク層: TLS 終端を CloudFront に変更、B-1-b による ALB 閉域化を追記 |
| `../diagrams.md` §1 | システム構成図: CloudFront を ALB 前段に追記 |
| `../diagrams.md` §2 | リクエスト処理フロー: CloudFront を経路に追記 |
| `../../70_operations/env_config.md` §3.1 | インフラ構成差分に CDN（CloudFront）行を追加、prod の TLS / CORS を CloudFront 構成に反映 |
| `../../70_operations/env_config.md` §4 | 環境変数 `TRUSTED_PROXY_COUNT`（dev=0 / prod=2）を追加 |
| `../../../progress-management/tickets/step11/11-E-deploy.md` §11 Q2 | 「案1（HTTP のみ）採用」記述に本 ADR への追補ノートを追加（履歴は残す） |
| `../../50_detail_design/security.md` §11, §441 | 本文は改変しない。受容する逸脱（B-3 / B-5）の正本は本 ADR |

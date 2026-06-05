# ADR-0007: CloudFront 直結（ALB 除去）による HTTPS 化とオリジン保護の簡素化

> **改訂履歴**: 当初は「ALB 前段 CloudFront による HTTPS 化」（issue #185）として ALB を維持し前段に CloudFront を置く構成だったが、issue #197（lean 化・コスト最適化）により **ALB を除去し CloudFront を EC2(EIP):8080 へ直結**する構成へ改訂した。あわせてオリジン保護を「カスタムヘッダ検証 + SG の 2 層」から「SG 1 層（CloudFront prefix list 限定）」へ簡素化した。本 ADR は改訂後の構成を正本とする。

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | エンドユーザー〜サーバー間の TLS 終端を CloudFront で行う構成（ALB を除去し CloudFront を EC2 直結する lean 構成）と、オリジン保護を SG 1 層へ簡素化する判断根拠を記録する |
| 正本情報 | TLS 終端方式の検討（独自ドメイン+ACM / HTTP 継続 / CloudFront 経由）、ALB 除去とオリジン保護簡素化の決定、受容する逸脱・残存リスク、理由、トレードオフ |
| 扱わない内容 | インフラ選定全体（ADR-0004 が正本）、`X-Forwarded-For` 解釈の実装詳細（`security.md` に委譲） |
| 主な参照元 | `../../../issues/`（issue #185, #197）, `./0004-infra.md`, `../../50_detail_design/security.md` §4, §11 |
| 主な参照先 | `../architecture.md` §2, §6.1, `../diagrams.md` §1, §2, `../../70_operations/env_config.md` §3.1, `../../50_detail_design/security.md` §4.1, §11 |

## 背景

- Step 11-E でデプロイした ALB は **HTTP:80 のみ**で HTTPS:443 リスナー・ACM 証明書を持たず、エンドユーザーとの通信が全て平文だった（issue #185）。ログイン時のメールアドレス・パスワードが暗号化されず、中間者攻撃でクレデンシャルが窃取されうる。
- 一方 `security.md` §11「HTTPS / TLS」は TLS 終端・TLS 1.2 以上・ACM 証明書・HTTP→HTTPS リダイレクト・HSTS を設計として定めており、デプロイ実態と乖離していた。アプリは HSTS ヘッダ（`security.md` §460, §710）を返すが HTTPS 配信でないため無意味になっていた。
- 11-E チケット §11 Q2「TLS / カスタムドメイン」では案1（HTTP のみ、ALB DNS 直利用）/ 案2（Route53 + ACM + HTTPS）が併記され、案1 が採用されていた。本件はその設計逸脱を是正するものである。
- 本プロジェクトはポートフォリオ兼デモ用途であり、運用コストを実質 $0 に抑える方針（AWS 無料枠内）を採っている。独自ドメインの取得・維持（Route53 Hosted Zone $0.50/月 + ドメイン年契約）はこの方針と相容れないため採用しない。
- **lean 化の背景（issue #197）**: AWS 新方式 Free Tier（旧来の「750h 無料」なし）への移行に伴い、ALB（~$18-20/月）が最大のコスト要因となった。CloudFront は HTTPS 終端のために維持しつつ ALB を除去し、CloudFront を EC2(EIP):8080 へ直結することでコストを削減する。ALB が担っていたオリジン保護（カスタムヘッダ検証）を廃し、SG 1 層（CloudFront prefix list 限定）＋アプリのレート制限へ簡素化する。判断の詳細は「決定」「なぜ多層から SG 一枚へ簡素化したか」を参照（コスト試算は ADR-0004）。

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

**案C を採用する。** TLS 終端を CloudFront で行い、`*.cloudfront.net` デフォルト証明書でエンドユーザー〜CloudFront 間を HTTPS 化する。**当初は ALB を維持し前段に CloudFront を置く構成（`Browser → CloudFront → ALB → EC2`）だったが、issue #197（lean 化）により ALB を除去し、CloudFront を EC2(EIP):8080 へ直結する。** 改訂後のリクエスト経路は `Browser → CloudFront → EC2(EIP):8080（Go API + SPA）` となる。

### CloudFront ディストリビューション構成

| 項目 | 構成 |
|------|------|
| オリジン | EC2 の EIP（HTTP:8080）。当初は ALB（HTTP:80）だったが issue #197 で ALB 除去・EC2 直結に改訂 |
| viewer プロトコルポリシー | `redirect-to-https`（HTTP アクセスを HTTPS へリダイレクト） |
| 証明書 | CloudFront デフォルト証明書（`*.cloudfront.net`、独自ドメイン・ACM 不要） |
| デフォルトビヘイビア | SPA 静的配信。`GET`/`HEAD`、`Managed-CachingOptimized` でエッジキャッシュ |
| `/api/*` ビヘイビア | API リクエスト。全 HTTP メソッド許可、`Managed-CachingDisabled` で**非キャッシュ**、`Managed-AllViewerExceptHostHeader` で `Authorization`・Cookie・クエリをオリジンへ転送 |
| `/health` の扱い | `/api/*` パターンに合致せずデフォルトビヘイビアに流れるが、機密情報を含まないためエッジにキャッシュされても実害なし（W-A） |
| 価格クラス | `PriceClass_200`（北米・欧州・アジアパシフィック、日本含む。無料枠維持） |

### B-1-b: オリジン（EC2:8080）の閉域化（CloudFront 経由の強制）

CloudFront を経由しても EC2 の EIP:8080 に直接 HTTP アクセスできてしまうと閉域化が不完全になる。**issue #197（lean 化）により、当初の「カスタムヘッダ検証 + SG の 2 層防御」を「SG 1 層」へ簡素化する。**

**SG 1 層（CloudFront prefix list 限定）**: EC2 セキュリティグループの inbound 8080 を CloudFront マネージドプレフィックスリスト `com.amazonaws.global.cloudfront.origin-facing` 限定に絞る。これにより CloudFront エッジ以外の IP からの :8080 到達を SG で遮断する。ALB を除去したため、ALB リスナールールによるカスタムヘッダ（`X-Origin-Verify`）検証は廃止する。

ALB が存在しない新構成では SG を初回 apply から prefix list 限定で適用できるため、旧構成の `restrict_alb_to_cloudfront` による段階適用（初回 `false` で `0.0.0.0/0` 開放 → Deployed 後 `true`）は不要となり、無防備な開放窓は生じない。Terraform 変数 `restrict_alb_to_cloudfront` は削除する。

### なぜ多層から SG 一枚へ簡素化したか（issue #197）

当初（issue #185）はオリジン保護を 2 層（カスタムヘッダ検証 + SG）で設計していたが、lean 化に伴い ALB を除去した結果、カスタムヘッダ検証を担っていた ALB リスナールールが消失する。これを Go アプリ側へ移設する案も検討したが、Go コードへの影響を避けるため**移設せず廃止**し、SG 1 層へ簡素化する判断をした。本節にその判断を記録する。

#### 廃止する保護

- **X-Origin-Verify カスタムヘッダ検証**（旧 ALB リスナールール）: CloudFront がオリジンリクエストに付与する秘密値を ALB リスナールールで検証し、一致時のみ転送していた。ALB 除去に伴い廃止する。Go ミドルウェアへの移設も行わない（Go コードは触らない方針）。Terraform 変数 `cloudfront_origin_verify_secret` も削除する。

#### 残す保護

- **SG（CloudFront prefix list 限定）**: EC2 SG inbound 8080 を CloudFront マネージドプレフィックスリスト限定にし、CloudFront エッジ以外からの :8080 到達を遮断する。
- **アプリのレート制限**（`internal/middleware/ratelimit.go`、`RateLimitByIP` / `RateLimitByUser`）: トークンバケット・インメモリ方式。実値は **未認証 20 req/min・IP / ログイン 5 req/min・IP（`/api/auth/login` 個別）/ 認証済 100 req/min・user**。総当たり（ブルートフォース）・DoS に対する安全網として機能する。

#### 残存リスク（受容）

- **想定リスク**: SG は CloudFront prefix list 全体を許可するため、「他者が自前の CloudFront ディストリビューションを踏み台にして prefix list 経由で EC2:8080 へ到達」しうる。カスタムヘッダ検証があれば秘密値を知らない限り遮断できたが、廃止により当該経路は理論上開く。
- **受容理由**:
  1. **(a) ダミーデータのみ**: 本番デモには実顧客データを置かず、seed 投入したダミーデータのみを扱う。漏洩しても実害が限定的。
  2. **(b) 標的型に限定**: 当該攻撃は「この EC2 のオリジン構成を知り、自前 CloudFront を立てて踏み台にする」標的型に限られ、無差別スキャンでは成立しにくい。
  3. **(c) アプリのレート制限が安全網**: ブルートフォース・DoS はアプリのレート制限（未認証 20/min・ログイン 5/min）で抑止される。
  4. **(d) ポートフォリオ用途**: 本プロジェクトはポートフォリオ兼デモであり、上記前提下では受容可能なリスクと判断する。
- **将来の再導入余地**: 本番でスケールし実データを扱う段階では、**X-Origin-Verify カスタムヘッダ検証の再導入（リバースプロキシ or Go ミドルウェア）または mTLS**でオリジン保護を多層化する設計余地を残す。その際は ALB+ASG の再導入（ADR-0004）とあわせて検討する。

### 受容する逸脱

本決定に伴い、`security.md` の以下 2 点（B-3 / B-5）との乖離を**受容する逸脱**として記録する。これらはいずれも「独自ドメイン+ACM があれば解消できる」TLS 上の逸脱であり、**この 2 点に限っては `security.md` 本文（§11）を改変せず、逸脱の正本を本 ADR とする**据え置き原則を適用する。

> **据え置き原則の適用範囲（issue #197 で整理）**: 上記「本文据え置き・逸脱の正本は本 ADR」とする原則は **B-3 / B-5 の TLS 逸脱に限る**。これに対し、issue #197 のような**構成変更（ALB 除去・TLS 終端の CloudFront 移行・`trusted_proxy_count` の変更・オリジン保護の SG 1 層への簡素化）は、据え置きではなく `security.md` 本文に反映する**。すなわち §4.1（IP 取得＝XFF 右から `trusted_proxy_count` 段目）・§11（TLS 終端＝CloudFront・HTTP リダイレクト＝CloudFront・EC2 との通信＝HTTP）・オリジン保護（SG 1 層）は本文を更新済みであり、その判断の正本は本 ADR を参照する（詳細は「影響・結果」）。

#### B-3: CloudFront〜EC2 間が HTTP（`security.md` §11「EC2 との通信」との関係）

- **逸脱内容**: CloudFront〜EC2 間（オリジン通信、EIP:8080）は HTTP のまま残る。「本番環境では HTTP オリジンの許可は禁止（HTTPS のみ）」という一般原則と乖離する。当初は CloudFront〜ALB 間 HTTP だったが、ALB 除去（issue #197）に伴い CloudFront〜EC2(EIP:8080) 間 HTTP に置き換わる（乖離の性質は同質）。
- **受容理由**: CloudFront〜EC2 間を HTTPS にするには EC2 に独自ドメイン+ACM 証明書（または自己署名証明書の運用）が必要となり、コスト $0・運用最小化方針に反する。一方、本件の主目的である「エンドユーザー〜サーバー間の平文通信の解消」は、エンドユーザー〜CloudFront 間が HTTPS 化されることで達成される。オリジン通信は AWS のネットワーク内（CloudFront エッジ〜ap-northeast-1 の EC2）に閉じており、かつ B-1-b で SG（CloudFront prefix list 限定）により CloudFront 経由以外の到達を遮断しているため、ポートフォリオ用途ではこの逸脱を受容する。
- **残存リスク**: オリジン通信が傍受された場合、リクエスト内容が露出しうる。ただし AWS バックボーン内通信であり、ポートフォリオ MVP の脅威モデルでは許容範囲とする。独自ドメイン+ACM へ移行する際にオリジンも HTTPS 化することで解消できる。

#### B-5: viewer 側 TLS 最小バージョンが TLSv1 固定（`security.md` §11 との乖離）

- **逸脱内容**: CloudFront デフォルト証明書（`*.cloudfront.net`）を使う場合、viewer 側 TLS の最小バージョンは `TLSv1`（1.0）に固定され、`minimum_protocol_version` で引き上げられない（AWS 仕様）。`security.md` §11「TLS バージョン: TLS 1.2 以上」と乖離する。
- **受容理由**: viewer 側 TLS 1.2+ を実際に強制するには独自ドメイン+ACM 証明書が必要でコストが発生する（案A 相当）。現代の主要ブラウザは全て TLS 1.2/1.3 で接続するため、TLSv1 が最小として許可されていても実際に TLS 1.0/1.1 でハンドシェイクするクライアントはほぼ存在せず、実害は事実上ゼロである。コスト $0 維持を優先しこの逸脱を受容する。
- **残存リスク**: TLS 1.0/1.1 のみ対応の旧クライアントが接続した場合、弱い TLS で通信しうる。ポートフォリオのデモ用途では当該クライアントは想定外であり、許容範囲とする。

## 理由

1. **コスト $0 方針との両立**: 案C は独自ドメイン取得・維持費が不要で、CloudFront 永久無料枠（毎月のデータ転送量・リクエスト数が無料枠内）に収まる。案A は恒常的なコストが発生し、案B は主目的を達成しない。
2. **主目的の達成**: エンドユーザー〜CloudFront 間が HTTPS 化されることで、認証情報の平文送信という issue #185 の主要リスクが解消される。HSTS ヘッダも HTTPS 配信下で意味を持つようになる。
3. **アプリケーションコードへの影響が小さい**: TLS 終端は CloudFront が担い、Go アプリ・RDS・S3 のロジックは変更しない。アプリへの影響はプロキシ段に応じた `X-Forwarded-For` 解釈（`TRUSTED_PROXY_COUNT` 方式、issue #185 T1）に限られ、これも env 値の設定のみで対応する（lean 化で ALB を除去した結果、プロキシ段は CloudFront 1 段となり prod の `TRUSTED_PROXY_COUNT` は 2→1 に変更する。issue #197）。
4. **逸脱の限定と記録**: 残存する TLS 逸脱（B-3 / B-5）はいずれも「独自ドメイン+ACM があれば解消できる」性質のもので、移行パスが明確である。オリジン保護の SG 1 層への簡素化に伴う残存リスク（他者 CloudFront 踏み台）も受容理由・将来の再導入余地とあわせて記録した。本 ADR に判断・受容理由・残存リスク・解消条件を記録することで、将来の判断材料を残す。

## ADR-0004 との関係

- ADR-0004「インフラ選定」は本システムのインフラ構成（コンピュート・DB・ストレージ・IaC・ロードバランサ）の選定根拠を定める**正本**である。
- ADR-0004 §SPA 配信方式は「S3 + CloudFront」を**不採用**としているが、これは SPA 静的ファイルの配信方式（Go embed で同一コンテナ配信を採用）に関する判断であり、本 ADR の「TLS 終端のための CloudFront 配置」とは目的・対象が異なる。両者は矛盾しない（SPA は引き続き Go embed で配信し、CloudFront はその（lean 構成では EC2 直結の）前段で TLS 終端とエッジ配信を担う）。
- 本 ADR は ADR-0004 に対する**差分追補**である。インフラ構成の正本性は ADR-0004 に残す。なお lean 化（issue #197）に伴う ALB 除去・EIP 直結・コスト運用（EventBridge stop/start・手動 destroy）は ADR-0004 本文にも反映済みであり、CloudFront 直結・オリジン保護簡素化の判断は本 ADR を参照すること。

## 影響・結果

- **TLS 終端の移動**: TLS 終端は CloudFront が担う。`security.md` §11 が当初定めていた ALB での TLS 終端・ACM 証明書・ALB での HTTP→HTTPS リダイレクトは、CloudFront 側（`redirect-to-https` + デフォルト証明書）で代替される。lean 化（issue #197）に伴い ALB が消えたため、§11 本文の「TLS 終端」「HTTP リダイレクト」「EC2 との通信」は CloudFront 直結構成に更新済み（本文反映の正本は本 ADR）。
- **クライアント IP 取得**: lean 構成ではプロキシ段は CloudFront 1 段のみ（ALB 除去）。アプリのレート制限はクライアント IP をキーにするため、`X-Forwarded-For` の解釈を `TRUSTED_PROXY_COUNT` 方式とし、**prod=1**（CloudFront 1 段。当初 ALB 併設時は 2 だったが issue #197 で 1 へ変更）、dev=0 とする（issue #185 T1 / B-2-c、issue #197）。修正しないとレート制限が誤動作する。`security.md` §4.1 の「IP 取得」も本文更新済み。
- **CORS**: prod の `CORS_ALLOWED_ORIGINS` は CloudFront ドメイン（`https://<cloudfront_domain_name>`）になる（ALB 除去前後で不変）。`env_config.md` §3.1 / §4.5 に反映する。
- **EC2:8080 直アクセスの遮断**: B-1-b の SG（CloudFront prefix list 限定）により EC2 EIP:8080 への直接 HTTP アクセスは SG で遮断され、エンドユーザーは CloudFront ドメイン経由でのみ到達できる（当初の「ALB 直アクセスを 403」から「EC2:8080 直アクセスを SG で遮断」へ）。
- **コスト**: CloudFront は永久無料枠内のため実質 $0。ALB 除去で ~$18-20/月 を削減（コスト試算は ADR-0004）。EC2・RDS・S3・EIP の費用は ADR-0004 のポートフォリオ対応／lean コスト節を参照。
- **将来の独自ドメイン移行・スケール**: 独自ドメイン+ACM を導入する場合、CloudFront の代替ドメイン名（CNAME）と ACM 証明書（us-east-1）を設定し viewer 側 TLS 1.2+ を強制（B-5 解消）、オリジン側も HTTPS 化（B-3 解消）できる。本番でスケールし実データを扱う段階では ALB+ASG の再導入とあわせて X-Origin-Verify 検証 or mTLS でオリジン保護を多層化する（「なぜ多層から SG 一枚へ簡素化したか」参照）。

## 反映先

| 反映先文書 | 反映内容 |
|-----------|---------|
| `../architecture.md` §2 | システム全体構成: リクエスト経路を CloudFront 直結に更新（`Browser → CloudFront → EC2(EIP):8080`、ALB 除去） |
| `../architecture.md` §6.1 | 多層防御 [1] ネットワーク層: TLS 終端を CloudFront、オリジン保護を SG 1 層（CloudFront prefix list 限定）に更新 |
| `../diagrams.md` §1 | システム構成図: ALB ノードを除去し CloudFront → EC2(EIP:8080) 直結に更新 |
| `../diagrams.md` §2 | リクエスト処理フロー: ALB を経路から除去、X-Origin-Verify 検証記述を削除 |
| `../../70_operations/env_config.md` §3.1 | インフラ構成差分: ロードバランサ行を ALB 除去（lean）に、CDN を CloudFront 直結に、TLS / CORS を反映 |
| `../../70_operations/env_config.md` §4 | 環境変数 `TRUSTED_PROXY_COUNT`（dev=0 / prod=1。CloudFront 1 段） |
| `../../../progress-management/tickets/step11/11-E-deploy.md` §11 Q2 | 「案1（HTTP のみ）採用」記述に本 ADR への追補ノートを追加（履歴は残す） |
| `../../50_detail_design/security.md` §4.1, §11 | 構成変更（ALB 除去・TLS 終端の CloudFront 移行・trusted_proxy_count・オリジン保護 SG 1 層）は本文に反映（据え置きは B-3/B-5 の TLS 逸脱に限る）。判断の正本は本 ADR |

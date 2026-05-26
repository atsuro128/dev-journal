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

### 実装計画（2026-05-20 確定）

architect が実装計画 v2 を策定、reviewer 再レビューで PASS（旧 blocker 3 / warning 6 全解消、新規 blocker なし）。設計判断 B-1〜B-4 をユーザー確定。

#### 確定した設計判断

- **B-1 = B-1-b（完全閉域）**: ALB SG inbound 80 を CloudFront マネージドプレフィックスリスト（`com.amazonaws.global.cloudfront.origin-facing`）限定 ＋ CloudFront→ALB のカスタムヘッダ秘密値を ALB リスナールールで検証（不一致は 403）。他者の自前 CloudFront 経由の到達も遮断する。
- **B-2 = B-2-c**: `remoteIP` を `TRUSTED_PROXY_COUNT` 方式に修正。実クライアント IP = `XFF[len - TRUSTED_PROXY_COUNT]`、prod=2（CloudFront 追記1 + ALB 追記1）、dev=0。
- **B-3 = 受容**: CloudFront〜ALB 間 HTTP を ADR-0007 に「受容する逸脱」として記録。`security.md` §441 本文は不変。
- **B-4 = 追補ノート**: 11-E チケット §11 Q2 の「案1 採用」記述は履歴として残し、追補ノートで ADR-0007 へ誘導。
- **B-5 = 受容（2026-05-20 追加）**: CloudFront デフォルト証明書（`*.cloudfront.net`）は viewer 側 TLS 最小バージョンが `TLSv1`（1.0）固定で引き上げ不可（AWS 仕様）。`security.md` §11「TLS 1.2 以上」と乖離するが、TLS 1.2+ 強制には独自ドメイン + ACM が必要でコスト発生のため、$0 維持で受容。ADR-0007 に B-3 と並べて「受容する逸脱」として記録。`security.md` §11 本文は不変。codex PR #151 再レビューで検出。

#### remoteIP フォールバック確定仕様（T1）

`n = TRUSTED_PROXY_COUNT`（既定 0）、`parts` = XFF をカンマ分割・trim・空要素除外した配列とする:

1. `n==0`（dev）: XFF を完全無視し `RemoteAddr` を採用。
2. `len(parts) < n`: `RemoteAddr` にフォールバック。
3. `len(parts) >= n`（n>=1）: `parts[len(parts) - n]` を実クライアント IP として採用。
4. XFF ヘッダ不在、または trim・空要素除外後に有効要素 0: `RemoteAddr` を採用（W-C 反映）。

#### タスク分解

| ID | 内容 | 担当 | 依存 | ブランチ |
|----|------|------|------|---------|
| T1 | `remoteIP` を TRUSTED_PROXY_COUNT 方式へ修正（logger.go / config.go / main.go / テスト） | backend-developer | なし | `fix/185-trusted-proxy-remoteip` |
| T3 | CloudFront 構成（cloudfront.tf 新設 / SG プレフィックスリスト限定 / カスタムヘッダ検証 / 2 ビヘイビア / outputs・variables） | platform-builder | T1 マージ後 | `fix/185-cloudfront-distribution` |
| T4 | `CORS_ALLOWED_ORIGINS` を CloudFront ドメインに、`TRUSTED_PROXY_COUNT=2` を prod 投入 | platform-builder | T3 | T3 PR に内包可 |
| T5 | 設計書整合: 11-E line 1317 ADR 参照訂正・line 280/330 §3.5 参照訂正・§11 Q2 追補ノート・ADR-0007 新規・architecture.md 構成図 | architect/designer | T1 着手後随時 | 設計成果物フロー |
| T6 | CloudFront 経由の疎通・CORS・HSTS・レート制限検証 | test-implementer + 指揮役 | T4 | — |

#### warning 反映（チケット必須事項）

- **W-C**: T1 の `remoteIP` 仕様に「XFF 空値ヘッダ・空要素のフォールバック」を含める（上記フォールバック仕様 規則 4 / 空要素除外に反映済み）。テストケース「XFF 空値ヘッダ → RemoteAddr」を追加。
- **W-A**: T3 完了条件に「`/health` の CloudFront キャッシュ挙動（キャッシュ対象外 or 実害なし）」を一文明記。
- **W-B**: T5 の ADR-0007 にバイパス対策方式（B-1-b 採用）と残存リスクの扱いを記録。

#### issue #184 との関係

issue #184（SPA ハンドラ HEAD 405）は独立 issue として別 PR で対応。T1 とは `cmd/server/main.go` を共有するため直列実行。#185 の T3 apply 前にマージする。

---

## 解決内容

### マージ済み PR

| PR | 内容 | マージ日 |
|----|------|---------|
| #149 | T1: `remoteIP` を TRUSTED_PROXY_COUNT 方式に修正（logger.go / config.go / main.go + テスト） | 2026-05-20 |
| #150 | #184（SPA fallback HEAD 405）副次解消 | 2026-05-20 |
| #151 | T3: CloudFront 構成（cloudfront.tf 新設、SG プレフィックスリスト限定、X-Origin-Verify ヘッダ検証、2 ビヘイビア、outputs/variables）| 2026-05-20 |
| #154 | 緊急 fix: SG description 日本語混入 + create_before_destroy 対応（本セッション中に発覚） | 2026-05-26 |

T5（設計書整合・ADR-0007）は dev-journal commit `1925081` / `2155854` で完了済み（W-1 は副次 #186 として既に起票・resolved 済み）。

### Step 11-E 実 apply セッション (2026-05-26) で完遂した残作業

- **T4 (`CORS_ALLOWED_ORIGINS` を CloudFront ドメインに更新、`TRUSTED_PROXY_COUNT=2` を prod 投入)**: terraform.tfvars を更新し 2 段階目 apply で反映。
- **T6 (CloudFront 経由疎通・CORS・HSTS・レート制限検証)**:
  - 疎通: `https://djhmwtrr79jdq.cloudfront.net/health` = 200 (`{"status":"ok","checks":{"database":"ok"}}`)
  - ALB 直接 (`http://expense-saas-portfolio-alb-...`): タイムアウト = SG が CloudFront プレフィックスリスト限定で接続不可（B-1-b 2 層目動作）
  - HSTS: `Strict-Transport-Security: max-age=31536000; includeSubDomains` 設定確認
  - その他: `X-Content-Type-Options: nosniff` / `X-Frame-Options: DENY`
  - CORS preflight: 許可 origin (`https://djhmwtrr79jdq.cloudfront.net`) は `Access-Control-Allow-Origin` 返却、非許可 origin (evil / 旧 ALB DNS) は未返却
  - レート制限: POST `/api/auth/login` を 15 連投で 6 回目から 429 発火（`LoginRateLimitPerMinute=5` 想定通り）

### B-1-b 2 層防御の動作確認結果

- **1 層目（カスタムヘッダ X-Origin-Verify、常時有効）**: ALB SG が 0.0.0.0/0 のままだった 1 段階目 apply 完了直後、ALB DNS 直接アクセスで 403 を確認。
- **2 層目（SG プレフィックスリスト限定、`restrict_alb_to_cloudfront=true` で有効化）**: 2 段階目 apply 後、ALB DNS 直接アクセスで接続タイムアウトを確認（CloudFront 由来トラフィックのみ受信）。

両層とも実環境で動作確認済み。

### 派生 issue

- **#189** (新規起票): 本 PR #154 の起因となった SG description 日本語混入の品質追跡用。再発防止の lint ルール検討は post-MVP。

## 解決日
2026-05-26

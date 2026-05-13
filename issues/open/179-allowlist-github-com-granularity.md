# devcontainer egress allowlist の github.com 全許可によるツール install リスク（issue #060 派生、post-MVP）

## 発見日
2026-05-13

## カテゴリ
infrastructure / security / governance / post-mvp

## 影響度
中（実害は agent ガバナンスでカバー可能だが、攻撃面として残存）

## 発見経緯
proactive — Step 11-E Phase 1 で platform-builder が事前承認なく OpenTofu バイナリを github.com からダウンロードし `/home/node/bin/terraform` として install した事案を受けて、egress allowlist の粒度問題を確認。

## 関連ステップ
- Step 11-E Phase 1 着手時に検出
- 対応は MVP リリース後（インフラ整備スプリント）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外として扱う。MVP リリース前には対応しない。管理方式は **ops-080** で検討中。

## 問題

`.devcontainer/proxy-allowlist.txt` で `github.com` をドメイン全体で許可している。issue #060「devcontainer egress allowlist の厳格化と根拠の欠如」の対象範囲だが、現状は `gh` CLI / git clone / MCP server install / その他 OSS バイナリ取得のために github.com 全体を許可せざるを得ない状態。

### 攻撃面

allowlist に github.com が含まれているため、devcontainer 内のエージェント・スクリプトは:
- 任意の github.com 上のリリースバイナリを `wget`/`curl` でダウンロード可能
- ダウンロードしたバイナリを `chmod +x` して `/home/node/bin/` 等に配置可能
- PATH 経由で任意のコマンド名で実行可能

これは「最小 egress」の方針（issue #060/#061）が想定していた攻撃面の盲点。

### 実例（2026-05-13 検出）

Step 11-E Phase 1 で platform-builder agent が以下を実施:
1. ADR-0004 で Terraform 採用と決まっていた
2. Terraform CLI install 元（releases.hashicorp.com）が allowlist 未許可
3. **エージェントが独自判断で github.com から OpenTofu バイナリをダウンロード**
4. `/home/node/bin/terraform` として配置（リネームで偽装）
5. lock ファイル生成・PR 作成まで OpenTofu で実施
6. 後続 codex レビューで ADR との乖離として検出

技術的に allowlist 上は許可された操作だが、**ADR 採用判断のサイレント置換**という重大ガバナンス問題に発展。

## 影響

### MVP リリース判定

なし。Step 11-E（デプロイ・スモークテスト）のブロッカーにならない。今回は memory `feedback_no_silent_tool_install.md` の追加と Dockerfile への正規 install 追加で個別対応済み。

### セキュリティ攻撃面

- agent / スクリプトが任意の github.com 上のバイナリを install できる状態
- 悪意ある fork / supply chain 攻撃のリスクは理論上残存
- ただし agent 自体が悪意なく動作する前提なら実害は低い

### ガバナンス

- ツール install をホワイトリスト管理する仕組みが不在
- agent の操作可能範囲が allowlist 粒度より広い

## 提案

### 対応タイミング

**MVP リリース後**（Step 11-F UAT 完了後）。インフラ整備スプリントの一部として扱う。

### 対応プラン

#### 案 A: allowlist を org / repo 粒度に絞る

`github.com` 全許可を以下のような粒度に絞る:
- `github.com/golang` — Go 公式
- `github.com/hashicorp` — HashiCorp（Terraform 等）
- `github.com/github` — GitHub MCP server
- `github.com/anthropics` — Anthropic 公式リソース
- `github.com/cli/cli` — gh CLI
- `github.com/dandavison/delta` — git-delta
- `github.com/deluan/zsh-in-docker` — zsh-in-docker

ただし squid proxy で path-based filter が可能か確認が必要（HTTPS だと CONNECT method で path 見えない、CA 証明書差し込み等の MITM 設定が必要）。

#### 案 B: squid + ssl_bump で HTTPS インスペクション

既存の squid-openssl + `/etc/squid/ssl` 構成を拡張して ssl_bump を有効化、HTTPS の URL パスでフィルタする。実装複雑度高、CA 証明書を全クライアントに信頼させる必要あり。

#### 案 C: agent governance で補完（即時対応・MVP 中も実施可能）

技術ではなく運用で縛る:
1. memory `feedback_no_silent_tool_install.md` を全 agent プロンプトに反映（**今回実施済み**）
2. ai-dev-framework/agents/ の agent プロンプトテンプレートに「事前承認なく install しない」を明示
3. 認可された install のみ Dockerfile に記載（issue #060 で議論済み）

**推奨**: 案 C で当面運用、MVP 後に案 A/B を検討。

### 受容方針（MVP 期間中）

案 C のガバナンス補完で現状維持。`.devcontainer/proxy-allowlist.txt` の github.com 全許可を継続。

## 関連 issue / PR

- 関連 issue: #060「devcontainer egress allowlist の厳格化と根拠の欠如」（親）
- 関連 issue: #061「devcontainer マウントとシークレット露出の最小化」（兄弟）
- 関連 PR: #145（Step 11-E Phase 1、本 issue の発見元）
- 関連 memory: `feedback_no_silent_tool_install.md`（即時対応）
- 関連: ops-080「Post-MVP スコープ管理方法」

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

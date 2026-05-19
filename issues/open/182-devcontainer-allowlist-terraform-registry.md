# devcontainer egress allowlist に registry.terraform.io が含まれず terraform validate 実行不可（issue #179 系統）

## 発見日
2026-05-19

## カテゴリ
infrastructure

## 影響度
低

## 発見経緯
proactive

## 関連ステップ
Step 11-E Phase 3（terraform validate 実行時）

## ブロッカー
なし

## 問題

devcontainer に Terraform 1.9.8 が install 済み（ADR-0004 / コミット `695703b`）だが、`terraform validate` を実行するには `terraform init` が必要で、init は `registry.terraform.io` から provider バイナリを取得する。しかし devcontainer の egress allowlist に `registry.terraform.io` が含まれていないため init が `Forbidden` で失敗する。

具体的なエラー:

```
Error: Failed to query available provider packages
Could not retrieve the list of available versions for provider hashicorp/aws:
could not connect to registry.terraform.io: failed to request discovery document:
Get "https://registry.terraform.io/.well-known/terraform.json": Forbidden
```

結果として、devcontainer 内で利用できる Terraform 機能は `terraform fmt` のみで、`validate` / `plan` / `apply` は実行不可。

## 影響

- ローカル CI（`/test` 相当）で `terraform validate` をスキップせざるを得ない
- CI または Windows ホスト側に validate を委ねる必要がある
- 関連 PR: PR #146、PR #147 で `terraform validate` 未実行のままマージ（`terraform fmt` のみで PASS）
- devcontainer に Terraform を install している意義が「fmt 専用」に限定されており、ADR-0004 の意図（環境統一）と乖離している

## 提案

post-MVP で以下のいずれかを検討:

1. **devcontainer egress allowlist に `registry.terraform.io` を追加**
   - issue #179（github.com 全許可問題）と合わせて allowlist 粒度化を検討
   - `releases.hashicorp.com` は既に許可済み（Terraform install 用）
2. **devcontainer に provider バイナリを事前バンドル**
   - image build 時に `terraform init` 済み状態にする
   - provider バージョン更新時の追従コストが発生
3. **devcontainer での Terraform 利用は fmt 限定として明文化**
   - validate / plan / apply は CI に委ねる方針を ADR-0004 補足に追記
   - 現状を追認する形で運用コストを最小化

### 着手タイミング

- Step 11-F UAT 完了後、または devcontainer 関連 issue（#060 / #179）の整理時に合わせて検討

### 参照

- issue #179（devcontainer egress allowlist の github.com 全許可、post-MVP）
- ADR-0004（インフラ構成）
- commit `695703b`（devcontainer に Terraform 1.9.8 install）

---

## 解決内容

## 解決日

# terraform/ec2.tf: SSM Session Manager + SSM Parameter Store 対応（issue #186 設計確定に追従）

## 発見日
2026-05-24

## カテゴリ
infrastructure

## 影響度
中

## 発見経緯
escalation

## 関連ステップ
Step 11-E / issue #186（UD-1 + UD-5 設計確定）

## ブロッカー
なし（着手は #186 Phase 0 完了後、Step 11-E 実デプロイ時に同期実施を想定）

## 問題

issue #186 の Phase 0 ユーザー判断で以下が確定した:

- **UD-1 = A**: 接続方式を SSH → SSM Session Manager に切り替え
- **UD-5 = B**: シークレット注入方式を Terraform variable + user_data 直接埋め込み → SSM Parameter Store（SecureString）+ systemd EnvironmentFile に切り替え

これに伴い、`expense-saas/infra/terraform/` 配下の Terraform 構成およびユーザーデータスクリプトを SSM ベースに移行する必要がある。現状実装は以下の通り:

| 構成要素 | 現状 | 目標（#186 設計確定） |
|---|---|---|
| EC2 接続 | SSH（`key_name` 指定、SG 22 番開放） | SSM Session Manager 経由（ポート開放不要、IAM 認証） |
| シークレット | Terraform variable + `user_data.sh.tpl` で平文埋め込み | SSM Parameter Store SecureString + 起動時 fetch + systemd EnvironmentFile |
| IAM | EC2 インスタンスプロファイルに最小限の権限 | + `AmazonSSMManagedInstanceCore` + `ssm:GetParameters` + `kms:Decrypt` |
| Security Group | 22 番 (SSH) インバウンド許可 | 22 番ルール削除（443 アウトバウンドのみで SSM 動作） |

## 影響

- **運用負荷の劇的軽減**: 自宅 Wi-Fi の IP 変動時に SG 更新が不要になる（接続元 IP に依存しない）
- **セキュリティ向上**:
  - SSH 鍵管理が不要（IAM 認証のみ）
  - 22 番ポートを世界に公開しない（SG インバウンドゼロ）
  - 接続ログが CloudTrail に残り監査可能
  - シークレットが KMS 暗号化保存、tfvars / user_data から平文除去
- **シークレットローテーション容易化**: SSM Parameter Store の値更新 + `systemctl restart app` のみで完結（terraform apply 不要）
- **Step 11-E 実デプロイ時の手順**: 本 issue の実装と #186 の運用ドキュメント書き直しが整合する必要あり

## 提案

issue #186 §実装計画 §6.3.1 の仮素案を実装に落とし込む。具体的なスコープと変更箇所:

### スコープ（`expense-saas/infra/terraform/` 配下）

1. **`iam.tf` 修正**:
   - EC2 インスタンスプロファイルに `arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore` をアタッチ
   - SSM Parameter Store アクセス用カスタムポリシーを追加（`ssm:GetParameters` を対象パラメータ ARN に限定）
   - KMS 復号権限を追加（`kms:Decrypt` を SecureString 暗号化に使う KMS キーに限定）

2. **`ec2.tf` 修正**:
   - `key_name` 属性を削除（SSH 鍵を払い出さない）
   - 不要になる関連変数があれば clean up

3. **`sg.tf`（または該当ファイル）修正**:
   - 22 番ポート (SSH) のインバウンドルールを削除
   - 443 アウトバウンド許可は維持（SSM エンドポイント疎通用）

4. **`user_data.sh.tpl` 修正**:
   - SSM Agent インストール処理を追加（Amazon Linux 2/2023 はプリインストール済みなら確認のみ）
   - SSM Parameter Store からシークレットを `aws ssm get-parameters --names ... --with-decryption` で取得する処理を追加
   - 取得した値を systemd EnvironmentFile に書き出す処理を追加
   - Terraform variable から直接埋め込む既存処理を削除（平文化を回避）
   - 取得失敗時の起動失敗 + ログ出力（fail-fast）を実装

5. **`variables.tf` 修正**:
   - SSM パラメータ名を変数化（例: `var.db_password_ssm_param_name`）
   - SSM パラメータの暗号化に使う KMS キー ARN を変数化（または default KMS でも可）

6. **SSM Parameter Store パラメータ作成**（手動 or 別 .tf）:
   - DB パスワード等の SecureString パラメータを SSM に作成
   - Terraform で管理するか、手動作成 + Terraform 参照にするかは別途判断

### 受け入れ基準

- [ ] `aws ssm start-session --target i-xxxxx` で EC2 に接続できる
- [ ] SSH（22 番）での接続が不可（SG ルール削除確認）
- [ ] SSM Parameter Store から DB パスワードが起動時に取得され、systemd EnvironmentFile に書き出されている
- [ ] tfvars / user_data 出力内に平文シークレットが残っていない
- [ ] CloudTrail に SSM 接続ログが記録されている
- [ ] `aws ec2 describe-instance-attribute --attribute userData` で user_data に平文シークレットが含まれていない
- [ ] アプリ起動・ヘルスチェック疎通確認 OK

### 着手タイミング

- **issue #186 Phase 0 確定後**（2026-05-24 確定済み）
- **issue #186 Phase 3 完了後 or Step 11-E 実デプロイ時**に併せて実施するのが効率的（運用ドキュメント書き直しと Terraform 修正の同期）

### 留意点

- SSM Parameter Store のパラメータ作成（特に SecureString の初期値）は別途手動 or 別 Terraform スクリプトが必要
- KMS キーは AWS マネージドキー（`alias/aws/ssm`）で十分（無料枠内）
- 既存の EC2 インスタンスへの適用は `terraform apply` で再作成 or インスタンスプロファイル付け替え + user_data 更新 + 手動 systemctl restart のいずれか
- 本 issue は **`expense-saas/` 配下の PR フロー**（dev-journal の設計成果物フローではない）

---

## 解決内容

PR #152（expense-saas、squash commit `445240b`）でマージ完了。

### 確定したユーザー判断（計画書 v2、§3）

- **P-0=A**: `terraform.tfvars` から JWT PEM 2 件（`jwt_private_key_pem` / `jwt_public_key_pem`）を削除。DB password 3 件は RDS `master_password` で Terraform 必須のため残置
- **P-1=B**: SSM Parameter は手動 `aws ssm put-parameter` で投入。`ssm.tf` / `aws_ssm_parameter` リソースは作成しない（IaC 一貫性より機密分離優先）
- **P-2=A**: KMS は AWS managed key `alias/aws/ssm`
- **P-3=A**: 既存 EC2 は `user_data_replace_on_change = true` + terraform apply で再作成
- **P-4=B**: AWS 実環境疎通系の受け入れ基準（#1/#5/#7）は Step 11-E 実 apply セッションに切り出し。本 PR は静的検証（fmt / grep / 構造レビュー）まで
- **P-5=A**: SSH 関連変数（`key_pair_name` / `allowed_ssh_cidr`）完全削除、SG 22 番 ingress 削除、`ec2.tf` から `key_name` 削除
- **P-6=B**: 非機密項目（CORS / TRUSTED_PROXY_COUNT）は Terraform variable のまま
- **P-8=A**: env_config.md §5.3.3 / §5.3.5 を同 PR スコープで同時修正（dev-journal は別コミット）

### 実装変更（PR #152、計画書 §4.1 T1-T8）

| ファイル | 変更内容 |
|---|---|
| `infra/terraform/iam.tf` | `AmazonSSMManagedInstanceCore` アタッチ + `ec2_ssm_parameters` inline policy 追加（`/expense-saas/*` 全 env 共通）。`ec2_kms_decrypt` は B-05 修正で削除 |
| `infra/terraform/ec2.tf` | `key_name` 削除、`templatefile` 引数に `s3_bucket` / `ssm_parameter_path_prefix` 追加、`user_data_replace_on_change = true` 追加、`depends_on` に SSM policy 2 件追加 |
| `infra/terraform/security_groups.tf` | L94-101 SSH (22 番) ingress 削除 |
| `infra/terraform/user_data.sh.tpl` | SSM Parameter Store fetch（`jq -r` + `--output json`、PEM 多行対応）+ 機密区間 `/dev/null` リダイレクト + `install -m 0600` atomic permission + 5 回 retry 実装 |
| `infra/terraform/variables.tf` | `key_pair_name` / `allowed_ssh_cidr` / `jwt_private_key_pem` / `jwt_public_key_pem` 削除、`ssm_parameter_path_prefix` 追加 |
| `infra/terraform/terraform.tfvars.example` | SSH 関連削除、SSM 投入手順コメント追加 |
| `infra/terraform/outputs.tf` | `ec2_public_ip` description 更新、`ec2_instance_id` output 追加 |
| `infra/terraform/README.md` | SSM 前提の apply 手順、KMS 明示ポリシー不要の理由を記載 |

### dev-journal 側コミット

- `e11f90e` env_config.md §5.3.3 (awk → jq) + §5.3.5 (IAM Resource 差分注記)（B-01 / B-02）
- `ac9e476` 計画書 v2 を `progress-management/working/187-architect-plan.md` にアーカイブ
- `db79487` env_config.md §5.3.5 B-05 修正注記（kms:Decrypt 明示ポリシー削除）

### レビューサイクル

- **architect 計画書**: v1 → reviewer CONDITIONAL PASS (B-01/B-02/B-03 + W-01〜W-05) → v2 → reviewer PASS
- **内部レビュー**: 初回 FAIL (B-04 S3_BUCKET 名不一致 / W-01 PEM 一時 permission) → 修正 (`2a3e81b`) → PASS
- **codex レビュー**: 初回 FAIL (B-05 kms:Decrypt の alias ARN 無効 / B-06 EC2 が IAM policy に depends_on なし) → 修正 (`862a3a4`) → PASS

### 本 issue スコープ外（Step 11-E 実 apply で実施）

受け入れ基準のうち AWS 実環境疎通系:
- (1) `aws ssm start-session --target i-xxxxx` で EC2 接続
- (2) SSH (22 番) 接続不可確認（実環境 nmap 等）
- (3) SSM Parameter Store fetch → `/etc/expense-saas/app.env` 確認
- (5) CloudTrail に SSM 接続ログ記録
- (6) `aws ec2 describe-instance-attribute --attribute userData` で平文不在
- (7) アプリ起動 + ヘルスチェック疎通

これらは PR #152 マージ時点では未検証。Step 11-E 実 apply セッションで `terraform apply` 実施時に併せて消化する。

### 派生 issue

- **本 issue から新規起票なし**（codex 指摘 B-05/B-06 は本 PR 内で全て解消）

## 解決日
2026-05-25

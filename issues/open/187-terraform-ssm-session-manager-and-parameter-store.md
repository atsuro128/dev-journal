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
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

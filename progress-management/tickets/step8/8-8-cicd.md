# CI/CD パイプライン

- 担当: platform-builder
- 依存: 8-2, 8-3
- 出力先: expense-saas/.github/workflows/
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | デプロイ方針 |

## 責務

- CI パイプライン（PR 時: lint → test → build）
- デプロイ後のワークフロー定義（E2E/スモークは枠のみ）
- ブランチ保護ルール
- 判断ポイント: ステージ構成、トリガー条件、ブランチ保護、マージ前チェック、デプロイ方式
- 含めない: 本番デプロイの実装（Step 8 スコープ外）
- 参考: ops-042（CI/CD テンプレート化 issue）の判断を反映する

## 完了条件

- CI パイプラインが PR 時に lint / test / build を自動実行する

# CI/CD パイプライン

- 担当: platform-builder
- 依存: 8-2, 8-3
- ブランチ: main 直接
- 出力先: expense-saas/.github/workflows/
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | §デプロイ |

## 責務

- CI パイプライン（PR 時: lint → test → build）
- デプロイ後のワークフロー定義（E2E/スモークは枠のみ）
- ブランチ保護ルール
- 含めない: 本番デプロイの実装（Step 8 スコープ外）

## 完了条件

- CI パイプラインが PR 時に lint / test / build を自動実行する

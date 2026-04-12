# デプロイ・スモークテスト

- 担当: platform-builder
- 依存: 11-D
- ブランチ: `step11/11-E-deploy`
- 出力先: expense-saas/（deploy.yml 等）、AWS リソース
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| 運用設計（リリース） | deliverables/docs/70_operations/release.md | リリース手順 |
| 運用設計（環境設定） | deliverables/docs/70_operations/env_config.md | 環境変数・シークレット |
| アーキテクチャ設計 | deliverables/docs/30_architecture/architecture.md | インフラ構成 |
| ADR-0004 | deliverables/docs/30_architecture/adr/0004-environment-config-portfolio.md | ポートフォリオ対応方針 |
| CI/CD パイプライン | expense-saas/.github/workflows/deploy.yml | デプロイステップ |

## 責務

- AWS リソース構築（ADR-0004 のポートフォリオ対応方針に基づき、EC2 t3.micro / RDS / S3）
- deploy.yml のデプロイステップ有効化（Docker ビルド → ECR プッシュ → デプロイ）
- ヘルスチェック疎通確認
- スモークテスト: 主要フロー1本（申請 → 承認 → 支払）をデプロイ済み環境で手動実行
- 含めない: 本番運用設定（監視・アラート等）、テストコードの追加

## 完了条件

- デプロイ済み環境で第三者がアクセスできる状態になっている
- ヘルスチェックエンドポイントが応答する
- 主要フロー1本（申請 → 承認 → 支払完了）がデプロイ済み環境で通る

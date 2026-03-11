# expense-saas ディレクトリ構成

プロダクト本体（経費精算SaaS）。実装コードのみを管理する。

```
expense-saas/
├── README.md
├── .gitignore
├── .github/
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── workflows/
│       └── explanation.md
├── apps/
│   ├── api/                     # バックエンド API (Go)
│   │   ├── go.mod
│   │   ├── cmd/
│   │   └── internal/
│   └── web/                     # フロントエンド (React / TypeScript / Vite)
│       ├── package.json
│       └── src/
├── packages/                    # フロントエンド共有パッケージ (npm workspaces)
│   ├── config/                  # ESLint / TSConfig 等の共通設定
│   ├── types/                   # TypeScript 型定義
│   └── ui/                      # 共有UIコンポーネント (shadcn/ui)
├── database/                    # マイグレーション / シード / ERD
├── docker/                      # ローカル開発用コンテナ
├── docs/                        # 公開向けドキュメント（実装フェーズで配置）
│   ├── api.md
│   ├── architecture.md
│   ├── runbook.md
│   └── tech-stack.md
├── infra/                       # IaC（Terraform 等）
├── scripts/                     # プロダクト側スクリプト
└── tests/                       # E2E / 統合テスト
```

## 構成の判断根拠

- **`apps/api/`**: Go プロジェクトのため `go.mod` で管理。npm の管轄外。
- **`packages/`**: フロントエンド (TypeScript) 側の共有パッケージのみを配置。DB アクセスは Go 側で `database/` のマイグレーションを参照する。
- **`database/`**: マイグレーションファイルを格納。Go バックエンドから参照する。
- **`docs/`**: 公開向けドキュメント（architecture, api, runbook, tech-stack）を配置。設計プロセスのドキュメントは `dev-journal/deliverables/docs/` で管理する。
- **`.github/`**: PR テンプレート・CI ワークフロー定義を配置。

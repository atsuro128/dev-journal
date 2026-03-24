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
│   │   └── src/
│   │       └── explanation.md
│   └── web/                     # フロントエンド (React / TypeScript / Vite)
│       ├── package.json
│       └── src/
│           └── explanation.md
├── packages/                    # フロントエンド共有パッケージ (npm workspaces)
│   ├── config/                  # ESLint / TSConfig 等の共通設定
│   │   └── explanation.md
│   ├── types/                   # TypeScript 型定義
│   │   └── explanation.md
│   └── ui/                      # 共有UIコンポーネント (shadcn/ui)
│       └── explanation.md
├── database/                    # マイグレーション / シード / ERD
│   └── explanation.md
├── docker/                      # ローカル開発用コンテナ
│   └── explanation.md
├── docs/                        # 公開向けドキュメント（実装フェーズで配置）
│   ├── api.md
│   ├── architecture.md
│   ├── runbook.md
│   └── tech-stack.md
├── infra/                       # IaC（Terraform 等）
│   └── explanation.md
├── scripts/                     # プロダクト側スクリプト
│   └── explanation.md
└── tests/                       # E2E / 統合テスト
    └── explanation.md
```

## 構成の判断根拠

- **`apps/api/`**: Go プロジェクト。基盤構築フェーズで `go.mod`, `cmd/`, `internal/` を配置予定。現在は `explanation.md` のみ。
- **`packages/`**: フロントエンド (TypeScript) 側の共有パッケージのみを配置。DB アクセスは Go 側で `database/` のマイグレーションを参照する。
- **`database/`**: マイグレーションファイルを格納。Go バックエンドから参照する。
- **`docs/`**: 公開向けドキュメント（architecture, api, runbook, tech-stack）を配置。設計プロセスのドキュメントは `dev-journal/deliverables/docs/` で管理する。
- **`.github/`**: PR テンプレート・CI ワークフロー定義を配置。

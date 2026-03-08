# プロダクト（project/）ディレクトリ構成

```
project/
├─ README.md
├─ .github/
│  ├─ workflows/
│  └─ PULL_REQUEST_TEMPLATE.md
├─ apps/
│  ├─ api/                        # バックエンド API (Rust / Actix Web)
│  │  ├─ Cargo.toml
│  │  └─ src/
│  └─ web/                        # フロントエンド (React / TypeScript / Vite)
│     ├─ package.json
│     └─ src/
├─ packages/                      # フロントエンド共有パッケージ (npm workspaces)
│  ├─ config/                     # ESLint / TSConfig 等の共通設定
│  ├─ types/                      # TypeScript 型定義（フロント専用）
│  └─ ui/                         # 共有UIコンポーネント (shadcn/ui)
├─ database/                      # SQLx マイグレーション / シード / ERD
├─ docker/                        # ローカル開発用コンテナ
├─ docs/                          # 公開用設計・運用ドキュメント
│  ├─ architecture.md
│  ├─ api.md
│  ├─ tech-stack.md
│  └─ runbook.md
├─ infra/                         # IaC（Terraform 等）
├─ scripts/                       # プロダクト側スクリプト
└─ tests/                         # E2E / 統合テスト
```

## 構成の判断根拠

- **`apps/api/`**: Rust プロジェクトのため `Cargo.toml` で管理。npm の管轄外。
- **`packages/`**: フロントエンド (TypeScript) 側の共有パッケージのみを配置。`db/` は削除 — DB アクセスは Rust 側で SQLx を使い `database/` のマイグレーションを参照するため、npm パッケージとしての `db/` は不要。
- **`database/`**: SQLx のマイグレーションファイルを格納。Rust バックエンドから直接参照する。

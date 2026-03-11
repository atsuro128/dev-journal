# バックエンド技術スタックを Rust から Go に変更

## 発見日
2026-03-11

## カテゴリ
architecture

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
Step 3（アーキテクチャ設計）

## 問題

バックエンドの技術スタックとして Rust（Actix Web / SQLx）を選定していたが、以下の理由から Go への変更が妥当と判断された:

1. **学習コスト**: Rust 未経験からの Web バックエンド開発は 3ヶ月以上必要。Java / Python / Node.js の経験はあるが Rust 経験はゼロ
2. **ポートフォリオの完成リスク**: 未完成のポートフォリオは逆効果。Rust では完成までの期間が大幅に伸びる
3. **転職市場との適合**: 自社開発 SaaS 企業では Go の採用が多く、Rust は Web バックエンドでの採用が少ない
4. **面接対応**: AI にコードを書かせて学ぶ戦略は可能だが、面接でコードの意図を説明できるレベルに達するには十分な期間が必要

## 影響

### 修正が必要な成果物
- `root-project/CLAUDE.md` — 技術スタック記述
- `deliverables/docs/30_arch/adr/0001-tech-stack.md` — 選定理由の書き直し
- `.devcontainer/Dockerfile` — Go ベースに変更
- 詳細設計（Step 5）— ORM / クエリビルダの選定（SQLx → sqlc / GORM 等）
- `dev-journal/guide/portfolio_project_steps.md` — Rust 固有の記述を Go に更新
- `dev-journal/references/glossary.md` — 技術用語の更新（該当があれば）

### 影響を受けない成果物
- 要件定義（Step 1）— 技術スタック非依存
- ドメイン設計（Step 2）— 技術スタック非依存
- スコープ定義 — 機能スコープに変更なし
- テナント分離・RBAC・状態遷移の設計方針 — 言語に依存しない

## 提案

1. 技術スタックを **Go（標準ライブラリ or Echo/Chi）+ PostgreSQL（sqlc or pgx）** に変更
2. フロントエンドは **React (TypeScript, Vite)** を維持
3. ADR `0001-tech-stack.md` に変更理由を記録（Rust 不採用の理由も含む）
4. 既存の設計成果物を順次更新

---

## 解決内容
全ドキュメント・設定ファイルのバックエンド技術スタック記述を Rust から Go に更新。具体的なフレームワーク・ライブラリ選定（Echo/Chi, sqlc/pgx 等）は Step 3 ADR に委ねる方針。変更対象: CLAUDE.md, AGENTS.md, DevContainer (Dockerfile/devcontainer.json/init-firewall.sh), コーディング規約, セキュリティポリシー, テスト方針, settings.json, 実装手順書, 開発コマンド, ディレクトリ構成, 参考リンク集, 用語集, PROJECT_SUMMARY.md, README テンプレート, expense-saas/.gitignore, Cargo.toml 削除。

## 解決日
2026-03-11

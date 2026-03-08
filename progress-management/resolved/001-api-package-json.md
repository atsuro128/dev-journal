# project/apps/api/ のディレクトリ構成と技術スタックの不整合

## 発見日
2026-03-04

## 関連ステップ
Step 7 Phase 1（プロジェクト初期化時に対処）

## カテゴリ
architecture

## 問題
`project/apps/api/` に `package.json` が存在するが、バックエンドは Rust (Actix Web) であり `Cargo.toml` であるべき。
また、`project/packages/` 配下（ui / config / types / db）は Node.js モノレポのパターンであり、Rust + React 混在構成での必要性が不明確。

## 影響
Phase 1 着手時にプロジェクト初期化で混乱する可能性がある。

## 提案
- `apps/api/` を Cargo プロジェクトとして初期化（`package.json` → `Cargo.toml`）
- `packages/` の必要性を再検討（Rust/TS 混在モノレポで npm workspaces をどこまで使うか）

## 解決内容
`apps/api/package.json` を削除し `Cargo.toml`（expense-api）として初期化済み。
`packages/` の必要性再検討は issue-008 として分離。

## 解決日
2026-03-04

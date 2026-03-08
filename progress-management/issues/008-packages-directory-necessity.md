# project/packages/ ディレクトリの必要性再検討

## 発見日
2026-03-04

## 関連ステップ
Step 7 Phase 1（プロジェクト初期化時に対処）

## カテゴリ
architecture

## 問題
`project/packages/` 配下に `config/` / `types/` / `ui/` が存在するが、各ディレクトリには `explanation.md` のみ。
Rust + React 混在構成において npm workspaces の共有パッケージとしてどこまで必要かが不明確。

## 影響
Phase 1 着手時にディレクトリ構成の判断が必要になる。

## 提案
- React フロントエンド用の共有パッケージとして必要なもの（ui / types）は残す
- Rust 側で不要なもの（config / db）は削除または再構成
- Phase 1 で実際に使うタイミングで最終判断する

## 解決内容


## 解決日


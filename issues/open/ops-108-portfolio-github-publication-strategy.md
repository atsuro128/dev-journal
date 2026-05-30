# ポートフォリオ用 GitHub 公開戦略の未決（README 内 dev-journal リンクの整合性）

## 発見日
2026-05-29

## カテゴリ
project-management

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
post-MVP（ポートフォリオ公開準備）

## ブロッカー
なし

## 問題

`expense-saas/README.md` をはじめ、Phase 2 以降で作成予定の各 README（`root-project/`, `ai-dev-framework/`, `dev-journal/`）は、設計成果物・ADR・テスト戦略への参照を `../dev-journal/...` の相対パスで記述している。

しかし本プロジェクトは 4 リポジトリ（`expense-saas` / `dev-journal` / `ai-dev-framework` / `root-project`）に分けて管理しており、これらを GitHub に**別リポジトリとして公開した場合、相対リンクは壊れる**。

現時点では「4 リポジトリを GitHub にどう公開するか」の戦略自体が未決のため、リンク方式を決められない状態。

## 影響

- ポートフォリオとして GitHub 公開した際、README 内の参照が機能しない
- 採用担当者・エンジニア層に対し、設計判断の追跡可能性（ADR / 設計成果物 / テスト戦略）を訴求できない
- ローカルで clone して見る場合は問題なく動作するため、見落としやすい

## 提案

公開戦略を以下の選択肢から決定し、それに応じて README のリンク方式を統一する。

| 案 | 内容 | メリット | デメリット |
|---|---|---|---|
| A | 4 リポジトリをそれぞれ public 公開し、**絶対 URL**（`https://github.com/<user>/dev-journal/blob/...`）で参照 | 確実に辿れる。リポジトリ分離設計を維持 | dev-journal を公開する覚悟が必要（運用メモ含む） |
| B | 4 リポジトリを **monorepo** に集約して 1 リポジトリで public 公開 | 相対パスがそのまま使える | リポジトリ分離設計の意図を捨てる |
| C | 重要な抜粋（ADR 一覧・アーキ図・テスト戦略サマリ）を `expense-saas/docs/` にコピー | expense-saas 単独で完結 | 重複管理・原本との乖離リスク |
| D | リンクを削除し、テキストで「dev-journal リポジトリの ADR 参照」と書くのみ | シンプル | ワンクリックで辿れず参照価値が大幅減 |

### 決定後の対応タスク

- `expense-saas/README.md` の `../dev-journal/...` `../ai-dev-framework/...` 参照を一括書き換え
- Phase 2-4 で作成する各 README（`root-project/`, `ai-dev-framework/`, `dev-journal/`）も同じ方式で書く
- 案 C を採用する場合は `expense-saas/docs/` の構築タスクが追加される
- **「リポジトリ / ディレクトリ」用語の統一**: 現在 4 つの README で表現が混在している（root-project: 「4 つのリポジトリ」、expense-saas: 「4 つのディレクトリ」、dev-journal: 「関連リポジトリ」）。公開戦略決定時に「リポジトリ」「ディレクトリ」のどちらに統一するかも合わせて判断し、4 README を一括書き換え

### 検討の判断材料

- dev-journal には進捗・issue・session-log など運用ログも含まれており、公開時に整理が必要か（→ 案 A の場合に検討）
- monorepo にすると AI 駆動開発フレームワーク（ai-dev-framework）の汎用性が薄れる（→ 案 B の判断材料）
- 重要な情報だけ抜粋すると「プロセス全体を残した」という訴求が薄れる（→ 案 C のトレードオフ）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

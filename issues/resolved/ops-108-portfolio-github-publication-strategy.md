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

## 採用方針（2026-05-30 決定 → 2026-05-31 案 A に変更）

> **注記**: 当初は下記の案 C（ハイブリッド monorepo）を採用したが、構築途中で「コミット履歴が残らない」「統合版は不要」という判断から **案 A（4 リポジトリ別公開・絶対 URL 参照）に変更**した。最終的な対応は末尾の「解決内容」を参照。

**案 C: 開発は 4 リポジトリ構成を維持、公開用は別途 monorepo として作成（ハイブリッド）**

理由:
- 開発時の AI 駆動運用（CLAUDE.md / .claude/agents/ / 各リポジトリのルール）が 4 リポジトリ構造前提で書かれており、統合すると運用が壊れる
- ポートフォリオ提示時は採用担当者が「4 URL を行き来する」より「1 リポジトリで全体把握」できる方が訴求力が高い
- dev-journal の運用ログ（個人メモ・愚痴等）を公開判断する際、開発用と分離できる

### 用語統一の扱い

「リポジトリ / ディレクトリ」表現の混在は **公開用 root README に「分離開発 → 統合公開」の説明を追加することで吸収**。他 3 README（expense-saas / dev-journal / ai-dev-framework）の「リポジトリ」表記は文脈で正当化される（実際の開発では別リポジトリだから）。

### 公開用 monorepo の構成

- 配置: `root-project/.portfolio-build/`（`.gitignore` 済み）
- リポジトリ名: `expense-saas-portfolio`
- 4 ディレクトリをコピー（過去 git 履歴は持ち込まない、コピー時点でスナップショット）
- `.claude/settings.local.json` / `.claude/memory/` 等のセンシティブファイルは除外
- README は root のみ書き換え（「分離開発 → 統合公開」説明）、他 3 README は現状コピー

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

**案 A（4 リポジトリをそれぞれ public 公開し、絶対 URL で相互参照）を採用**して解決した。

### 案 C → 案 A への変更経緯（2026-05-31）

案 C（公開用 monorepo）で一度 `expense-saas-portfolio` リポジトリを private push まで進めたが、ユーザーから「コミット履歴がないのは良くない」「統合版は不要」との指摘を受け、以下の観点から案 A に方針変更した。

- コミット履歴（215 commit の開発プロセス）こそポートフォリオの訴求点
- 採用担当者の AI 警戒に対し、リポジトリ分離 + 履歴で「人間が設計判断した」ことを示せる
- pinned 4 リポジトリの実績量感

### 案 A の実装内容

1. **author email 統一**: `git filter-repo` で 215 commit（root-project 56 / expense-saas 147 / dev-journal 12）の author email を `atsuro128@users.noreply.github.com` に書き換え（個人メアド露出を排除）
2. **README の絶対 URL 化**: 各 README の `../other-repo/...` 相対参照を `https://github.com/atsuro128/{repo}/blob|tree/master/{path}` に置換
3. **4 リポジトリを public 化 + description 更新**: root-project / expense-saas / dev-journal / ai-dev-framework
4. **バックアップ**: `.author-fix-backup/` に 3 リポジトリの mirror clone を保持

### 用語統一（2026-06-01 完了）

issue 本文の残課題「リポジトリ / ディレクトリ用語の混在」を解消した。

- 実態は「外部の 4 開発単位 = リポジトリ」「各リポジトリ内部の構成 = ディレクトリ」の使い分けに概ね収束しており、ai-dev-framework / dev-journal は整合済みだった
- 不整合は root-project / expense-saas の「構成」セクションで見出し（リポジトリ）と表ヘッダ（ディレクトリ）が食い違う 2 箇所のみ
- 両 README の該当箇所を「リポジトリ」に統一（root-project: master 直コミット `635f27d` / expense-saas: PR #156 マージ `ebc3f50`）

### 公開済み 4 リポジトリ（PUBLIC）

| リポジトリ | URL |
|---|---|
| root-project | https://github.com/atsuro128/root-project |
| expense-saas | https://github.com/atsuro128/expense-saas |
| dev-journal | https://github.com/atsuro128/dev-journal |
| ai-dev-framework | https://github.com/atsuro128/ai-dev-framework |

### 残務（issue スコープ外）

- 案 C の残骸 `expense-saas-portfolio`（PRIVATE）は、delete_repo スコープを Claude に渡さない判断によりユーザーが GitHub UI で手動削除する

## 解決日
2026-05-31（公開・絶対 URL 化）/ 2026-06-01（用語統一・状態遷移）

# 2026-03-12 セッションログ

## 08:31 セッション
- 作業: 2026-03-11 の日報を作成（daily-reports/2026-03-11.md）
- 内容: git ログ・セッションログから昨日の実施内容（Dev Container 導入、Issue #018 Rust→Go 変更、hook 改善）・判断事項・明日のタスクを整理

## 14:08 セッション
- 作業: Auto Memory 機能の廃止
- 背景: Dev Container 環境ではコンテナ再ビルドで `~/.claude/` が揮発し、Auto Memory が消失する。チーム開発での共有も困難。公式ベストプラクティスでは永続的な知見は CLAUDE.md / .claude/rules/ に集約を推奨
- 実施内容:
  - `.claude/memory/` 内のファイル2件を `private-materials/` にアーカイブ移動
  - `.claude/memory/` ディレクトリを削除
  - `.claude/rules/workflow.md` のメモリ保存ルールを「Auto Memory 不使用」ルールに変更

## 14:30 セッション
- 作業: Git author 情報の統一と .gitconfig マウント設定
- 背景: リポジトリを public にする際、個人メールアドレス（`atsuro-1997@outlook.jp`）が全コミット履歴から露出する問題を解消
- 実施内容:
  - `git filter-repo` で4リポジトリ（root-project / dev-journal / ai-dev-framework / expense-saas）の全177コミットを書き換え
  - author: `atsuro128 <atsuro128@users.noreply.github.com>` に統一
  - 4リポジトリとも force push 完了
  - ホスト側 `git config --global` を `atsuro128` / noreply メールに更新
  - `.devcontainer/devcontainer.json` に `.gitconfig` の bind mount を追加（コンテナ内への自動反映）

## 14:48 セッション
- 作業: VS Code 全体検索（Ctrl+Shift+F）でネストされた独立 git リポジトリ内ファイルがヒットしない問題の修正
- 背景: root-project/.gitignore でサブリポジトリ（dev-journal/ / ai-dev-framework/ / expense-saas/）を除外していたため、VS Code がこれらのディレクトリを検索対象から除外していた
- 原因: VS Code は親ディレクトリの .gitignore を参照し、除外対象フォルダ内のファイルを全検索から除外する（`search.useParentIgnoreFiles` がデフォルト true）
- 対応: .gitignore からサブリポジトリの除外エントリを削除。ワークスペースファイル（root-project.code-workspace）は試行の結果不要と判断し削除
- 副作用: `git status` にサブリポジトリが untracked として表示されるが、明示的に `git add` しない限りコミット・push には含まれない

## 15:18 セッション
- 作業: 公開向け語句整理 — 「ポートフォリオ」「転職」「面接」等の練習用プロジェクトを示す語句を除去
- 方針: 中程度（deliverables・ガイド・AGENTS.md・ADR を対象、内部ログ・日報は据え置き）
- 対象ファイルと変更:
  - `PROJECT_SUMMARY.md`: 未編集のまま `private-materials/` へ移動（内部資料として保管）
  - `AGENTS.md`: 「ポートフォリオプロジェクト」→ プロダクト名のみに
  - `deliverables/docs/00_goals.md`: 概要・見出しから転職/ポートフォリオ表現を除去
  - `deliverables/docs/10_requirements/requirements.md`: 「ポートフォリオの現実的な目標」→「小〜中規模 SaaS の現実的な目標」
  - `guide/portfolio_project_steps.md`: タイトル・説明文・テスト見出しから除去
  - `guide/implementation-guide.md`: 目標・Step 22 見出しから除去
  - `references/decisions/ADR-001-repository-restructuring.md`: 4箇所を中立表現に
  - `references/decisions/ADR-002-ai-operations-redesign.md`: 3箇所を中立表現に
  - `ai-operations/overview.md`: 「ポートフォリオ」→「プロジェクト」
- 備考: コミット履歴の該当語句（4件）は公開時に squash で対応予定
- 追加作業: `portfolio_project_steps.md` → `project_steps.md` にリネーム。参照5箇所（CLAUDE.md, AGENTS.md, review-procedure.md, Issue #005, directory-structures/dev-journal.md）を更新

## 16:09 セッション
- 作業: Dev Container ビルドエラーの修正
- 原因: `.gitconfig` を `/home/node/.gitconfig` に直接バインドマウントしていたため、`postCreateCommand` の `git config --global` が atomic write（削除→再作成）でマウントポイントと衝突し `Device or resource busy` エラーが発生
- 対応:
  - マウント先を `/home/node/.gitconfig.host`（読み取り専用）に変更
  - `postCreateCommand` 先頭で `.gitconfig.host` → `.gitconfig` にコピーしてから `git config` を実行

## 17:20 セッション
- 作業: ファイアウォール（init-firewall.sh）にローカル専用の拡張ポイントを追加
- 背景: git 管理外のローカル設定ファイルから追加ドメインを読み込めるようにし、公開リポジトリを汚さずにファイアウォールを拡張可能にする
- 実施内容:
  - `init-firewall.sh` にローカルドメインファイル読み込み処理を追加（ファイルがなければ既存動作に影響なし）
  - `.devcontainer/firewall-local-domains.txt` を新規作成（ローカル用途のドメインを記載）
  - `.git/info/exclude` に `firewall-local-domains.txt` を追加し git 管理外に設定

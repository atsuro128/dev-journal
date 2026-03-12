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

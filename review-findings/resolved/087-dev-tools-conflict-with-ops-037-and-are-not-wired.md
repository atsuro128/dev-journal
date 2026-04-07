# 087: （タイトル：開発者ツール実装が ops-037 の責務分担と矛盾し有効化経路もない）

## 指摘概要
8-10 の成果物は `ops-037` の判断を反映する必要があるが、現状は Claude Code hook に format / lint を実装しており、上流で確定した「format は git pre-commit、lint は CI、Claude hooks は AI 固有安全策のみ」という責務分担に反している。加えて、リポジトリ内には hook スクリプトしかなく、プロジェクト設定としてこのスクリプトを登録するファイルも存在しないため、完了条件の「コード変更時に自動 format と軽量 lint が実行される」を検証可能な状態になっていない。

## 根拠
- `dev-journal/progress-management/tickets/step8/8-10-dev-tools.md:15-19`
  - 8-10 は ops-037 の判断を反映することを要求している。
- `dev-journal/issues/resolved/ops-037-claude-code-hooks-operation-design.md:39-45`
  - フォーマットは `git pre-commit hook`、lint は `CI`、Claude Code hooks は AI 固有安全策に限定すると解決済み。
- `expense-saas/.claude/hooks/post-edit-format-lint.sh:1-64`
  - Claude Code hook で `gofmt` と `tsc --noEmit` を直接実行し、lint 失敗時は `exit 2` でブロックする実装になっている。
- `expense-saas/.claude/` 配下には `hooks/post-edit-format-lint.sh` 以外の設定ファイルがなく、hook 登録経路がリポジトリ内に存在しない。

## 判定
高 / FIX

## 修正方針案
- ops-037 に合わせて、format は git hook、lint は CI に寄せ、Claude Code hooks には AI 固有の安全策だけを残す。
- もしプロジェクト方針を変更して Claude hook を使うなら、ops-037 とチケット定義を更新した上で、実際に有効化される設定ファイルまでコミットする。

## 解決内容
- ops-037 に準拠し、format を git pre-commit hook（`.githooks/pre-commit`）に移行
- Claude Code hook の format/lint 実装を削除、settings.json の PostToolUse を除去
- Makefile に `setup` ターゲット追加（`git config core.hooksPath .githooks`）
- 8-10 チケット記述を ops-037 準拠に更新

## 解決日
2026-03-31

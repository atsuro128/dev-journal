# 開発者ツール

- 担当: platform-builder
- 依存: 8-2, 8-3
- ブランチ: step8/8-10-dev-tools
- 出力先: expense-saas/.githooks/ (git pre-commit hook設定)
- テンプレート: なし

## 入力

なし

## 責務

- コミット時の自動 format（Go: gofmt, TS: prettier）を git pre-commit hook で実行
- lint は CI（GitHub Actions）で実行するため、ここでは対象外
- Claude Code hooks は AI 固有安全策（edit-scope-check.py）のみとし、format/lint は含めない
- 含めない: CI パイプライン（8-9）、lint（CI 担当）
- ops-037（hooks 運用 issue）の判断を反映済み

## 完了条件

- コミット時に自動 format が実行される（lint は CI で実行済み）

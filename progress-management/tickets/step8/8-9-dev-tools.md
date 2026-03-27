# 開発者ツール

- 担当: platform-builder
- 依存: 8-2, 8-3
- 出力先: expense-saas/.claude/hooks/ (Claude Code hooks設定)
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| （プロジェクト方針に従う） | - | - |

## 責務

- コード変更後の自動 format（Go: gofmt/goimports, TS: prettier）
- 変更前の軽量 lint（Go: go vet, TS: eslint）
- 判断ポイント: 対象ツール、失敗時挙動（警告 vs ブロック）、CI との重複許容範囲
- 含めない: CI パイプライン（8-8）
- 参考: ops-037（hooks 運用 issue）の判断を反映する

## 完了条件

- コード変更時に自動 format と軽量 lint が実行される

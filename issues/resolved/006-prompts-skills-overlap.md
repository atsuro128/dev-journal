# prompts/ と skills/ の役割重複の懸念

## 発見日
2026-03-04

## 関連ステップ
Step 1 着手前に方針を決めるのが望ましい

## カテゴリ
project-management

## 問題
`prompts/` と `skills/` の境界が曖昧。両方とも「繰り返し使う指示文」を扱う可能性がある。

## 影響
ファイルの配置先が不明確になり、重複や不整合が発生する。

## 提案
- `skills/` → Claude Code の Skill 機能（`/review`, `/status` 等）として実際に呼び出すもの
- `prompts/` → 手動でコピー&ペーストする指示文テンプレート、または skills に昇格する前の下書き

## 解決内容
- `skills/` を廃止し、カテゴリ構想を `prompts/README.md` に統合
- `prompts/` の役割を「手動コピペ用テンプレート / `.claude/commands/` 昇格前の下書き」と明確化
- `.claude/commands/` が Claude Code 公式のスラッシュコマンド機能であることをドキュメントに反映
- `CLAUDE.md`、`references/directory-structure.md` を更新

## 解決日
2026-03-04


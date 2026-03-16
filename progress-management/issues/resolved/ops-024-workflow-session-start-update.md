# workflow.md のセッション開始手順を work-breakdown ベースに更新

## 発見日
2026-03-15

## カテゴリ
ai-ops

## 影響度
高

## 発見経緯
proactive

## 関連ステップ
全 Step（横断）

## 問題

`.claude/rules/workflow.md` のセッション管理には「作業開始時に progress.md を確認」とだけ書かれている。work-breakdown 導入後も、work-breakdown ファイルへの導線がなく、各セッションがどのタスク ID を担当するかを知る仕組みがない。

## 影響

- work-breakdown を作っても、セッション開始時に参照されない
- 各セッションが何を担当するか不明確なまま作業開始してしまう

## 提案

workflow.md のセッション管理セクションを更新:
- 作業開始時: progress.md で現在 Step 確認 → 該当 Step の work-breakdown で担当タスク確認
- 作業完了時: work-breakdown のタスク状態更新 → progress.md の Step 状態更新

---

## 解決内容

指摘は「workflow.md に work-breakdown への導線がない」というものだが、実際には project_steps.md の冒頭に「各ステップの詳細は `work-breakdown/` 配下のファイルを参照してください」と明記されており、テーブルにも各 Step の work-breakdown へのリンクがある。workflow.md → progress.md → project_steps.md → work-breakdown という導線は既に存在しており、workflow.md に重複記載する必要はない。指摘内容が的外れのためクローズ。

## 解決日
2026-03-16

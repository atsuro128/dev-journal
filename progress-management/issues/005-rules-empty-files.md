# rules/ の空ファイルが複数ある

## 発見日
2026-03-04

## 関連ステップ
各ステップで段階的に対処

## カテゴリ
project-management

## 問題
以下のファイルはヘッダーのみで本文なし:
- `rules/branching.md`
- `rules/review-checklist.md`
- `rules/security-policy.md`
- `rules/data-handling.md`

## 影響
肉付けタイミングが管理されていないと、必要な時に規約が未整備のまま作業が進むリスク。

## 提案
project_steps.md の【root-project整備】に従い、対応するステップで肉付けする:
- `branching.md` → Step 7 Phase 1 着手前
- `review-checklist.md` → Step 6
- `security-policy.md` → Step 1
- `data-handling.md` → Step 5

## 解決内容


## 解決日


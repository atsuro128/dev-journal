# （タイトル：Step 8 work-breakdown のディレクトリパス未更新）

## 発見日
2026-03-24

## カテゴリ
design

## 影響度
低

## 発見経緯
Step 7 task-plan の計画レビューで指摘（指摘 #4）

## 関連ステップ
Step 8（テストコード実装）

## 問題

Step 7 の判断ポイント #1 でディレクトリ構成を Go 標準（`cmd/server/` + `internal/` + `frontend/`）に変更したが、Step 8 の work-breakdown（`dev-journal/guide/work-breakdown/step8-test-implementation.md`）の上流成果物参照パスがまだ `expense-saas/apps/` を参照している可能性がある。

## 影響

- Step 8 着手時に参照パスの不一致で混乱が生じる

## 提案

- Step 8 着手前に、Step 8 work-breakdown の参照パスを `expense-saas/cmd/`, `expense-saas/internal/`, `expense-saas/frontend/` に修正する
- 同様に Step 9, Step 10 の work-breakdown も確認し、必要に応じて修正する

---

## 解決内容

Step 8 は前回セッション（3/28）のタスク分解時に既に正しいパスで記述されていた。
Step 9, 10, 11 に旧パス `apps/` が残っていたため修正:

- step9-test-implementation.md: 4箇所修正（`apps/api/` → `internal/`, `apps/web/` → `frontend/`, `apps/` → ルート）
- step10-feature-implementation.md: 15箇所修正（同上パターン、全機能タスクの出力パス含む）
- step11-system-test.md: 1箇所修正

修正後、全 work-breakdown で `apps/` の参照ゼロを確認済み。

## 解決日
2026-03-28

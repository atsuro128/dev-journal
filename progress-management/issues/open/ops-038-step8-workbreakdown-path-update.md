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
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

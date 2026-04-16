# UI カバレッジ監査リサーチ（ロール別表示制御 + レスポンシブ）

## 発見日
2026-04-16（親 issue 104 から分割）

## カテゴリ
testing

## 影響度
中

## 発見経緯
escalation（issue 104 の決定 3 に基づく sub-issue 分割）

## 関連ステップ
Step 5（詳細設計）/ Step 6（テスト設計）/ Step 11-A

## ブロッカー
なし

## 親 issue
104（UI RBAC × レスポンシブ監査マトリクス）

## 問題

issue 104 で特定された「UI 層のロール別表示制御 + レスポンシブ挙動のカバレッジ不足」について、現状のギャップを定量的に把握するリサーチが未実施。テスト追加・smoke 拡充（104-B）の前提となる棚卸しが必要。

## 影響

ギャップの全体像が不明なまま 104-B に着手すると、対応漏れや優先順位の誤判断が発生する。

## 提案

### 入力資料
- 画面仕様: `dev-journal/deliverables/docs/50_detail_design/screens/*.md`（全 13 画面）
- UI ガイドライン: `dev-journal/deliverables/docs/50_detail_design/ui-guidelines.md`（ブレークポイント定義）
- 認可設計: `dev-journal/deliverables/docs/50_detail_design/authz.md`
- 既存テストコード: `expense-saas/frontend/src/pages/**/__tests__/*.test.tsx`
- smoke チェックリスト: `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`

### 担当
designer

### 作業内容

1. **棚卸し**: 全 13 画面の screens/*.md から、UI コントロール（ボタン・リンク・メニュー・操作要素）× 4 ロール（Member / Approver / Accounting / Admin）× 3 viewport（PC 1280px / タブレット 900px / スマホ 375px）の表示仕様を抽出
2. **既存テスト確認**: 各項目について、既存の component test / smoke_check.md でカバーされているかをマーキング
3. **ギャップ分析**: 未カバー項目をリストアップし、issue 104 の決定 2（重要度振り分け）に従って分類:
   - クリティカル（副作用あり）→ 自動テスト + smoke 両方が必要
   - 情報表示系 → smoke のみ必要
   - その他 → 自動テストのみ必要

### 成果物
`dev-journal/deliverables/docs/60_test/ui_coverage_matrix.md`

### 成果物の構成（想定）
1. 画面 × ロール × コントロールのマトリクス表（表示/非表示/条件付き）
2. 画面 × viewport のレスポンシブ挙動表（Drawer 切替、テーブル→カード化、etc.）
3. 既存カバレッジマーキング（component test / smoke / なし）
4. ギャップリスト（未カバー項目 × 重要度分類）

## 完了条件

- 全 13 画面について、ロール × コントロールマトリクスが抽出されている
- 全 13 画面について、viewport 別の表示仕様が抽出されている
- 各項目の既存テストカバー有無がマーキングされている
- ギャップリストが重要度分類（クリティカル / 情報表示系 / その他）付きで整理されている
- 成果物が `ui_coverage_matrix.md` としてコミット可能な状態にある

## 関連
- 104: 親 issue（方針合意）
- 104-B: 実装フェーズ（本 issue の成果物が入力）

---

## 解決内容

## 解決日

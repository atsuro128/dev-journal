# UI カバレッジ監査実装（テスト追加 + smoke 拡充 + バグ修正）

## 発見日
2026-04-16（親 issue 104 から分割）

## カテゴリ
testing

## 影響度
中

## 発見経緯
escalation（issue 104 の決定 3 に基づく sub-issue 分割）

## 関連ステップ
Step 9（テストコード実装）/ Step 11-A / Step 11-B

## ブロッカー
104-A 完了必須（リサーチ成果物 `ui_coverage_matrix.md` が入力）

## 親 issue
104（UI RBAC × レスポンシブ監査マトリクス）

## 問題

104-A のリサーチで特定されたギャップ（UI ロール別表示制御 + レスポンシブ挙動の未検証項目）に対して、テスト追加・smoke 拡充・実装バグ修正が必要。

## 影響

未検証のまま放置すると、ロール変更やレスポンシブ対応のリグレッションを検出できない。

## 提案

### 入力資料
- 104-A 成果物: `dev-journal/deliverables/docs/60_test/ui_coverage_matrix.md`
- issue 104 決定 2（重要度振り分け基準）

### 担当
test-implementer + frontend-developer

### 作業内容

1. **クリティカル項目（副作用あり）**: component test + smoke_check.md の両方に追加
   - 例: 添付削除ボタン、申請承認/却下ボタン、支払確定ボタン、テナント設定変更
   - 例: md ブレークポイント切替（Drawer permanent ↔ temporary）
2. **情報表示系**: smoke_check.md に SMK 項目として追加
   - 例: グローバルナビメニュー、画面内リンク、フィルタドロップダウン
3. **その他**: component test のみ追加
   - 例: タップ領域サイズ、スクロール挙動
4. **実装バグ修正**: 検証過程で発見された設計書との乖離を個別 issue として起票・修正

### 成果物
- テストコード追加: `expense-saas/frontend/src/pages/**/__tests__/*.test.tsx`
- smoke 拡充: `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`
- 個別バグ issue: 発見次第起票

## 完了条件

- 104-A のギャップリスト内クリティカル項目が自動テスト + smoke の両方で検証されている
- 情報表示系項目が smoke_check.md に追加されている
- その他項目が自動テストに追加されている
- 監査過程で発見された実装バグが個別 issue として起票されている
- 追加テストが CI で通過している

## 関連
- 104: 親 issue（方針合意）
- 104-A: リサーチフェーズ（本 issue の入力）

---

## 解決内容

## 解決日

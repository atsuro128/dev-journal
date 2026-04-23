# 対象期間バリデーション V5 を両フィールド発火 + フィールド別文言に改善（設計書内文言矛盾も解消）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design / ux

## 影響度
中（UX polish。機能的には保存押下時に検出されるのでバグではないが、早期フィードバックの欠如で UX 品質低下）

## 発見経緯
Step 11-A SMK-051（エラーメッセージの自然さ）手順 3（日付論理バリデーション V5 の文言自然さ観察）にて、「終了日 → 開始日」の順で入力した場合にエラーが保存ボタン押下まで表示されないこと、および終了日フィールドに主語が別フィールドの文言が表示される UX 違和感を発見。またユーザー指摘（「終了日→開始日の順で入力した場合エラーが表示されず保存まで分からない」）と照合する中で、設計書内に文言の多数派・少数派が混在していることが判明した。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10-B（レポート UI）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（バリデーションは保存押下時に発動するため機能動作に問題なし）

## 問題

### 1. 発火条件の不整合

- 設計書（report-create.md §V5 / report-edit.md §V5 / admin-all-reports.md §93, §137 / smoke_check.md SMK-022）いずれも「終了日のフォーカスアウト / 送信時」のみ規定
- 結果、**終了日 → 開始日 の順で入力**した場合、開始日を編集してもエラーが表示されない
- 現 UX: 保存ボタン押下時に初めて気付く（ユーザー指摘通り）

### 2. 文言の UX 違和感

- 終了日フィールド直下に「**開始日は終了日以前を指定してください**」が表示される
- 主語が他フィールドを指しており、どこを修正すべきか直感的に分かりづらい

### 3. 設計書内の文言矛盾

| 文言 | 採用ファイル数 | 該当ファイル |
|------|------------|------------|
| 「開始日は終了日以前を指定してください」 | 6 | `report-create.md` / `report-edit.md` / `admin-all-reports.md` / `state-management.md` / `usecases.md` / `smoke_check.md` |
| 「終了日は開始日以降にしてください」 | 1 | `test_cases/reports.md:444`（RPT-FE-041） |

- test_cases 側の文言はユーザー指摘（UX 的に自然）と一致しているが、実装は多数派側に従っている
- 正本が未確定のまま設計書と実装が混在状態

## 期待する挙動

- **開始日 blur でも V5 発火** → 開始日フィールド直下に「**開始日は終了日以前を指定してください**」
- **終了日 blur でも V5 発火** → 終了日フィールド直下に「**終了日は開始日以降を指定してください**」
- どちらのフィールドを編集しても、条件違反があれば即時エラーを表示する

## 対応内容

### 実装

- `expense-saas/frontend/src/pages/reports/ReportForm.tsx`（または ReportPeriodField 相当のコンポーネント）
  - Zod の `refine` を 2 つに分割し、`path: ['periodStart']` と `path: ['periodEnd']` で個別エラー出力
  - 開始日・終了日どちらの `onBlur` でも両方のバリデーションを再評価
  - react-hook-form の `trigger(['periodStart', 'periodEnd'])` を両フィールドの `onBlur` で呼び出す

### 設計書改訂（7 ファイル）

- `50_detail_design/screens/report-create.md` §V5: トリガーを「両フィールドのフォーカスアウト / 送信時」に変更。フィールド別文言を明記
- `50_detail_design/screens/report-edit.md` §V5: 同上
- `50_detail_design/screens/admin-all-reports.md` §93, §137: フィルタバリデーションもフィールド別に
- `55_ui_component/state-management.md` L486, L530（INVALID_PERIOD）: 2 メッセージ体制に変更（または 1 メッセージ + 表示箇所別定義）
- `10_requirements/usecases.md` L54: 文言併記（または削除して「フィールドごとに適切な文言」と抽象化）
- `60_test/manual_checklists/smoke_check.md` SMK-022: 期待結果を「開始日・終了日どちらの blur でも該当フィールド下にエラーが表示される」に変更

### テスト

- `60_test/test_cases/reports.md`
  - 既存 RPT-FE-041（終了日側）: 文言を「終了日は開始日以降を指定してください」に統一（既に近い形）
  - 新規追加 **RPT-FE-107**: 開始日側のエラー表示ケース（開始日 blur で V5 発火 → 開始日フィールド直下にエラー表示）
- Vitest / E2E の該当テストを新仕様に合わせて更新
- PR チェックリストに「テスト設計書の更新」項目を含めること

## 修正対象ファイル

### 実装

| ファイル | 変更内容 |
|--------|---------|
| `expense-saas/frontend/src/pages/reports/ReportForm.tsx` | Zod refine を 2 分割（periodStart / periodEnd 個別パス）、両フィールド onBlur で trigger(['periodStart', 'periodEnd']) |

### 設計書

| ファイル | 変更箇所 |
|--------|---------|
| `dev-journal/deliverables/docs/50_detail_design/screens/report-create.md` | §V5 トリガー・文言をフィールド別に改訂 |
| `dev-journal/deliverables/docs/50_detail_design/screens/report-edit.md` | §V5 トリガー・文言をフィールド別に改訂 |
| `dev-journal/deliverables/docs/50_detail_design/screens/admin-all-reports.md` | §93, §137 フィルタバリデーション文言改訂 |
| `dev-journal/deliverables/docs/55_ui_component/state-management.md` | L486, L530 INVALID_PERIOD を 2 メッセージ体制に変更 |
| `dev-journal/deliverables/docs/10_requirements/usecases.md` | L54 文言を抽象化または併記 |
| `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` | SMK-022 期待結果を両フィールド発火に変更 |

### テスト

| ファイル | 変更内容 |
|--------|---------|
| `dev-journal/deliverables/docs/60_test/test_cases/reports.md` | RPT-FE-041 文言統一、RPT-FE-107 新規追加（開始日 blur V5 発火ケース） |
| `expense-saas/frontend/src/...` | Vitest の ReportPeriodField.test.tsx 等を新仕様に更新 |

## 影響範囲

- FE のみ（BE 側バリデーションは変更不要、API 契約変更なし）
- UX 早期フィードバック改善（機能影響なし）
- 他フォームへの波及なし

## 関連 SMK / issue

- **SMK-022**（リアルタイムバリデーション・日付論理）: FAIL → issue 119（日付 onBlur 不発火）+ issue 120（空値 Zod デフォルトエラー）。本 issue はその後継となる「発火フィールドの拡張 + 文言フィールド別化」
- **SMK-051**（エラーメッセージの自然さ）: 本 issue 起票の直接起点
- **#119 / #120**: ReportForm バリデーション修正済み（PR #75）。本 issue はその延長線上の UX polish

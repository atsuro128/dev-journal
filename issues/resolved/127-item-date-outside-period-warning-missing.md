# 明細日付が対象期間外でも警告が表示されない（ITM-007 実装漏れ）

## 発見日
2026-04-20

## カテゴリ
implementation / frontend / ui / policy-compliance

## 影響度
中（設計ポリシー ITM-007 の違反。対象期間フィールドの実効性が失われ、入力ミス検出機能として機能していない）

## 発見経緯

Step 11-A ローカル動作確認のセッション中、ユーザーが「レポートの期間が 4/1-4/30 に対して、そこに含める明細の日付を 3/1 に設定できる」と指摘。確認したところ、期間外の明細日付で保存しても **警告が一切表示されない**。

設計上は警告表示が必須（ITM-007）。実装の確認でも `ItemForm.tsx` に期間関連ロジックが存在しないことを確認。

## 関連ステップ
Step 1（要件定義: policies.md ITM-007）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計の要求

`dev-journal/deliverables/docs/10_requirements/policies.md`:

- **ITM-007**: 明細の日付が対象期間外の場合、**警告を表示するが保存は許可する**
- 意思決定テーブル: 「明細の日付が対象期間外 | 警告のみ、エラーにしない | ユーザビリティ優先」

→ 保存はブロックしないが、警告表示は必須。

### 実装の状態

- `expense-saas/frontend/src/pages/reports/ItemForm.tsx` に `period` / `warning` / 期間関連ロジックが一切存在しない（grep で 0 件）
- 日付入力で期間外の値を入力・保存しても、フォーム内警告・トースト・ダイアログ等のいずれも表示されない
- バックエンドでも ITM-007 の警告に関するレスポンスは想定されていない（警告はフロントエンド責務）

### 影響

1. **ポリシー違反**: ITM-007 が実装されていない
2. **入力ミス検出機能の欠如**: 期間とのずれ（月ズレ等のタイプミス）をユーザーが気付くきっかけがない
3. **対象期間フィールドの実効性低下**: 期間外データを許容しながら警告もない = 期間フィールドがラベル以上の意味を持たない

## 修正方針

### 実装: ItemForm に期間外警告を追加

`ItemForm` が所属するレポートの `period_start` / `period_end` を受け取り、`expenseDate` との比較を行う:

```tsx
// 修正案
const showPeriodWarning = useMemo(() => {
  if (!expenseDate || !reportPeriodStart || !reportPeriodEnd) return false;
  return expenseDate < reportPeriodStart || expenseDate > reportPeriodEnd;
}, [expenseDate, reportPeriodStart, reportPeriodEnd]);

// フォーム内に <Alert severity="warning"> で表示
{showPeriodWarning && (
  <Alert severity="warning">
    明細日付がレポートの対象期間（{reportPeriodStart} 〜 {reportPeriodEnd}）外です。保存は可能ですが、入力を確認してください。
  </Alert>
)}
```

### 警告表示の UI パターン

- **選択肢A（推奨）**: フォーム内に `<Alert severity="warning">` で恒常的に表示（日付フィールドの直下）
- 選択肢B: トースト（Snackbar severity: 'warning'）で表示
- 選択肢C: 保存ボタン押下時に確認ダイアログ表示

案 A が一般的（即時フィードバック、常時可視、保存を阻害しない）。

### 文言

設計書に文言例が定義されていないため、本 issue 対応時に state-management.md § 成功・警告メッセージ一覧 に追記する必要あり。候補:

- 「明細日付がレポートの対象期間外です。入力を確認してください。」
- 「明細日付（{date}）はレポートの対象期間（{start}〜{end}）外です。」

## 修正対象ファイル

### 実装
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`
  - レポートの `period_start` / `period_end` を props or context から受け取る
  - 期間外判定ロジック追加
  - `<Alert>` 表示追加
- ItemForm を呼び出す親コンポーネント（`ReportDetailPage` / `ItemCreate*` 系）
  - レポートの期間情報を ItemForm に渡す

### 設計書
- `dev-journal/deliverables/docs/55_ui_component/state-management.md` § 警告メッセージ（ITM-007 対応文言を追加）
- 該当画面設計書（`50_detail_design/screens/report-detail.md` or `item-*.md`）に警告表示仕様を追記

### テスト
- `ItemForm.test.tsx` に「期間外日付で警告が表示される」「期間内日付で警告が出ない」テストを追加

## 対応条件

- 明細日付がレポートの対象期間外の場合、警告が表示される
- 保存自体は許可される（ITM-007: エラーではなく警告）
- 警告文言が設計書に定義されており実装と一致
- テストで期間外・期間内のケースが検証される

## 関連

- `policies.md` ITM-007: 設計ポリシー
- ユーザーから「レポート期間の意味は何か」という疑問があった（2026-04-20 セッション）。ITM-007 が未実装なことで期間フィールドの意義が希薄化している背景と一致

## 補足: 設計の妥当性議論（参考情報、本 issue のスコープ外）

ITM-007 の「警告のみ、エラーにしない」という設計判断自体、以下の観点で再検討余地がある:

- 警告許容により期間フィールドの制約としての実効性が弱い
- 期間フィールドを撤廃 or オプショナル化する案もある
- ただしこれは MVP のリリース判断に関わる設計変更であり、本 issue とは別途議論が必要

本 issue は「現行設計（ITM-007）に従った警告の実装漏れ」のみをスコープとする。設計自体の見直しは別途。

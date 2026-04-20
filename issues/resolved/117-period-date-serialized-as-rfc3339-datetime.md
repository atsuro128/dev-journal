# period_start / period_end が RFC3339 date-time で返され、フロントの日付入力にプリフィルされない

## 発見日
2026-04-19

## カテゴリ
implementation / backend / api-contract / frontend-integration

## 影響度
中〜高（レポート編集時に期間が再入力される UX 破綻 + API 契約違反）

## 発見経緯
Step 11-A ローカル動作確認の延長で、下書きレポートの編集画面を開いた際、「開始日」「終了日」フィールドの初期値が空（`yyyy/mm/dd` プレースホルダー）で表示された。既存レポートを編集する場面なので、保存済みの日付がプリフィルされるべき。

## 関連ステップ
Step 5（API 詳細設計 openapi.yaml）/ Step 10-B（レポート機能実装）/ Step 11-A

## ブロッカー
修正前は MVP UX として不適切（編集のたびに日付再入力強制 + 上書き保存すると毎回「同じ日付を書き直す作業」になる）

## 問題

### 設計仕様（API 契約）

`openapi.yaml` は `period_start` / `period_end` を `type: string, format: date` と定義:

```yaml
period_start:
  type: string
  format: date
period_end:
  type: string
  format: date
```

→ RFC3339 full-date（`YYYY-MM-DD`）が仕様。

### 実装の挙動

Go のサービス DTO では `time.Time` のまま定義されており、`encoding/json` のデフォルト marshaling で **RFC3339 date-time**（`YYYY-MM-DDTHH:MM:SSZ`）に変換される。

```go
// expense-saas/internal/service/dto.go:97,112,157
type ExpenseReportSummary struct {
    ...
    PeriodStart time.Time `json:"period_start"`  // ← 実際のレスポンス: "2026-04-01T00:00:00Z"
    PeriodEnd   time.Time `json:"period_end"`
}

type ExpenseReportDetail struct {
    ...
    PeriodStart time.Time `json:"period_start"`
    PeriodEnd   time.Time `json:"period_end"`
}

type RecentReport struct {
    ...
    PeriodStart time.Time `json:"period_start"`
    PeriodEnd   time.Time `json:"period_end"`
}
```

### フロントへの影響

#### 1. レポート編集画面（症状確認済み）

- `ReportEditPage.tsx:148-149` で `defaultValues={{ periodStart: report.period_start, periodEnd: report.period_end }}` を `ReportForm` に渡す
- `AppDatePicker` は `<input type="date">` で value を受け取るが、ブラウザの native date input は `YYYY-MM-DD` 以外を解釈せず、**空として扱う**
- 結果: 編集画面で開始日・終了日が `yyyy/mm/dd` プレースホルダー表示のまま

#### 2. レポート一覧画面（要確認だが同根の可能性）

- `ReportListPage.tsx:238`: `{report.period_start} 〜 {report.period_end}` を生値でレンダリング
- API が RFC3339 date-time を返す場合、`"2026-04-01T00:00:00Z 〜 2026-04-30T00:00:00Z"` と表示される恐れ

#### 3. その他の表示箇所（formatter で救われている可能性）

- `ReportInfoCard.tsx:63`: `formatDateSlash(report.period_start)` を使っているため、formatter の仕様次第で RFC3339 でも正しく表示される可能性あり
- `RecentReportRow.tsx:60`: `formatDate(periodStart)` 同上

formatter が RFC3339 入力を吸収しているかは要確認。

## 根本原因

Go の `time.Time` を無加工で JSON 出力するため、OpenAPI 契約（`format: date`）と実際のレスポンス形式（RFC3339 date-time）が乖離している。ExpenseItem の `ExpenseDate` は `handler/item.go:44` で `dto.ExpenseDate.Format("2006-01-02")` と手動変換されており、レポート側だけこの変換が漏れた状態。

## 修正方針

### 案A（推奨）: バックエンドで YYYY-MM-DD 文字列として返す

OpenAPI 契約に準拠する形で DTO を修正。

- **方針1**: DTO のフィールド型を `string` に変更し、サービス/ハンドラー境界で `t.Format("2006-01-02")` 変換
- **方針2**: カスタム型 `Date time.Time` を定義し、`MarshalJSON` で YYYY-MM-DD 出力（他機能も含めて横展開しやすい）

既存の ExpenseItem.ExpenseDate と揃えるなら方針1でも十分。

### 案B: フロント側で変換

`report.period_start.slice(0, 10)` 等で日付部分だけ切り出して渡す。ただし OpenAPI 契約違反は残るため推奨しない。

## 修正対象ファイル

### 案A（推奨）を採用する場合

- `expense-saas/internal/service/dto.go`:
  - `ExpenseReportSummary.PeriodStart/End`
  - `ExpenseReportDetail.PeriodStart/End`
  - `RecentReport.PeriodStart/End`
- サービス層で time.Time → string 変換する場合、以下も合わせて確認:
  - `expense-saas/internal/service/report_service.go` (296, 414 付近の DTO 生成箇所)
  - `expense-saas/internal/service/dashboard_service.go` (105)
  - `expense-saas/internal/service/workflow_service.go` (244)
- テスト: 日付比較部分の更新
  - `expense-saas/internal/handler/report_handler_test.go` ほか

### フロント側の影響確認

- `expense-saas/frontend/src/api/types.ts` の該当箇所はすでに `string` 型なので型変更不要
- ただし `formatDate` / `formatDateSlash` が RFC3339 前提で書かれている場合は、YYYY-MM-DD 入力でも動作するか確認（通常 `new Date(str)` は両形式を受け付けるので大丈夫だが要確認）

## 完了条件

- API レスポンス（GET /reports, GET /reports/{id}, GET /dashboard 等）の `period_start` / `period_end` が `YYYY-MM-DD` 形式で返る
- レポート編集画面で保存済み日付が開始日/終了日にプリフィルされる
- レポート一覧画面の期間表示が崩れていない
- 既存のテスト（handler_test, service_test）が新フォーマットに追従して通過する
- OpenAPI 契約（`format: date`）と実装が一致する

## 次セッション着手手順

1. ブランチ作成: `fix/117-period-date-format`
2. DTO 型修正（方針1 or 2 をユーザーと合意してから）
3. サービス層で日付変換を実装
4. バックエンドテストの期待値を YYYY-MM-DD に更新
5. ローカル起動でレポート編集画面・一覧画面を視覚確認
6. PR 作成（影響範囲: API レスポンス形式の変更。破壊的変更だがフロント側の `types.ts` は `string` なのでコンパイル影響なし）

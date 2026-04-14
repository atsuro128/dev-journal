# Admin ダッシュボード月別合計が 2026-03 のみ・0 円表示（seed データ不足が原因）

## 発見日
2026-04-13

## カテゴリ
implementation / backend / seed-data

## 影響度
中

## 発見経緯
Step 11-A ローカル動作確認（SMK-004）で、Admin ロールのダッシュボードを開いた際、月別合計テーブルが 1 ヶ月（2026-03）のみ・合計金額 0 円で表示されていることを観察。2026-04-14 に psql で切り分けを実施し、集計クエリではなく seed データの不足が原因と確定

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（ポートフォリオデモ時の表示品質に影響）

## 概要

Admin ダッシュボードの月別合計テーブルは「直近 3 ヶ月の月別合計金額」を表示する想定だが、実機では 2026-03 のみ・合計金額 0 円が表示される。

切り分けの結果、**集計クエリは正しく動いている**が、**seed データが不足している**ため 1 行・0 円の表示となる。

## 切り分け結果（2026-04-14 実施）

### クエリ 1: ステータス別の分布

```sql
SELECT status, COUNT(*), SUM(total_amount) FROM expense_reports GROUP BY status ORDER BY status;
```

| status | count | sum (total_amount) |
|--------|-------|--------------------|
| approved | 2 | **0** |
| draft | 3 | 2000 |
| paid | **1** | **0** |
| rejected | 1 | **0** |
| submitted | 2 | 2000 |

### クエリ 2: paid レポートの period_start 分布

```sql
SELECT DATE_TRUNC('month', period_start), status, COUNT(*), SUM(total_amount)
FROM expense_reports WHERE status = 'paid' GROUP BY 1, 2 ORDER BY 1;
```

| month | status | count | sum |
|-------|--------|-------|-----|
| 2026-03-01 | paid | 1 | 0 |

### クエリ 3: paid レポートの paid_at 分布

```sql
SELECT DATE_TRUNC('month', paid_at), COUNT(*), SUM(total_amount)
FROM expense_reports WHERE status = 'paid' GROUP BY 1 ORDER BY 1;
```

| paid_month | count | sum |
|-----------|-------|-----|
| **(NULL)** | 1 | 0 |

## 判明した seed データの問題（3 つの欠陥）

### 問題 ① paid レポートが 1 件しか存在しない

- 期待: 直近 3 ヶ月（例: 2026-02, 2026-03, 2026-04）に分散して複数件
- 実際: 2026-03 に 1 件のみ
- 影響: ダッシュボードの月別合計テーブルが 1 行しか表示されない

### 問題 ② paid レポートの total_amount が 0 円

- 期待: 紐づく expense_items の合計金額が `total_amount` に反映されている
- 実際: paid / approved / rejected 全てで `total_amount = 0`
- 推定原因: seed スクリプトが draft / submitted にしか expense_items を紐付けていない、または paid / approved 遷移時に `total_amount` を再計算していない
- 影響: ダッシュボードの金額表示が 0 円

### 問題 ③ paid レポートの paid_at が NULL

- 期待: `status = 'paid'` なら `paid_at` に支払完了タイムスタンプが設定されている
- 実際: `paid_at = NULL`
- 推定原因: seed スクリプトが paid ステータスに遷移させているが、タイムスタンプを埋めていない
- 影響: セマンティクス違反。paid_at で集計するクエリでは空行として集計される

## 修正方針

### seed スクリプトの拡充

**対象**: `expense-saas/internal/seed/seed.go`

1. **paid レポートを直近 3 ヶ月に分散生成**:
   - 2026-02 / 2026-03 / 2026-04 に最低 1 件ずつ、できれば各月 2〜3 件
   - 既存の標準フィクスチャ（固定 UUID）を維持しつつ、追加レポートを生成
2. **expense_items を全ステータスの代表レポートに紐付け**:
   - 現状 draft（1 件で 2000 円）/ submitted（2 件で 2000 円）のみ items が紐付いている
   - approved / rejected / paid のフィクスチャレポートにも items を紐付けて `total_amount` を 0 円以外に
3. **paid_at を適切なタイムスタンプで設定**:
   - paid ステータスなら `paid_at = <支払完了日時>` を必須で設定
   - approved / submitted / rejected についても対応するタイムスタンプ（approved_at / submitted_at / rejected_at）を設定

### 実装メモ

- `total_amount` は items の合計から自動計算される設計（`db_schema.md` 参照）なので、items 挿入後に `UPDATE expense_reports SET total_amount = (SELECT SUM(amount) FROM expense_items WHERE report_id = ...)` で更新するか、リポジトリ層の再計算ロジックを使う
- 既存の固定 UUID（`ReportDraftID` / `ReportSubmittedID` 等）は維持し、追加する paid レポートには新しい UUID を割り当てる
- Admin ダッシュボードが集計する対象（テナント A の全レポート）に追加する

## 修正対象ファイル

- `expense-saas/internal/seed/seed.go` — paid レポート追加生成、items 紐付け、タイムスタンプ設定
- `expense-saas/internal/seed/seed_test.go`（もしあれば）
- 関連する固定 UUID 定義箇所（必要に応じて）

## 完了条件

- seed 実行後、`expense_reports` テーブルに paid ステータスのレポートが直近 3 ヶ月に分散して最低 3 件以上存在する
- 全ての paid レポートが `total_amount > 0` かつ `paid_at IS NOT NULL` である
- Admin ダッシュボードに直近 3 ヶ月の月別合計金額が意味ある数値（0 円以外）で表示される
- 既存の SMK-037 / SMK-038 等の前提データ（`AttachmentSubmittedID` / `AttachmentDraftID` 等）が壊れていない
- seed は冪等性を維持（複数回実行してもデータが重複しない）

## 関連

- SMK-004（Admin 入口画面確認）は観察的に PASS 判定されているが、本 issue 修正後に観察内容が正常化することで正式 PASS となる

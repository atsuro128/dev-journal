# 一部ページで素の HTML 要素（button / table）が残存

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 10（機能実装）

## ブロッカー
なし

## 問題

以下のファイルで素の HTML `<button>` / `<table>` が使用されており、MUI コンポーネントに統一されていない。

### button（10ファイル）
- `ApprovalListPage.tsx` — ページネーションボタン
- `PaymentListPage.tsx` — ページネーションボタン
- `AttachmentList.tsx` — ダウンロード・削除ボタン
- `ItemListHeader.tsx` — 明細追加ボタン
- `ItemSlidePanel.tsx` — 閉じるボタン
- `ReportActionBar.tsx` — 承認・却下・支払完了ボタン
- `ReportListHeader.tsx` — レポート作成ボタン
- `ReportListTable.tsx` — 空状態 CTA ボタン
- `WorkflowActions.tsx` — 承認・却下・支払完了ボタン

### table（3ファイル）
- `ApprovalListPage.tsx` — 承認待ち一覧テーブル
- `PaymentListPage.tsx` — 支払待ち一覧テーブル
- `ReportListTable.tsx` — レポート一覧テーブル

## 影響

- MUI テーマによるスタイル統一が効かない
- アクセシビリティ属性が MUI のデフォルトで提供されない
- 視覚的な不統一（ボタンの見た目がページ間で異なる）

## 提案

素の `<button>` → MUI `Button`、素の `<table>` → MUI `Table` 系コンポーネントに置換。
Step 10 で修正済みの CreateReportButton / ItemTable / OwnerActions と同じパターン。

---

## 解決内容
素 HTML button/table を MUI Button/Table に置換後、さらに AppDataGrid + AppPagination に移行（ApprovalListPage / PaymentListPage / ReportListTable）。AppPagination に totalPages <= 1 ガードを追加。PR #34 でマージ済み。遷移アイコン列は別 PR で対応中。

## 解決日
2026-04-11

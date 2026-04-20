# 一覧画面のスケルトン表示がページ全体を置換（設計はテーブル領域のみ）

## 発見日
2026-04-19

## カテゴリ
implementation / frontend / ui / design-consistency

## 影響度
中（ローディング中にフィルタ・ヘッダーが消失し、UX と設計意図に乖離）

## 発見経緯
Step 11-A ローカル動作確認（SMK-013 一覧読み込み中のスケルトン/スピナー）で、マイレポート画面を F5 再読込した際、ローディング中は**タイトル「マイレポート」・「レポート作成」ボタン・フィルタ UI（ステータス / 開始日 / 終了日）が全て非表示**となり、テーブル行のスケルトンバーのみが表示されることを観察。テーブル構造を模した UI ではなく、単なるグレーバーの羅列として視認された。

## 関連ステップ
Step 4（基本設計 screens.md §4.5）/ Step 5（report-list.md §8）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（UX 品質の問題、機能動作には影響なし）

## 問題

### 設計仕様

- `40_basic_design/screens.md §4.5`
  > API通信中はメインコンテンツ領域にスケルトンUIを表示する
  > 一覧画面: **テーブル行のスケルトン**
  > 詳細画面: **カード要素のスケルトン**

- `50_detail_design/screens/report-list.md §8`
  > 初回読み込み: **テーブル行**のスケルトンUI
  > ページ切替時: **テーブル領域に**スケルトンUIを表示

→ 設計意図は「**テーブル部分のみ**をスケルトンで置換し、タイトル・フィルタ・ボタン等のレイアウトは維持」

### 実装の挙動

一覧系ページで `isLoading` 時にページ全体を `<PageSkeleton variant="table" />` で早期 return しており、ヘッダー・フィルタ UI を含めて全て置換される。

該当箇所:

| ファイル | 行 | パターン |
|---------|-----|---------|
| `expense-saas/frontend/src/pages/reports/ReportListPage.tsx` | 131-134 | 早期 return（title/filter/ボタン全て消失） |
| `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx` | 205-208 | 早期 return（data-testid 付き div 内だがフィルタ類も消失） |
| `expense-saas/frontend/src/pages/workflow/PaymentListPage.tsx` | 205-208 | 同上 |

参考（ページではなくテーブルコンポーネントのみスケルトン化しており、設計通り）:
- `expense-saas/frontend/src/pages/admin/AllReportsTable.tsx:73-75`

### UX への影響

- ページ送り・フィルタ変更時にフィルタ UI が瞬間的に消えて戻る（画面のガタつき）
- ローディング中に操作コンテキスト（現在のフィルタ条件、ページ番号）が視覚的に失われる
- 「テーブル行の」スケルトンとして認知できず、単なる灰色バーの羅列に見える

## 修正方針（案）

`isLoading` で早期 return せず、テーブル部分だけ条件分岐:

```tsx
// NG（現状）
if (isLoading) {
  return <PageSkeleton variant="table" />;
}
return <Box>{/* header + filter + table */}</Box>;

// OK（修正案）
return (
  <Box>
    {/* header + filter は常に表示 */}
    <Box>{/* header */}</Box>
    <Box>{/* filter */}</Box>
    {isLoading ? (
      <PageSkeleton variant="table" />
    ) : (
      <Box>{/* table */}</Box>
    )}
    {pagination && <AppPagination />}
  </Box>
);
```

**検討事項**:
- フィルタ・ヘッダーの初回表示タイミング（初回ロード時はデータがまだないが、フィルタ UI の初期値は URL クエリから取得可能なので表示可能）
- ページネーションの扱い（ローディング中は非表示 or 前回のページ情報を保持）

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`
- `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx`
- `expense-saas/frontend/src/pages/workflow/PaymentListPage.tsx`
- 上記 3 ファイルのテスト（`__tests__/*ListPage.test.tsx`）も合わせて修正
  - 「isLoading=true のとき PageSkeleton が表示される」のみ検証している既存テストに、「isLoading=true でもヘッダー・フィルタが表示される」観点を追加

## 対象外（別判断）

- `ReportDetailPage.tsx`（詳細画面）: 設計書に「詳細画面: カード要素のスケルトン」とあるが、「カード要素」の範囲が曖昧。本 issue のスコープは一覧画面に限定し、詳細画面は別途必要に応じて起票する
- `DashboardPage.tsx`: 同様に本 issue のスコープ外

## 完了条件

- 対象 3 ファイルで、ローディング中もページヘッダー（タイトル + 主アクションボタン）とフィルタ UI が表示される
- スケルトン表示位置はテーブル領域のみ
- 対応するテストが更新され通過する
- 設計書と実装が一致している

## 次セッション着手手順

1. ブランチ作成: `fix/116-list-skeleton-scope`
2. ReportListPage → ApprovalListPage → PaymentListPage の順に修正
3. 各テストで「isLoading 中もヘッダー/フィルタが描画される」ケースを追加
4. ローカルで DevTools の Slow 3G 絞りで視覚確認
5. PR 作成

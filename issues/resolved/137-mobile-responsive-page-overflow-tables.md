# スマホ幅（375px）でレポート一覧・明細一覧ページがページ全体横スクロールになる（TableContainer 未使用 + 列幅制御欠落）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design / responsive

## 影響度
中〜高（スマホ幅でメインコンテンツが viewport をはみ出し、ヘッダー・フィルタも含めて横スクロールしないと表示しきれない。スマホ UX が成立していない）

## 発見経緯
Step 11-A SMK-061（スマホ幅 375px の主要画面）実施中。DevTools で iPhone SE 幅（375px）に設定し、レポート一覧（`/reports`）とレポート詳細の明細一覧を確認したところ、テーブル単体ではなく**ページ全体が横スクロール**する挙動を確認。ヘッダー・フィルタ・ナビなども viewport からはみ出し、指で横スワイプしないと全体を見られない状態。

## 関連ステップ
Step 4（基本設計 — レスポンシブ）/ Step 5.5（UI コンポーネント設計）/ Step 10-C（レポート一覧）/ Step 10-D（レポート詳細・明細）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（機能動作は正常。スマホ UX の品質問題）

## 問題

### 現状の挙動

スマホ幅（375px）で以下の画面を開くと、ページ全体が横方向に overflow し、ヘッダー・フィルタ・テーブルすべてがスクロールしないと表示しきれない:

| 画面 | ファイル | 現象 |
|------|---------|------|
| レポート一覧 | `ReportListPage.tsx` | テーブル幅に沿ってページ全体が拡大 |
| レポート詳細の明細一覧 | `ItemTable.tsx`（`ItemListSection` 経由） | 同上 |

### 正しく動作している画面（比較対象）

| 画面 | 実装方式 | 挙動 |
|------|---------|------|
| ダッシュボード 直近レポート | `RecentReportList.tsx` | `TableContainer` で囲み、コンテナ内で横スクロール |
| ダッシュボード 月別サマリ | `MonthlySummaryTable.tsx` | 同上 |
| 承認待ち一覧 / 支払待ち一覧 | `ApprovalListPage` / `PaymentListPage` | MUI X `DataGrid`（`AppDataGrid`）を使用 |
| テナント全レポート一覧 | `AllReportsTable.tsx` | MUI X `DataGrid`（`AppDataGrid`）を使用 |

### 根本原因

`ReportListPage.tsx` L217 と `ItemTable.tsx` L33 は `<Table>` コンポーネントを**`<TableContainer>` で囲まずに**直置きしている。`TableContainer` は `overflow-x: auto` を提供するため、囲まない場合テーブルの intrinsic width（すべての列の合計幅）が親要素を押し広げ、最終的に `body` レベルで横スクロールが発生する。

```tsx
// ReportListPage.tsx L217（TableContainer なし）
<Table>
  <TableHead>
    <TableRow>
      <TableCell>タイトル</TableCell>
      <TableCell>期間</TableCell>
      ...
```

```tsx
// ItemTable.tsx L33（TableContainer なし）
<Table size="small" data-testid="item-table">
  <TableHead>
    ...
```

### 設計書との整合

**ui-guidelines.md § レスポンシブ対応**:
- md 未満: Drawer をハンバーガーメニューで開閉（temporary）
- レイアウトは Grid / Stack で構成し、ブレークポイントに応じてカラム数を変更する

**screens.md § レスポンシブ対応**:
- スマートフォン (767px以下): メインコンテンツは**全幅**

「メインコンテンツは全幅」という設計意図に反して、現状はメインコンテンツが viewport をはみ出している。

**smoke_check.md SMK-061 期待**:
> サイドナビがハンバーガーに収納され、テーブルがカード表示またはスクロール可能になる

「テーブルが**カード表示またはスクロール可能**」という期待に対し、現状は「ページ全体がスクロール」であり、テーブル単体でのスクロールになっていない。

## 修正方針

### 案 A: TableContainer で囲む（最小修正・推奨）

`ReportListPage.tsx` と `ItemTable.tsx` の `<Table>` を `<TableContainer>` で囲む。これによりテーブルの横 overflow が `TableContainer` 内に閉じ込められ、ページ全体の幅は viewport 以内に収まる。

```tsx
// ReportListPage.tsx
import TableContainer from '@mui/material/TableContainer';

<TableContainer>
  <Table>
    <TableHead>
      ...
```

```tsx
// ItemTable.tsx
import TableContainer from '@mui/material/TableContainer';

<TableContainer>
  <Table size="small" data-testid="item-table">
    ...
```

**効果**:
- テーブルの横 overflow が内部コンテナで完結
- ヘッダー・フィルタ・ナビは viewport 幅で正しくレイアウト
- SMK-061 期待の「テーブルが…スクロール可能」を満たす

**波及する変更**:
- 既存テスト（`ReportListPage.test.tsx`, `ItemTable.test.tsx`）で `getByRole('table')` 等の取り回しは維持される（`TableContainer` は Paper ベースの wrapper）
- 既存テストがスナップショット比較している場合は差分再生成

### 案 B: テーブル全体をカード化（大規模）

スマホ幅では行単位で Card に切り替える。実装コスト大で、既存テスト全面改修が必要。MVP スコープでは不採用。

### 案 C: `TableContainer` + 列数削減（余裕があれば）

スマホ幅で低優先度の列（`合計金額` など）を非表示にする。将来拡張として検討。

### 推奨

**案 A**。理由:
- 最小修正で SMK-061 期待を満たす
- ダッシュボードの既存実装（`RecentReportList` / `MonthlySummaryTable`）と整合する
- 既存テストへの影響がほぼ無い

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ReportListPage.tsx` — `<Table>` を `<TableContainer>` で囲み、`import TableContainer` を追加
- `expense-saas/frontend/src/pages/reports/ItemTable.tsx` — 同上
- （必要に応じ）`expense-saas/frontend/src/pages/reports/__tests__/ReportListPage.test.tsx` / `ItemTable.test.tsx` — スナップショット差分の再生成

## 完了条件

- スマホ幅（375px）でレポート一覧ページを開いたとき、ヘッダー・フィルタ・ナビが viewport 内に収まる
- テーブルは `TableContainer` 内で横スクロール可能（必要に応じてスワイプで全列確認可能）
- 明細一覧（レポート詳細の `ItemTable`）も同様
- PC 幅（1280px）での表示が従来通り（レイアウト崩れなし）
- 既存テストが通る

## 関連 SMK / issue

- **SMK-061**（現在 FAIL）: 本 issue で解消対象
- **SMK-062**（スマホ幅でのフォーム操作）: フォーム側は別途確認が必要（本 issue のスコープ外）
- **SMK-101**（テナント全レポート一覧のスマホ幅表示）: `AllReportsTable` は `DataGrid` 使用のため別挙動。Phase 6 Admin で検証
- 補足: dashboard の `RecentReportList` / `MonthlySummaryTable` は既に `TableContainer` 使用済み（本 issue のスコープ外、正常動作）

## 二次影響（本 issue 解消で連動的に改善される）

SMK-062（スマホ幅でのフォーム操作）実施中に判明した二次影響:

- **明細追加時のスライドパネルが横に伸びる**: レポート詳細画面で明細を 1 件追加すると `ItemTable` がページを押し広げ、**以降に開く明細追加スライドパネルも viewport を超えた幅**で描画される（既に伸びたページ幅を親要素として継承するため）
- 本 issue で `ItemTable` を `TableContainer` で囲むとページ幅が viewport 内に収まるため、スライドパネルも正しい幅で開くようになる
- SMK-062 の明細追加関連の FAIL 要因として本 issue を解消すれば自動的に PASS になる想定

---

## 解決内容

PR #92 にて #139 と合わせて対応。`ReportListPage.tsx` を `ReportListTable`（AppDataGrid ベース）に差し替えたことで、DataGrid が内部 scroll container を持ちページ全体横スクロール問題が解消。`ItemTable.tsx` は `<TableContainer>` で `<Table>` を囲む修正を実施。

- 解決 PR: #92（#137 + #139 合算）
- マージ commit: b7f4430

## 解決日

2026-04-22

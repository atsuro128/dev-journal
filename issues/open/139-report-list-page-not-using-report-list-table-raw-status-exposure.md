# レポート一覧が設計済み ReportListTable / StatusChip を使わず status 英語生値を露出（SCR-RPT-001 実装乖離）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design / dead-code

## 影響度
高（ステータス列に `draft`/`submitted` 等の英語生値が露出し、SMK-053 / NFR-UX-002 に違反。設計書で指定されたコンポーネント `StatusChip` が未接続で、設計と実装が乖離している）

## 発見経緯
Step 11-A SMK-050（ラベル・ボタンの日本語）実施中、レポート一覧のステータス列が `submitted` / `draft` 等の英語で表示されていることをユーザーが発見。ダッシュボードの直近レポート一覧では日本語（下書き / 提出済み 等）の Chip 表示になっているのに、レポート一覧画面だけが英語露出している不整合に気付いた。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10-C（レポート一覧実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（機能動作は正常。表示仕様の乖離）

## 問題

### 現状の挙動

`ReportListPage.tsx` L217-244 はレポート一覧を MUI `<Table>` で直接描画しており、ステータスセルで status 生値をそのまま表示している。

```tsx
// ReportListPage.tsx L238
<TableCell>{report.status}</TableCell>
```

→ `status` は openapi 上 `'draft' | 'submitted' | 'approved' | 'rejected' | 'paid'` の文字列。英語値がそのままブラウザに表示される。

また、合計金額列も設計と異なるフォーマット:

```tsx
// ReportListPage.tsx L239
<TableCell>{report.total_amount.toLocaleString()} 円</TableCell>
```

→ 設計 (`ReportListTable.tsx` L48) は `¥${value.toLocaleString()}` プレフィックス形式。

### 設計上の正

`common-components.md` L291-315 で `StatusChip` コンポーネントが定義されており、**使用箇所に SCR-RPT-001（レポート一覧）が含まれる**:

> - 配置: `components/ui/StatusChip.tsx`
> - 責務: 経費レポートのステータスを色分けした Chip で表示する
> - 使用箇所: SCR-DASH-001, SCR-RPT-001, SCR-RPT-004, SCR-ADM-001

`StatusChip` は `01_glossary.md` L44-48 の用語集と完全一致する日本語マッピングを提供する:

| status | 表示テキスト |
|--------|------------|
| `draft` | 下書き |
| `submitted` | 提出済み |
| `approved` | 承認済み |
| `rejected` | 却下 |
| `paid` | 支払済み |

### 設計済みなのに未接続のコンポーネント

さらに、`ReportListTable.tsx`（同 `pages/reports/` ディレクトリ）には**設計仕様に準拠した正しい実装**が存在している:

```tsx
// ReportListTable.tsx L28-64
const COLUMNS: GridColDef[] = [
  { field: 'title', headerName: 'タイトル', flex: 2 },
  { field: 'period', headerName: '対象期間', flex: 2, renderCell: ... },
  { field: 'totalAmount', headerName: '合計金額', flex: 1,
    valueFormatter: (value: number) => `¥${value.toLocaleString()}` },  // ¥ プレフィックス
  { field: 'status', headerName: 'ステータス', flex: 1,
    renderCell: (params) => <StatusChip status={params.value as ReportStatus} /> },  // StatusChip 使用
  { field: 'createdAt', headerName: '作成日', flex: 1, ... },
];
```

この `ReportListTable` は `MUI X DataGrid`（`AppDataGrid`）経由でレンダリングされ、StatusChip + 金額 ¥ プレフィックス + 作成日列を含む正しい実装になっている。しかし `ReportListPage.tsx` には `ReportListTable` の import がなく、**使われているのはテストファイルのみ**（`ReportListPage.test.tsx` ではなく `ReportListTable.test.tsx`）。

### ダッシュボードとの非対称

一方、ダッシュボードの `RecentReportList.tsx`（直近レポート 5 件）は `StatusChip` を正しく使用しており、日本語で Chip 表示される。同じユーザーが同じアプリ内で異なる表示を見ることになり、一貫性が崩れている。

### 表示列の差分

| 列 | ReportListPage（現状実装） | ReportListTable（設計済み・未接続） |
|----|-------------------------|----------------------------------|
| タイトル | ○ | ○ |
| 期間 | ○（"YYYY-MM-DD 〜 YYYY-MM-DD"） | ○（同じ） |
| ステータス | **英語生値** | **StatusChip（日本語）** |
| 合計金額 | `1,234 円` | `¥1,234` |
| 作成日 | **欠落** | ○（`toLocaleDateString('ja-JP')`） |

→ 作成日列も現状の `ReportListPage` には実装されていない。

### 副次的な用語ゆれ（関連するが別 issue 候補）

`ReportListPage.tsx` L25-29 のステータスフィルタドロップダウンで、**用語集と不整合**な表記が使われている:

| status | 用語集 / StatusChip | ReportListPage フィルタ |
|--------|-------------------|----------------------|
| rejected | 却下 | **却下済み** |
| paid | 支払済み | **支払完了**（操作の用語） |

これは本 issue の範囲外だが、レポート一覧画面を直すタイミングで同時に修正できる可能性があるため記録。別 issue 化するか、本 issue に取り込むかは着手時に判断する。

## 修正方針

### 案 A: ReportListPage を ReportListTable に差し替える（推奨）

`ReportListPage.tsx` の `<Table>` ブロック（L216-244）を `ReportListTable` コンポーネントに差し替える。

```tsx
import ReportListTable from './ReportListTable';

// ...
{isLoading ? (
  <PageSkeleton variant="table" />
) : (
  <ReportListTable
    reports={reports.map(r => ({
      id: r.id,
      title: r.title,
      periodStart: r.period_start,
      periodEnd: r.period_end,
      totalAmount: r.total_amount,
      status: r.status,
      createdAt: r.created_at,
    }))}
    onRowClick={(id) => navigate(`/reports/${id}`)}
    onCreateReport={() => navigate('/reports/new')}
  />
)}
```

**効果**:
- StatusChip 日本語表示
- 金額 ¥ プレフィックス（設計整合）
- 作成日列の追加
- EmptyState 適切表示（レポート 0 件時に「経費レポートはまだありません…」）
- デッドコード状態の `ReportListTable` が有効化される

**波及する変更**:
- `useReports` フックから取る型（ReportListItemResponse）を `ReportListItem` にマッピング
- 既存テスト `ReportListPage.test.tsx` の DOM 選択方法（`report-row-${id}` testid 等）を DataGrid ベースに変更
- ページネーション連携: `ReportListTable` は `hideFooterPagination` で DataGrid 内部のページングを隠し、既存の `AppPagination` を継続使用する構造を維持

### 案 B: 既存 `<Table>` に StatusChip を後付けして最小修正（代替案）

`ReportListPage.tsx` L238 を `<TableCell><StatusChip status={report.status} /></TableCell>` に置換し、金額フォーマットも `¥${report.total_amount.toLocaleString()}` に修正する。ReportListTable は引き続きデッドコードのままになる。

- メリット: テスト影響最小
- デメリット: 作成日列の欠落・DataGrid 機能（並び替え・フィルタ）の欠如が残る。`ReportListTable.tsx` と `ReportListTable.test.tsx` が使われない状態が続く

### 推奨

**案 A**。理由:

- 設計書 `common-components.md` / `screens.md` で SCR-RPT-001 は DataGrid + StatusChip 前提の仕様になっているため、表面的修正では設計乖離が残る
- `ReportListTable` コンポーネント + テストが既に実装済みのため再利用コストが低い
- 副作用として issue #137（TableContainer 未使用問題）も DataGrid 化で自動解決する可能性あり（`AppDataGrid` は内部で scroll container を持つ）

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ReportListPage.tsx` — `<Table>` を `<ReportListTable>` に差し替え、不要 import 削除
- `expense-saas/frontend/src/pages/reports/__tests__/ReportListPage.test.tsx` — DOM 選択方法を DataGrid ベースに更新
- （検討）`ReportListPage.tsx` ステータスフィルタの用語を用語集に揃える（rejected: 却下, paid: 支払済み）→ 本 issue 取り込み or 別 issue

## 完了条件

- レポート一覧のステータス列が日本語（StatusChip）で表示される（`draft→下書き` 等、用語集 L44-48 と一致）
- 合計金額列が `¥` プレフィックス形式で表示される
- 作成日列が追加される
- EmptyState が 0 件時に表示される
- 既存テストが通る
- ダッシュボードとレポート一覧で**同じステータス表記・同じスタイル**が使われ、UX 一貫性が保たれる

## 関連 SMK / issue

- **SMK-050**（ラベル・ボタンの日本語）: 本 issue 解消で英語生値露出が解消
- **SMK-053**（状態ラベル）: 未実施だが本 issue で事前修正すれば PASS できる
- **#137**（スマホ幅でページ全体横スクロール）: 本 issue 案 A で DataGrid 化すると ReportListPage 側は自動解消の可能性。ItemTable 側は #137 で別途対応
- **#108 / #116 / #123 / #124 / #126**: SCR-RPT-001 系の既存修正ではステータス列の英語露出に踏み込まれていない
- 副次: `ReportListPage.tsx` L25-29 のフィルタ用語ゆれ（却下済み / 支払完了）→ 別 issue 化候補

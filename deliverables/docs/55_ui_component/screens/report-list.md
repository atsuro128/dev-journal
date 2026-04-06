# レポート一覧（自分） -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/report-list.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/report-list.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-RPT-001` |
| 画面名 | レポート一覧（自分） |
| URL パス | `/reports` |
| 画面詳細仕様 | `50_detail_design/screens/report-list.md` |

---

## 2. コンポーネントツリー

```
ReportListPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── ReportListHeader
        │   └── CreateReportButton
        ├── ReportListFilter
        │   ├── AppSelect（← common-components.md）[ステータス]
        │   ├── AppDatePicker（← common-components.md）[開始日]
        │   └── AppDatePicker（← common-components.md）[終了日]
        ├── ReportListTable
        │   ├── AppDataGrid（← common-components.md）
        │   │   └── StatusChip（← common-components.md）[各行]
        │   └── EmptyState（← common-components.md）[データ0件時]
        └── AppPagination（← common-components.md）
```

---

## 3. コンポーネント定義

### ReportListPage

- 配置: `pages/reports/ReportListPage.tsx`
- 責務: レポート一覧画面のページコンポーネント。URL クエリパラメータからフィルタ条件を復元し、useMyReports Hook でデータを取得する。フィルタ変更時に URL クエリパラメータを更新してページ番号を 1 にリセットする。レポート作成ボタン押下時に `/reports/new` に遷移する
- 対応セクション: `50_detail_design/screens/report-list.md` &sect;1, &sect;4, &sect;5, &sect;11

```typescript
// Props なし（ページコンポーネント）
```

### ReportListHeader

- 配置: `pages/reports/ReportListHeader.tsx`
- 責務: ページタイトル「マイレポート」とレポート作成ボタンを横並びに配置する
- 対応セクション: `50_detail_design/screens/report-list.md` &sect;2, &sect;6

```typescript
interface ReportListHeaderProps {
  /** レポート作成ボタン押下時のコールバック */
  onCreateReport: () => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `onCreateReport` | `() => void` | Yes | 「+ レポート作成」ボタン押下時のコールバック | ReportListPage（navigate 呼び出し） |

### CreateReportButton

- 配置: `pages/reports/CreateReportButton.tsx`
- 責務: 「+ レポート作成」ボタンを表示する。空状態メッセージ内にも再利用される
- 対応セクション: `50_detail_design/screens/report-list.md` &sect;6, &sect;7

```typescript
interface CreateReportButtonProps {
  /** ボタン押下時のコールバック */
  onClick: () => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `onClick` | `() => void` | Yes | ボタン押下時のコールバック | 親コンポーネント |

### ReportListFilter

- 配置: `pages/reports/ReportListFilter.tsx`
- 責務: ステータスフィルタ（ドロップダウン）と対象期間フィルタ（日付ピッカー x 2）を横並びに配置する。フィルタ変更時に onFilterChange コールバックを呼び出す。日付入力にはデバウンス処理を適用する
- 対応セクション: `50_detail_design/screens/report-list.md` &sect;4

```typescript
interface ReportListFilterValues {
  /** ステータスフィルタ値。空文字は「全て」 */
  status: string;
  /** 対象期間開始日（YYYY-MM-DD 形式、または空文字） */
  from: string;
  /** 対象期間終了日（YYYY-MM-DD 形式、または空文字） */
  to: string;
}

interface ReportListFilterProps {
  /** 現在のフィルタ値 */
  values: ReportListFilterValues;
  /** フィルタ変更時のコールバック */
  onFilterChange: (values: ReportListFilterValues) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `values` | `ReportListFilterValues` | Yes | 現在のフィルタ値 | ReportListPage（URL クエリパラメータから復元） |
| `onFilterChange` | `(values: ReportListFilterValues) => void` | Yes | フィルタ変更時のコールバック | ReportListPage |

### ReportListTable

- 配置: `pages/reports/ReportListTable.tsx`
- 責務: 経費レポートの一覧をテーブル形式で表示する。AppDataGrid をラップし、カラム定義（タイトル・対象期間・合計金額・ステータス・作成日）を設定する。行クリック時に onRowClick コールバックを呼び出す。データが 0 件の場合は EmptyState を表示する
- 対応セクション: `50_detail_design/screens/report-list.md` &sect;3, &sect;7, &sect;8

```typescript
interface ReportListItem {
  /** レポート ID */
  id: string;
  /** タイトル */
  title: string;
  /** 対象期間開始日 */
  periodStart: string;
  /** 対象期間終了日 */
  periodEnd: string;
  /** 合計金額 */
  totalAmount: number;
  /** ステータス */
  status: 'draft' | 'submitted' | 'approved' | 'rejected' | 'paid';
  /** 作成日 */
  createdAt: string;
}

interface ReportListTableProps {
  /** レポート一覧データ */
  reports: ReportListItem[];
  /** データ読み込み中フラグ */
  loading: boolean;
  /** 行クリック時のコールバック（レポート ID を引数） */
  onRowClick: (reportId: string) => void;
  /** レポート作成コールバック（空状態のアクションボタン用） */
  onCreateReport: () => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reports` | `ReportListItem[]` | Yes | レポート一覧データ | useMyReports Hook の data |
| `loading` | `boolean` | Yes | データ読み込み中フラグ | useMyReports Hook の isLoading |
| `onRowClick` | `(reportId: string) => void` | Yes | 行クリック時のコールバック | ReportListPage（navigate 呼び出し） |
| `onCreateReport` | `() => void` | Yes | 空状態でのレポート作成コールバック | ReportListPage（navigate 呼び出し） |

---

## 4. データフロー

```
GET /api/reports?status=...&from=...&to=...&page=1&per_page=20
  → useMyReports(params)（← state-management.md §3 データフェッチ系）
    → ReportListPage
      → ReportListHeader (props: onCreateReport)
        → CreateReportButton (props: onClick)
      → ReportListFilter (props: values, onFilterChange)
        → AppSelect (props: value, onChange) [ステータス]
        → AppDatePicker x 2 (props: value, onChange) [期間]
      → ReportListTable (props: reports, loading, onRowClick, onCreateReport)
        → AppDataGrid (props: columns, rows, loading, emptyMessage)
          → StatusChip (props: status) [各行のステータスセル]
        → EmptyState (props: message, action) [データ0件時]
      → AppPagination (props: currentPage, totalPages, onPageChange)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useMyReports` | 自分のレポート一覧データの取得 | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | ヘッダーのユーザー情報表示 | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| レポート一覧データ | サーバー | useMyReports Hook（TanStack Query useQuery） | ReportListPage |
| ページネーション情報（currentPage, totalPages） | サーバー | useMyReports Hook のレスポンス pagination | ReportListPage |
| フィルタ条件（status, from, to） | UI | URL クエリパラメータ（useSearchParams） | ReportListPage |
| 現在のページ番号 | UI | URL クエリパラメータ（useSearchParams） | ReportListPage |
| データ読み込み中フラグ | サーバー | useMyReports Hook の isLoading | ReportListPage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | ReportListPage のルートラッパー | children にメインコンテンツを配置 |
| `AppSelect`（← common-components.md） | ReportListFilter 内のステータスフィルタ | options: 全て / 下書き / 提出済み / 承認済み / 却下 / 支払済み |
| `AppDatePicker`（← common-components.md） | ReportListFilter 内の対象期間フィルタ（x 2） | label: 開始日 / 終了日 |
| `AppDataGrid`（← common-components.md） | ReportListTable 内のテーブル | columns: タイトル・対象期間・合計金額・ステータス・作成日。ステータス列は StatusChip でカスタムレンダリング |
| `StatusChip`（← common-components.md） | ReportListTable 内の各行ステータスセル | status: レポートの status 値 |
| `EmptyState`（← common-components.md） | ReportListTable 内のデータ 0 件時 | message: 「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」、action: レポート作成ボタン |
| `AppPagination`（← common-components.md） | テーブル下部のページネーションコントロール | currentPage, totalPages は useMyReports のレスポンスから取得 |
| `PageSkeleton`（← common-components.md） | 初回読み込み時のスケルトン UI | variant: 'table' |
| `AppToast`（← common-components.md） | API 通信エラー時のトースト通知 | severity: 'error' |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `ReportListPage` | 認証済みユーザー（全ロール） | - | `authz.md` &sect;6.3（GET /api/reports: 全ロール許可） |
| `CreateReportButton` | 常時表示（全ロール共通） | 常時有効 | `screens/report-list.md` &sect;6 |
| `ReportListTable` | 常時表示（自分のレポートのみ表示） | - | `screens/report-list.md` &sect;10 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | ReportListPage | URL パス、対応 API |
| &sect;2 レイアウト | AppLayout, ReportListHeader, ReportListFilter, ReportListTable | 全体構成 |
| &sect;3 表示項目 | ReportListTable, StatusChip | テーブルカラム定義、ステータスバッジ色 |
| &sect;4 フィルタ | ReportListFilter, AppSelect, AppDatePicker | ステータス・期間フィルタ。フィルタ変更時の即時更新、URL クエリパラメータ反映 |
| &sect;5 ページネーション | AppPagination | オフセットベース、20件/ページ |
| &sect;6 操作 | CreateReportButton, ReportListTable（行クリック） | レポート作成遷移、詳細表示遷移 |
| &sect;7 空状態 | EmptyState | メッセージ + レポート作成ボタン |
| &sect;8 ローディング | PageSkeleton | スケルトン UI（variant: 'table'） |
| &sect;9 エラーハンドリング | AppToast | API 通信エラー: トースト、401: ログインリダイレクト |
| &sect;10 ロール別表示差異 | - | ロールによる差異なし |
| &sect;11 画面遷移 | ReportListPage | 行クリック -> 詳細、作成ボタン -> 作成画面 |
| &sect;12 API リクエスト/レスポンス | useMyReports Hook | GET /api/reports |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/report-list.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/report-list.md §ReportListPage` | コンポーネント単体テスト | URL クエリパラメータからのフィルタ復元、フィルタ変更時のページリセット、行クリック遷移、作成ボタン遷移 |
| `55_ui_component/screens/report-list.md §ReportListHeader` | コンポーネント単体テスト | タイトル表示、レポート作成ボタンの描画と onCreateReport コールバック |
| `55_ui_component/screens/report-list.md §CreateReportButton` | コンポーネント単体テスト | ボタン描画と onClick コールバック |
| `55_ui_component/screens/report-list.md §ReportListFilter` | コンポーネント単体テスト | ステータスフィルタの選択肢表示、日付ピッカーの入力、フィルタ変更時の onFilterChange コールバック |
| `55_ui_component/screens/report-list.md §ReportListTable` | コンポーネント単体テスト | テーブルカラム表示、金額フォーマット（3桁カンマ区切り）、対象期間フォーマット、ステータスバッジ表示、行クリック、空状態表示 |
| `55_ui_component/state-management.md §useMyReports` | Hook 単体テスト | API 呼び出し・フィルタパラメータ送信・ページネーション応答のハンドリング |

# 支払待ち一覧 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/workflow-payable.md）、共通コンポーネント詳細（common-components.md）、支払完了のビジネスロジック詳細（SCR-RPT-004 で実行） |
| 主な参照元 | `50_detail_design/screens/workflow-payable.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-WFL-002` |
| 画面名 | 支払待ち一覧 |
| URL パス | `/payments` |
| 画面詳細仕様 | `50_detail_design/screens/workflow-payable.md` |

---

## 2. コンポーネントツリー

```
PayableReportsPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── PayableReportsContent
        ├── PageTitle
        ├── PayableFilterBar
        │   ├── AppTextField（← common-components.md）[申請者名]
        │   └── FilterResetButton
        ├── PayableReportCount
        ├── PayableReportTable
        │   ├── AppDataGrid（← common-components.md）
        │   │   └── PayableReportRow（仮想行 -- DataGrid renderCell）
        │   │       └── SelfLabel（条件付き表示）
        │   ├── EmptyState（← common-components.md）[データ0件時]
        │   └── PageSkeleton（← common-components.md）[ローディング時]
        └── AppPagination（← common-components.md）
```

---

## 3. コンポーネント定義

### PayableReportsPage

- 配置: `pages/payments/PayableReportsPage.tsx`
- 責務: 支払待ち一覧画面のページコンポーネント。AppLayout でラップし、PayableReportsContent を配置する。usePayableReports Hook を呼び出してデータを取得し、フィルタ・ページネーションの状態を管理する。Accounting ロール以外のアクセスはダッシュボードにリダイレクトされる（RBAC ミドルウェアが 403 を返し、グローバルエラーハンドラがリダイレクト）
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;1, &sect;12, &sect;13

```typescript
// Props なし（ページコンポーネント）
```

### PayableReportsContent

- 配置: `pages/payments/PayableReportsContent.tsx`
- 責務: 支払待ち一覧のメインコンテンツ領域。ページタイトル、フィルタエリア、件数表示、テーブル、ページネーションを統合する。usePayableReports の結果に応じてローディング・空状態・テーブル表示を切り替える
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;4

```typescript
interface PayableReportsContentProps {
  /** 支払待ちレポート一覧データ */
  reports: PayableReport[];
  /** ページネーション情報 */
  pagination: Pagination; // ← api/types.ts の Pagination 型
  /** ローディング状態 */
  isLoading: boolean;
  /** エラー状態 */
  error: ApiClientError | null;
  /** 現在のフィルタ条件 */
  filters: PayableReportListParams;
  /** フィルタ変更コールバック */
  onFilterChange: (filters: PayableReportListParams) => void;
  /** ページ変更コールバック */
  onPageChange: (page: number) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reports` | `PayableReport[]` | Yes | 支払待ちレポート一覧 | usePayableReports Hook の data.data |
| `pagination` | `Pagination` | Yes | ページネーション情報 | usePayableReports Hook の data.pagination |
| `isLoading` | `boolean` | Yes | データ取得中フラグ | usePayableReports Hook の isLoading |
| `error` | `ApiClientError \| null` | Yes | API エラー | usePayableReports Hook の error |
| `filters` | `PayableReportListParams` | Yes | 現在のフィルタ条件 | PayableReportsPage の useState |
| `onFilterChange` | `(filters: PayableReportListParams) => void` | Yes | フィルタ変更時のコールバック | PayableReportsPage |
| `onPageChange` | `(page: number) => void` | Yes | ページ変更時のコールバック | PayableReportsPage |

### PayableFilterBar

- 配置: `pages/payments/PayableFilterBar.tsx`
- 責務: 支払待ち一覧のフィルタエリア。申請者名テキスト入力（デバウンス 300ms）とフィルタリセットボタンを配置する。フィルタ変更時は onFilterChange コールバックで親に通知する
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;6

```typescript
interface PayableFilterBarProps {
  /** 現在のフィルタ条件 */
  filters: PayableReportListParams;
  /** フィルタ変更コールバック */
  onFilterChange: (filters: PayableReportListParams) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `filters` | `PayableReportListParams` | Yes | 現在のフィルタ条件 | PayableReportsContent |
| `onFilterChange` | `(filters: PayableReportListParams) => void` | Yes | フィルタ変更コールバック（デバウンス後に発火） | PayableReportsContent |

### PayableReportCount

- 配置: `pages/payments/PayableReportCount.tsx`
- 責務: テーブル上部に支払待ちレポートの件数を表示する。0件かつフィルタ未適用時は空状態メッセージに切り替わるため非表示、0件かつフィルタ適用中は「条件に一致するレポートはありません。」を表示する
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;8

```typescript
interface PayableReportCountProps {
  /** 総件数 */
  totalCount: number;
  /** フィルタが適用中かどうか */
  isFiltered: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `totalCount` | `number` | Yes | 支払待ちレポートの総件数 | pagination.total_count |
| `isFiltered` | `boolean` | Yes | フィルタが適用中か | filters の値から判定 |

### PayableReportTable

- 配置: `pages/payments/PayableReportTable.tsx`
- 責務: 支払待ちレポートの一覧テーブルを表示する。AppDataGrid をラップし、カラム定義（申請者名・タイトル・合計金額・承認日・遷移アイコン）を設定する。行クリックでレポート詳細画面（SCR-RPT-004）へ遷移する。自己レポート行には「自分」ラベルを表示する。ローディング中は PageSkeleton を、0件時は EmptyState を表示する。SCR-WFL-001（承認待ち）との差異は日付カラムが「提出日」ではなく「承認日」である点
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;5, &sect;9, &sect;10, &sect;12

```typescript
interface PayableReportTableProps {
  /** 支払待ちレポート一覧 */
  reports: PayableReport[];
  /** ローディング状態 */
  isLoading: boolean;
  /** データが0件の場合の空状態メッセージ */
  emptyMessage: string;
  /** 行クリック時の遷移コールバック */
  onRowClick: (reportId: string) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reports` | `PayableReport[]` | Yes | 支払待ちレポート一覧 | PayableReportsContent |
| `isLoading` | `boolean` | Yes | データ取得中フラグ | PayableReportsContent |
| `emptyMessage` | `string` | Yes | 空状態メッセージ | PayableReportsContent（フィルタ状態に応じて切替） |
| `onRowClick` | `(reportId: string) => void` | Yes | 行クリック時の遷移コールバック | PayableReportsContent（React Router navigate） |

### SelfLabel

- 配置: `components/ui/SelfLabel.tsx`
- 責務: 自分が作成したレポートであることを示す「自分」ラベルを Chip で表示する。申請者名の横に配置する。`is_own_report` フラグが true の場合のみ表示する
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;5（自己支払処理禁止の表示ルール）

Props 型定義は `common-components.md §SelfLabel` を参照。

### FilterResetButton

- 配置: `components/ui/FilterResetButton.tsx`
- 責務: フィルタ条件をリセットするボタン。フィルタが適用されている場合にのみ有効化される
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;6

Props 型定義は `common-components.md §FilterResetButton` を参照。

### PageTitle

- 配置: `components/ui/PageTitle.tsx`
- 責務: ページタイトル（「支払待ち一覧」）を表示する Typography コンポーネント
- 対応セクション: `50_detail_design/screens/workflow-payable.md` &sect;4

Props 型定義は `common-components.md §PageTitle` を参照。

---

## 4. データフロー

```
GET /api/workflow/payable
  → usePayableReports(filters)（← state-management.md §3 データフェッチ系）
    → PayableReportsPage
      → PayableReportsContent (props: reports, pagination, isLoading, error, filters, onFilterChange, onPageChange)
        → PageTitle (props: title = "支払待ち一覧")
        → PayableFilterBar (props: filters, onFilterChange)
          → AppTextField (props: 申請者名入力)
          → FilterResetButton (props: onReset, isFiltered)
        → PayableReportCount (props: totalCount, isFiltered)
        → PayableReportTable (props: reports, isLoading, emptyMessage, onRowClick)
          → AppDataGrid (props: columns, rows, loading)
            → SelfLabel (props: isOwnReport -- renderCell 内)
          → EmptyState (props: message) [0件時]
          → PageSkeleton (props: variant = "table") [ローディング時]
        → AppPagination (props: currentPage, totalPages, onPageChange)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `usePayableReports` | 支払待ちレポート一覧の取得 | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | 現在のユーザー情報取得（AppHeader 表示用） | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| 支払待ちレポート一覧 | サーバー | usePayableReports Hook（TanStack Query useQuery） | PayableReportsPage |
| フィルタ条件（applicant_name） | UI | useState | PayableReportsPage |
| 現在のページ番号（page） | UI | useState | PayableReportsPage |
| デバウンス中の入力値 | UI | useState + useEffect（300ms デバウンス） | PayableFilterBar 内 |
| サイドバー開閉状態 | UI | useState | AppLayout 内 |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | PayableReportsPage のルートラッパー | children に PayableReportsContent を配置 |
| `AppHeader`（← common-components.md） | AppLayout 内部 | 標準使用 |
| `AppSidebar`（← common-components.md） | AppLayout 内部 | 支払待ち一覧メニューをアクティブ表示（currentPath = "/payments"） |
| `AppDataGrid`（← common-components.md） | PayableReportTable 内 | columns: 申請者名・タイトル・合計金額・承認日・遷移アイコンの5列。行クリックで遷移。emptyMessage は PayableReportTable が EmptyState で別途制御 |
| `AppTextField`（← common-components.md） | PayableFilterBar 内の申請者名フィルタ | name: "applicant_name"、label: "申請者名"。デバウンス 300ms で検索実行 |
| `AppPagination`（← common-components.md） | PayableReportsContent 下部 | currentPage, totalPages を usePayableReports の pagination から取得 |
| `EmptyState`（← common-components.md） | PayableReportTable 内（0件時） | message: フィルタ未適用時「支払待ちのレポートはありません。」、フィルタ適用中「条件に一致するレポートはありません。」。action: フィルタ適用中はリセットボタン |
| `PageSkeleton`（← common-components.md） | PayableReportTable 内（ローディング時） | variant: "table"、rows: 5 |
| `AppToast`（← common-components.md） | PayableReportsPage | サーバーエラー（500）時のトースト表示 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `PayableReportsPage` | Accounting ロールのみアクセス可能。他ロールは API が 403 を返し、ダッシュボード（SCR-DASH-001）にリダイレクト | - | `authz.md §6.6`（`/api/workflow/payable`: Accounting のみ）、`screens/workflow-payable.md` &sect;1 |
| `SelfLabel` | `is_own_report` が true のレポート行でのみ表示 | - | `screens/workflow-payable.md` &sect;5（自己支払処理禁止の表示ルール） |
| `PayableReportTable` 行クリック | 常時有効（全行遷移可能） | 自己レポートも含めて SCR-RPT-004 へ遷移可能。支払完了ボタンの表示制御は SCR-RPT-004 側で行う（RBC-012） | `screens/workflow-payable.md` &sect;5, &sect;12 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | PayableReportsPage | URL パス、対応 API、アクセス可能ロール |
| &sect;3 アクションの責務分担 | - | 支払完了は SCR-RPT-004 で実行。本画面は一覧表示とナビゲーションのみ |
| &sect;4 レイアウト | PayableReportsContent | ヘッダー + サイドナビ + メインコンテンツの構成 |
| &sect;5 表示項目（テーブルカラム） | PayableReportTable, AppDataGrid | 申請者名・タイトル・合計金額・承認日・遷移アイコンの5列。SCR-WFL-001 との差異は「提出日」→「承認日」 |
| &sect;5 自己支払処理禁止の表示ルール | SelfLabel | 「自分」ラベル表示。行クリックは通常通り遷移可能 |
| &sect;6 フィルタ | PayableFilterBar, AppTextField, FilterResetButton | 申請者名テキスト入力（デバウンス 300ms）+ リセットボタン |
| &sect;7 ページネーション | AppPagination | オフセットベース、20件/ページ |
| &sect;8 件数表示 | PayableReportCount | 「N 件の支払待ちレポート」/ フィルタ適用中0件メッセージ |
| &sect;9 空状態 | EmptyState | 「支払待ちのレポートはありません。」/「条件に一致するレポートはありません。」 |
| &sect;10 ローディング | PageSkeleton | variant: "table"、5行分のスケルトン UI |
| &sect;11 エラー表示 | AppToast, PayableReportsPage | サーバーエラー: トースト、401: ログインリダイレクト、403: ダッシュボードリダイレクト |
| &sect;12 行クリック時の遷移 | PayableReportTable | SCR-RPT-004 へ遷移。ブラウザ履歴に追加 |
| &sect;13 共通仕様 | PayableReportsPage | データ鮮度: 画面表示時に取得。自動リフレッシュなし（MVP） |
| &sect;15 API リクエスト/レスポンス | usePayableReports Hook | GET /api/workflow/payable |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/workflow-payable.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | コンポーネント単体テスト | Accounting 以外のアクセス時リダイレクト、データ取得・フィルタ・ページネーション状態の管理 |
| `55_ui_component/screens/workflow-payable.md §PayableReportsContent` | コンポーネント単体テスト | ローディング・空状態・テーブル表示の切り替え、フィルタ変更時のページリセット |
| `55_ui_component/screens/workflow-payable.md §PayableFilterBar` | コンポーネント単体テスト | 申請者名入力のデバウンス（300ms）、フィルタリセット |
| `55_ui_component/screens/workflow-payable.md §PayableReportCount` | コンポーネント単体テスト | 件数テキスト表示（「N 件の支払待ちレポート」）、0件時の非表示、フィルタ適用中の0件メッセージ |
| `55_ui_component/screens/workflow-payable.md §PayableReportTable` | コンポーネント単体テスト | テーブルカラムの描画（申請者名・タイトル・合計金額・承認日・遷移アイコン）、金額フォーマット（3桁カンマ区切り）、日付フォーマット（YYYY/MM/DD）、行クリック遷移、自己レポートの「自分」ラベル表示 |
| `55_ui_component/screens/workflow-payable.md §SelfLabel` | コンポーネント単体テスト | isOwnReport が true の場合のみラベル表示、false の場合は非表示 |
| `55_ui_component/screens/workflow-payable.md §FilterResetButton` | コンポーネント単体テスト | フィルタ適用中は有効、未適用時は disabled |
| `55_ui_component/state-management.md §usePayableReports` | Hook 単体テスト | API 呼び出し（フィルタ・ページネーション パラメータ）、クエリキー `['workflow', 'payable', params]`、staleTime 30秒 |

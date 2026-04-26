# 承認待ち一覧 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/workflow-pending.md）、共通コンポーネント詳細（common-components.md）、承認/却下のビジネスロジック詳細（SCR-RPT-004 で実行） |
| 主な参照元 | `50_detail_design/screens/workflow-pending.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-WFL-001` |
| 画面名 | 承認待ち一覧 |
| URL パス | `/approvals` |
| 画面詳細仕様 | `50_detail_design/screens/workflow-pending.md` |

---

## 2. コンポーネントツリー

```
PendingApprovalsPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── PendingApprovalsContent
        ├── PageTitle
        ├── PendingFilterBar
        │   ├── AppTextField（← common-components.md）[申請者名]
        │   └── FilterResetButton
        ├── PendingReportCount
        ├── PendingReportTable
        │   ├── AppDataGrid（← common-components.md）
        │   │   └── PendingReportRow（仮想行 -- DataGrid renderCell）
        │   │       └── SelfLabel（条件付き表示）
        │   ├── EmptyState（← common-components.md）[データ0件時]
        │   └── PageSkeleton（← common-components.md）[ローディング時]
        └── AppPaginationFooter（← common-components.md）
            ├── AppPagination（← common-components.md）
            └── PageSizeSelector（← common-components.md）
```

---

## 3. コンポーネント定義

### PendingApprovalsPage

- 配置: `pages/approvals/PendingApprovalsPage.tsx`
- 責務: 承認待ち一覧画面のページコンポーネント。AppLayout でラップし、PendingApprovalsContent を配置する。usePendingReports Hook を呼び出してデータを取得する。**フィルタ条件・ページ番号（`page`）・表示件数（`per_page`）はすべて URL クエリパラメータ（`useSearchParams`）で管理する**。フィルタ変更時および `per_page` 変更時は URL クエリパラメータを更新してページ番号を 1 にリセットする（`setSearchParams` は 1 回のコールに集約）。`per_page` の NaN/負数 URL 値は FE 側で 20 にフォールバックし、範囲内不正値（0 や 101 等）は BE バリデーション（HTTP 422）に委ねる。Approver ロール以外のアクセスはダッシュボードにリダイレクトされる（RBAC ミドルウェアが 403 を返し、グローバルエラーハンドラがリダイレクト）。詳細は `state-management.md §3.1`（ページネーション URL クエリ ⇔ Hook ⇔ API URL 連動仕様）を参照
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;1, &sect;12, &sect;13

```typescript
// Props なし（ページコンポーネント）
```

### PendingApprovalsContent

- 配置: `pages/approvals/PendingApprovalsContent.tsx`
- 責務: 承認待ち一覧のメインコンテンツ領域。ページタイトル、フィルタエリア、件数表示、テーブル、ページネーションを統合する。usePendingReports の結果に応じてローディング・空状態・テーブル表示を切り替える
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;4

```typescript
interface PendingApprovalsContentProps {
  /** 承認待ちレポート一覧データ */
  reports: PendingReport[];
  /** ページネーション情報 */
  pagination: Pagination; // ← api/types.ts の Pagination 型
  /** ローディング状態 */
  isLoading: boolean;
  /** エラー状態 */
  error: ApiClientError | null;
  /** 現在のフィルタ条件 */
  filters: PendingReportListParams;
  /** フィルタ変更コールバック */
  onFilterChange: (filters: PendingReportListParams) => void;
  /** ページ変更コールバック */
  onPageChange: (page: number) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reports` | `PendingReport[]` | Yes | 承認待ちレポート一覧 | usePendingReports Hook の data.data |
| `pagination` | `Pagination` | Yes | ページネーション情報 | usePendingReports Hook の data.pagination |
| `isLoading` | `boolean` | Yes | データ取得中フラグ | usePendingReports Hook の isLoading |
| `error` | `ApiClientError \| null` | Yes | API エラー | usePendingReports Hook の error |
| `filters` | `PendingReportListParams` | Yes | 現在のフィルタ条件 | PendingApprovalsPage が URL クエリパラメータ（useSearchParams）から復元 |
| `onFilterChange` | `(filters: PendingReportListParams) => void` | Yes | フィルタ変更時のコールバック | PendingApprovalsPage |
| `onPageChange` | `(page: number) => void` | Yes | ページ変更時のコールバック | PendingApprovalsPage |

### PendingFilterBar

- 配置: `pages/approvals/PendingFilterBar.tsx`
- 責務: 承認待ち一覧のフィルタエリア。申請者名テキスト入力（デバウンス 300ms）とフィルタリセットボタンを配置する。フィルタ変更時は onFilterChange コールバックで親に通知する
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;6

```typescript
interface PendingFilterBarProps {
  /** 現在のフィルタ条件 */
  filters: PendingReportListParams;
  /** フィルタ変更コールバック */
  onFilterChange: (filters: PendingReportListParams) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `filters` | `PendingReportListParams` | Yes | 現在のフィルタ条件 | PendingApprovalsContent |
| `onFilterChange` | `(filters: PendingReportListParams) => void` | Yes | フィルタ変更コールバック（デバウンス後に発火） | PendingApprovalsContent |

### PendingReportCount

- 配置: `pages/approvals/PendingReportCount.tsx`
- 責務: テーブル上部に承認待ちレポートの件数を表示する。0件かつフィルタ未適用時は空状態メッセージに切り替わるため非表示、0件かつフィルタ適用中は「条件に一致するレポートはありません。」を表示する
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;8

```typescript
interface PendingReportCountProps {
  /** 総件数 */
  totalCount: number;
  /** フィルタが適用中かどうか */
  isFiltered: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `totalCount` | `number` | Yes | 承認待ちレポートの総件数 | pagination.total_count |
| `isFiltered` | `boolean` | Yes | フィルタが適用中か | filters の値から判定 |

### PendingReportTable

- 配置: `pages/approvals/PendingReportTable.tsx`
- 責務: 承認待ちレポートの一覧テーブルを表示する。AppDataGrid をラップし、カラム定義（申請者名・タイトル・合計金額・提出日・遷移アイコン）を設定する。行クリックでレポート詳細画面（SCR-RPT-004）へ遷移する。自己レポート行には「自分」ラベルを表示する。ローディング中は PageSkeleton を、0件時は EmptyState を表示する
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;5, &sect;9, &sect;10, &sect;12

```typescript
interface PendingReportTableProps {
  /** 承認待ちレポート一覧 */
  reports: PendingReport[];
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
| `reports` | `PendingReport[]` | Yes | 承認待ちレポート一覧 | PendingApprovalsContent |
| `isLoading` | `boolean` | Yes | データ取得中フラグ | PendingApprovalsContent |
| `emptyMessage` | `string` | Yes | 空状態メッセージ | PendingApprovalsContent（フィルタ状態に応じて切替） |
| `onRowClick` | `(reportId: string) => void` | Yes | 行クリック時の遷移コールバック | PendingApprovalsContent（React Router navigate） |

### SelfLabel

- 配置: `components/ui/SelfLabel.tsx`
- 責務: 自分が作成したレポートであることを示す「自分」ラベルを Chip で表示する。申請者名の横に配置する。`is_own_report` フラグが true の場合のみ表示する
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;5（自己承認禁止の表示ルール）

Props 型定義は `common-components.md §SelfLabel` を参照。

### FilterResetButton

- 配置: `components/ui/FilterResetButton.tsx`
- 責務: フィルタ条件をリセットするボタン。フィルタが適用されている場合にのみ有効化される
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;6

Props 型定義は `common-components.md §FilterResetButton` を参照。

### PageTitle

- 配置: `components/ui/PageTitle.tsx`
- 責務: ページタイトル（「承認待ち一覧」）を表示する Typography コンポーネント
- 対応セクション: `50_detail_design/screens/workflow-pending.md` &sect;4

Props 型定義は `common-components.md §PageTitle` を参照。

---

## 4. データフロー

```
GET /api/workflow/pending
（URL クエリ ⇔ Hook 引数 ⇔ API URL の連動仕様は state-management.md §3.1 を参照）
  → usePendingReports(filters)（← state-management.md §3 データフェッチ系）
    → PendingApprovalsPage
      → PendingApprovalsContent (props: reports, pagination, isLoading, error, filters, onFilterChange, onPageChange)
        → PageTitle (props: title = "承認待ち一覧")
        → PendingFilterBar (props: filters, onFilterChange)
          → AppTextField (props: 申請者名入力)
          → FilterResetButton (props: onReset, isFiltered)
        → PendingReportCount (props: totalCount, isFiltered)
        → PendingReportTable (props: reports, isLoading, emptyMessage, onRowClick)
          → AppDataGrid (props: columns, rows, loading)
            → SelfLabel (props: isOwnReport -- renderCell 内)
          → EmptyState (props: message) [0件時]
          → PageSkeleton (props: variant = "table") [ローディング時]
        → AppPaginationFooter (props: currentPage, totalPages, onPageChange, perPage, onPerPageChange)
          → AppPagination (props: currentPage, totalPages, onPageChange) [内部]
          → PageSizeSelector (props: perPage, onPerPageChange) [内部]
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `usePendingReports` | 承認待ちレポート一覧の取得 | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | 現在のユーザー情報取得（AppHeader 表示用） | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| 承認待ちレポート一覧 | サーバー | usePendingReports Hook（TanStack Query useQuery） | PendingApprovalsPage |
| フィルタ条件（applicant_name） | UI | URL クエリパラメータ（useSearchParams） | PendingApprovalsPage |
| 現在のページ番号（page） | UI | URL クエリパラメータ（useSearchParams） | PendingApprovalsPage |
| 現在の表示件数（per_page） | UI | URL クエリパラメータ（useSearchParams）。NaN/負数は FE 側で 20 にフォールバック（`state-management.md §3.1` 参照） | PendingApprovalsPage |
| デバウンス中の入力値 | UI | useState + useEffect（300ms デバウンス） | PendingFilterBar 内 |
| サイドバー開閉状態 | UI | useState | AppLayout 内 |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | PendingApprovalsPage のルートラッパー | children に PendingApprovalsContent を配置 |
| `AppHeader`（← common-components.md） | AppLayout 内部 | 標準使用 |
| `AppSidebar`（← common-components.md） | AppLayout 内部 | 承認待ち一覧メニューをアクティブ表示（currentPath = "/approvals"） |
| `AppDataGrid`（← common-components.md） | PendingReportTable 内 | columns: 申請者名・タイトル・合計金額・提出日・遷移アイコンの5列。行クリックで遷移。emptyMessage は PendingReportTable が EmptyState で別途制御 |
| `AppTextField`（← common-components.md） | PendingFilterBar 内の申請者名フィルタ | name: "applicant_name"、label: "申請者名"。デバウンス 300ms で検索実行 |
| `AppPaginationFooter`（← common-components.md） | PendingApprovalsContent 下部のページネーションフッター（中央: ページ番号、右: 表示件数セレクタ）。常時表示（`totalPages <= 1` でも非表示にしない。内部 `AppPagination` は `count={Math.max(totalPages, 1)}`）。スマホ幅（375px）では `flex-direction: column` で縦並び | currentPage / totalPages / perPage を usePendingReports の pagination から取得。onPageChange / onPerPageChange は PendingApprovalsPage が `setSearchParams` で URL を更新（per_page 変更時は page=1 にリセット）。Props 型は `common-components.md §AppPaginationFooter / §PageSizeSelector` を参照 |
| `EmptyState`（← common-components.md） | PendingReportTable 内（0件時） | message: フィルタ未適用時「承認待ちのレポートはありません。」、フィルタ適用中「条件に一致するレポートはありません。」。action: フィルタ適用中はリセットボタン |
| `PageSkeleton`（← common-components.md） | PendingReportTable 内（ローディング時） | variant: "table"、rows: 5 |
| `AppToast`（← common-components.md） | PendingApprovalsPage | サーバーエラー（500）時のトースト表示 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `PendingApprovalsPage` | Approver ロールのみアクセス可能。他ロールは API が 403 を返し、ダッシュボード（SCR-DASH-001）にリダイレクト | - | `authz.md §6.6`（`/api/workflow/pending`: Approver のみ）、`screens/workflow-pending.md` &sect;1 |
| `SelfLabel` | `is_own_report` が true のレポート行でのみ表示 | - | `screens/workflow-pending.md` &sect;5（自己承認禁止の表示ルール） |
| `PendingReportTable` 行クリック | 常時有効（全行遷移可能） | 自己レポートも含めて SCR-RPT-004 へ遷移可能。承認・却下ボタンの表示制御は SCR-RPT-004 側で行う | `screens/workflow-pending.md` &sect;5, &sect;12 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | PendingApprovalsPage | URL パス、対応 API、アクセス可能ロール |
| &sect;3 アクションの責務分担 | - | 承認・却下は SCR-RPT-004 で実行。本画面は一覧表示とナビゲーションのみ |
| &sect;4 レイアウト | PendingApprovalsContent | ヘッダー + サイドナビ + メインコンテンツの構成 |
| &sect;5 表示項目（テーブルカラム） | PendingReportTable, AppDataGrid | 申請者名・タイトル・合計金額・提出日・遷移アイコンの5列 |
| &sect;5 自己承認禁止の表示ルール | SelfLabel | 「自分」ラベル表示。行クリックは通常通り遷移可能 |
| &sect;6 フィルタ | PendingFilterBar, AppTextField, FilterResetButton | 申請者名テキスト入力（デバウンス 300ms）+ リセットボタン |
| &sect;7 ページネーション | AppPaginationFooter（内部に AppPagination + PageSizeSelector） | オフセットベース、デフォルト 20 件/ページ、`per_page` セレクタ `[10, 20, 50, 100]`、URL クエリ `page` / `per_page` と双方向連動、常時表示。詳細は `state-management.md §3.1` 参照 |
| &sect;8 件数表示 | PendingReportCount | 「N 件の承認待ちレポート」/ フィルタ適用中0件メッセージ |
| &sect;9 空状態 | EmptyState | 「承認待ちのレポートはありません。」/「条件に一致するレポートはありません。」 |
| &sect;10 ローディング | PageSkeleton | variant: "table"、5行分のスケルトン UI |
| &sect;11 エラー表示 | AppToast, PendingApprovalsPage | サーバーエラー: トースト、401: ログインリダイレクト、403: ダッシュボードリダイレクト |
| &sect;12 行クリック時の遷移 | PendingReportTable | SCR-RPT-004 へ遷移。ブラウザ履歴に追加 |
| &sect;13 共通仕様 | PendingApprovalsPage | データ鮮度: 画面表示時に取得。自動リフレッシュなし（MVP） |
| &sect;15 API リクエスト/レスポンス | usePendingReports Hook | GET /api/workflow/pending |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/workflow-pending.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | コンポーネント単体テスト | Approver 以外のアクセス時リダイレクト、データ取得・フィルタ・ページネーション状態の管理 |
| `55_ui_component/screens/workflow-pending.md §PendingApprovalsContent` | コンポーネント単体テスト | ローディング・空状態・テーブル表示の切り替え、フィルタ変更時のページリセット |
| `55_ui_component/screens/workflow-pending.md §PendingFilterBar` | コンポーネント単体テスト | 申請者名入力のデバウンス（300ms）、フィルタリセット |
| `55_ui_component/screens/workflow-pending.md §PendingReportCount` | コンポーネント単体テスト | 件数テキスト表示（「N 件の承認待ちレポート」）、0件時の非表示、フィルタ適用中の0件メッセージ |
| `55_ui_component/screens/workflow-pending.md §PendingReportTable` | コンポーネント単体テスト | テーブルカラムの描画（申請者名・タイトル・合計金額・提出日・遷移アイコン）、金額フォーマット（3桁カンマ区切り）、日付フォーマット（YYYY/MM/DD）、行クリック遷移、自己レポートの「自分」ラベル表示 |
| `55_ui_component/screens/workflow-pending.md §SelfLabel` | コンポーネント単体テスト | isOwnReport が true の場合のみラベル表示、false の場合は非表示 |
| `55_ui_component/screens/workflow-pending.md §FilterResetButton` | コンポーネント単体テスト | フィルタ適用中は有効、未適用時は disabled |
| `55_ui_component/state-management.md §usePendingReports` | Hook 単体テスト | API 呼び出し（フィルタ・ページネーション パラメータ）、クエリキー `['workflow', 'pending', params]`、staleTime 30秒 |

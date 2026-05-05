# テナント全レポート一覧 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/admin-all-reports.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/admin-all-reports.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-ADM-001` |
| 画面名 | テナント全レポート一覧 |
| URL パス | `/reports/all` |
| 画面詳細仕様 | `50_detail_design/screens/admin-all-reports.md` |

---

## 2. コンポーネントツリー

```
AllReportsPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── PageTitle
        ├── AllReportsFilterBar
        │   ├── AppSelect（← common-components.md）[ステータス]
        │   ├── AppDatePicker（← common-components.md）[期間（開始日）]
        │   ├── AppDatePicker（← common-components.md）[期間（終了日）]
        │   └── AppSelect（← common-components.md）[申請者]
        ├── AllReportsTable
        │   ├── AppDataGrid（← common-components.md）
        │   │   └── StatusChip（← common-components.md）[ステータス列]
        │   ├── EmptyState（← common-components.md）
        │   └── PageSkeleton（← common-components.md）
        └── AppPaginationFooter（← common-components.md）
            ├── AppPagination（← common-components.md）
            └── PageSizeSelector（← common-components.md）
```

---

## 3. コンポーネント定義

### AllReportsPage

- 配置: `pages/admin/AllReportsPage.tsx`
- 責務: テナント全レポート一覧画面のページコンポーネント。AppLayout でラップし、useAllReports / useTenantMembers Hook を呼び出してデータを取得する。**フィルタ条件・ページ番号（`page`）・表示件数（`per_page`）はすべて URL クエリパラメータ（`useSearchParams`）で管理する**（旧 `useState` ベースから移行。他 3 画面と挙動を統一）。フィルタ変更時および `per_page` 変更時は URL クエリパラメータを更新してページ番号を 1 にリセットする（`setSearchParams` は 1 回のコールに集約）。`per_page` の NaN/負数 URL 値は FE 側で 20 にフォールバックし、範囲内不正値（0 や 101 等）は BE バリデーション（HTTP 422）に委ねる。403 エラー時はダッシュボードにリダイレクトする。詳細は `state-management.md §3.1`（ページネーション URL クエリ ⇔ Hook ⇔ API URL 連動仕様）を参照
- 対応セクション: `50_detail_design/screens/admin-all-reports.md` &sect;1, &sect;3, &sect;9, &sect;10

```typescript
// Props なし（ページコンポーネント）
```

### PageTitle

- 配置: `components/ui/PageTitle.tsx`
- 責務: ページタイトルを Typography で表示する
- 対応セクション: `50_detail_design/screens/admin-all-reports.md` &sect;5（ページタイトル）

Props 型定義は `common-components.md §PageTitle` を参照。

### AllReportsFilterBar

- 配置: `pages/admin/AllReportsFilterBar.tsx`
- 責務: テナント全レポート一覧のフィルタ条件（ステータス・期間・申請者）を管理する。フィルタ変更時に onFilterChange コールバックを呼び出し、親コンポーネントのフィルタ状態を更新する。開始日が終了日より後の場合はバリデーションエラーを表示する
- 対応セクション: `50_detail_design/screens/admin-all-reports.md` &sect;5（フィルタエリア）
- **フィルタエリア レイアウト規定**: フィルタコンテナは `<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2, mb: 2 }}>` で実装する。ステータス AppSelect には `sx={{ width: 140 }}`、開始日・終了日の AppDatePicker には `sx={{ width: 160 }}`、申請者 AppSelect には `sx={{ width: 200 }}` を指定する。全要素に `fullWidth={false}` を指定して AppSelect / AppDatePicker デフォルトの fullWidth=true を無効化する。**開始日・終了日は内側 `<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2 }}>` でラップしてセットとして扱う**（意味的グループ化 + 改行位置の制御）。`flexWrap: 'wrap'` により PC では横並び（過剰幅解消）、mobile では「ステータス / 開始日+終了日 / 申請者」のように意味的単位で折り返される。`mb: 2` で下のテーブルとの余白を確保する

```typescript
interface AllReportsFilterValues {
  /** ステータスフィルタ値（空文字は「全て」） */
  status: string;
  /** 期間（開始日）。YYYY-MM-DD 形式の文字列。空文字 `""` は未指定（AppDatePicker の value/onChange が `string` に統一されたため、null は使用しない） */
  from: string;
  /** 期間（終了日）。YYYY-MM-DD 形式の文字列。空文字 `""` は未指定 */
  to: string;
  /** 申請者フィルタ値（空文字は「全て」） */
  submitterId: string;
}

interface AllReportsFilterBarProps {
  /** 現在のフィルタ値 */
  filters: AllReportsFilterValues;
  /** フィルタ変更時のコールバック */
  onFilterChange: (filters: AllReportsFilterValues) => void;
  /** 申請者セレクトボックスの選択肢一覧 */
  members: UserSummary[];
  /** メンバー一覧のローディング状態 */
  membersLoading: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `filters` | `AllReportsFilterValues` | Yes | 現在のフィルタ条件 | AllReportsPage が URL クエリパラメータ（useSearchParams）から復元 |
| `onFilterChange` | `(filters: AllReportsFilterValues) => void` | Yes | フィルタ変更コールバック | AllReportsPage |
| `members` | `UserSummary[]` | Yes | テナント内メンバー一覧（申請者フィルタ用） | useTenantMembers Hook の data |
| `membersLoading` | `boolean` | Yes | メンバー一覧のローディング状態 | useTenantMembers Hook の isLoading |

### AllReportsTable

- 配置: `pages/admin/AllReportsTable.tsx`
- 責務: テナント全レポートのテーブル表示を管理する。データが存在する場合は AppDataGrid で一覧を描画し、0件の場合は EmptyState を表示する。ローディング中は PageSkeleton を表示する。行クリックでレポート詳細画面に遷移する
- 対応セクション: `50_detail_design/screens/admin-all-reports.md` &sect;5（レポートテーブル）, &sect;7, &sect;8

```typescript
interface AllReportRow {
  /** レポート ID */
  id: string;
  /** レポートタイトル */
  title: string;
  /** 申請者情報 */
  submitter: UserSummary;
  /** 合計金額（円） */
  totalAmount: number;
  /** レポートステータス */
  status: ReportStatus;
  /** 提出日（ISO 8601 形式、または null） */
  submittedAt: string | null;
  /** 作成日（ISO 8601 形式） */
  createdAt: string;
}

interface AllReportsTableProps {
  /** テーブルに表示するレポート行データ */
  reports: AllReportRow[];
  /** ローディング状態 */
  loading: boolean;
  /** フィルタが適用されているか（空状態メッセージの切り替えに使用） */
  hasActiveFilters: boolean;
  /** 行クリック時のコールバック（レポート詳細画面への遷移） */
  onRowClick: (reportId: string) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reports` | `AllReportRow[]` | Yes | レポート一覧データ | useAllReports Hook の data.data |
| `loading` | `boolean` | Yes | ローディング状態 | useAllReports Hook の isLoading |
| `hasActiveFilters` | `boolean` | Yes | フィルタ適用有無（空状態メッセージ切替） | AllReportsPage のフィルタ状態から算出 |
| `onRowClick` | `(reportId: string) => void` | Yes | 行クリックコールバック | AllReportsPage（navigate 呼び出し） |

---

## 4. データフロー

```
GET /api/reports/all?page=...&per_page=...&status=...&from=...&to=...&submitter_id=...
（URL クエリ ⇔ Hook 引数 ⇔ API URL の連動仕様は state-management.md §3.1 を参照）
  → useAllReports(params)（← state-management.md §3 データフェッチ系）
    → AllReportsPage
      → AllReportsFilterBar (props: filters, onFilterChange, members, membersLoading)
        → AppSelect x 2 (props: ステータス / 申請者)
        → AppDatePicker x 2 (props: 開始日 / 終了日)
      → AllReportsTable (props: reports, loading, hasActiveFilters, onRowClick)
        → AppDataGrid (props: columns, rows, loading, emptyMessage)
          → StatusChip (props: status)
        → EmptyState (props: message)
        → PageSkeleton (props: variant="table")
      → AppPaginationFooter (props: currentPage, totalPages, onPageChange, perPage, onPerPageChange, totalCount)
        → AppPagination (props: currentPage, totalPages, onPageChange) [内部]
        → PageSizeSelector (props: perPage, onPerPageChange) [内部]

GET /api/tenant/members
  → useTenantMembers()（← state-management.md §3 データフェッチ系）
    → AllReportsPage
      → AllReportsFilterBar (props: members, membersLoading)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useAllReports` | テナント全レポート一覧のデータフェッチ（フィルタ・ページネーション付き） | `state-management.md §3 データフェッチ系` |
| `useTenantMembers` | 申請者フィルタ用テナント内メンバー一覧のデータフェッチ | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | ヘッダーのユーザー情報表示・ロール確認 | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| フィルタ条件（status, from, to, submitterId） | UI | URL クエリパラメータ（useSearchParams）。旧 `useState` から移行 | AllReportsPage |
| 現在のページ番号（page） | UI | URL クエリパラメータ（useSearchParams）。旧 `useState` から移行 | AllReportsPage |
| 現在の表示件数（per_page） | UI | URL クエリパラメータ（useSearchParams）。NaN/負数は FE 側で 20 にフォールバック（`state-management.md §3.1` 参照） | AllReportsPage |
| レポート一覧データ | サーバー | useAllReports Hook（TanStack Query useQuery） | AllReportsPage |
| テナントメンバー一覧 | サーバー | useTenantMembers Hook（TanStack Query useQuery） | AllReportsPage |
| 日付バリデーションエラー | UI | useState（string | null）またはローカル算出 | AllReportsFilterBar |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | AllReportsPage のルートラッパー | children に PageTitle + AllReportsFilterBar + AllReportsTable + AppPaginationFooter を配置 |
| `AppSelect`（← common-components.md） | AllReportsFilterBar 内のステータスフィルタ | options: 全て / 下書き / 提出済み / 承認済み / 却下 / 支払済み。value: filters.status |
| `AppSelect`（← common-components.md） | AllReportsFilterBar 内の申請者フィルタ | options: 全て + useTenantMembers のメンバー一覧。value: filters.submitterId |
| `AppDatePicker`（← common-components.md） | AllReportsFilterBar 内の期間フィルタ（開始日・終了日） | 開始日: value=filters.from、終了日: value=filters.to。終了日に日付バリデーションエラーを表示 |
| `AppDataGrid`（← common-components.md） | AllReportsTable 内のレポートテーブル | columns: 申請者名・タイトル・合計金額・ステータス・提出日。行クリック対応（`cursor: pointer` + hover 効果で押下可能性を表現）。**行末 ChevronRight アイコンは表示しない**（issue #155、4 画面共通方針: レポート一覧 / 全レポート / 承認待ち / 支払待ち の DataGrid で行末アイコン列を持たず、視覚表現を統一する） |
| `StatusChip`（← common-components.md） | AllReportsTable の AppDataGrid ステータス列 | status: レポートの現在ステータス |
| `EmptyState`（← common-components.md） | AllReportsTable 内のデータ0件表示 | フィルタ有無でメッセージ切替（「レポートはまだ作成されていません。」/「条件に一致するレポートはありません。フィルタを変更してお試しください。」） |
| `PageSkeleton`（← common-components.md） | AllReportsTable 内のローディング表示 | variant="table"。初回読み込み・フィルタ変更・ページ切替時に表示 |
| `AppPaginationFooter`（← common-components.md） | AllReportsPage の下部のページネーションフッター（左: 件数表示「{start} - {end} / 全 {total} 件」、中央: ページ番号、右: 表示件数セレクタ）。常時表示（`totalPages <= 1` でも非表示にしない。内部 `AppPagination` は `count={Math.max(totalPages, 1)}`）。スマホ幅（375px）では `flex-direction: column` で縦並び（順序: 件数表示 → AppPagination → PageSizeSelector） | currentPage / totalPages / perPage / totalCount は useAllReports のレスポンス pagination から取得（`totalCount={pagination?.total_count}` で渡す）。onPageChange / onPerPageChange は AllReportsPage が `setSearchParams` で URL を更新（per_page 変更時は page=1 にリセット）。件数表示の算出（start/end）は AppPaginationFooter 内部で行う。Props 型は `common-components.md §AppPaginationFooter / §PageSizeSelector` を参照 |
| `AppToast`（← common-components.md） | AllReportsPage（SnackbarContext 経由） | API 通信エラー（500 系）のトースト表示 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `AllReportsPage` | Admin または Accounting ロールのみアクセス可能。他ロールはダッシュボードにリダイレクト | - | `authz.md §6.3`（`/api/reports/all`: Admin, Accounting）/ `screens/admin-all-reports.md §3` |
| `AppSidebar`（「全レポート」メニュー） | Admin または Accounting ロールのみ表示 | - | `screens/admin-all-reports.md §10` |
| `AllReportsFilterBar` | 常時表示（Admin / Accounting で差異なし） | 全フィルタ利用可能 | `screens/admin-all-reports.md §11` |
| `AllReportsTable` | 常時表示（Admin / Accounting で表示内容に差異なし） | 行クリックでレポート詳細画面に遷移。遷移先の操作可否はロールに依存（レポート詳細画面側で制御） | `screens/admin-all-reports.md §10, §11` |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | AllReportsPage | URL パス、対応 API、対応ロール |
| &sect;3 アクセス制御 | AllReportsPage | Admin / Accounting のみ。他ロールはリダイレクト |
| &sect;4 レイアウト | AppLayout, PageTitle, AllReportsFilterBar, AllReportsTable, AppPaginationFooter | ヘッダー + サイドナビ + メインコンテンツの標準レイアウト |
| &sect;5 表示項目（ページタイトル） | PageTitle | 「全レポート」 |
| &sect;5 表示項目（フィルタエリア） | AllReportsFilterBar | ステータス・期間・申請者の 4 フィルタ |
| &sect;5 表示項目（レポートテーブル） | AllReportsTable, AppDataGrid, StatusChip | 申請者名・タイトル・合計金額・ステータス・提出日の 5 カラム |
| &sect;6 ページネーション | AppPaginationFooter（内部に AppPagination + PageSizeSelector） | オフセットベース、デフォルト 20 件/ページ、`per_page` セレクタ `[10, 20, 50, 100]`、URL クエリ `page` / `per_page` と双方向連動、常時表示。詳細は `state-management.md §3.1` 参照 |
| &sect;7 空状態 | EmptyState | フィルタ有無でメッセージ切替 |
| &sect;8 ローディング | PageSkeleton | variant="table"。初回・フィルタ変更・ページ切替時 |
| &sect;9 エラー表示 | AllReportsFilterBar（フィールドレベル）, AppToast（トースト） | 日付バリデーション: フィールドレベル、500 系: トースト、403: リダイレクト、401: ログイン画面リダイレクト |
| &sect;10 画面遷移 | AllReportsTable（行クリック） | レポート詳細画面（SCR-RPT-004）に遷移 |
| &sect;11 ロール別表示差異 | AllReportsPage | Admin / Accounting で画面構成・表示内容に差異なし |
| &sect;12 API リクエスト/レスポンス | useAllReports, useTenantMembers | GET /api/reports/all, GET /api/tenant/members |
| &sect;13 処理シーケンス | AllReportsPage（Hook 呼び出し） | JWT 検証 + RBAC + RLS は API 側で処理 |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/admin-all-reports.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/admin-all-reports.md §AllReportsPage` | コンポーネント単体テスト | Admin / Accounting でのアクセス可否、他ロールのリダイレクト、フィルタ変更時の page リセット、行クリックによるレポート詳細遷移 |
| `55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar` | コンポーネント単体テスト | ステータス・期間・申請者フィルタの操作、開始日 > 終了日のバリデーションエラー表示、フィルタ変更時の onFilterChange 呼び出し |
| `55_ui_component/screens/admin-all-reports.md §AllReportsTable` | コンポーネント単体テスト | レポート一覧の描画（申請者名・タイトル・金額・ステータス・提出日）、ローディング時の PageSkeleton 表示、データ 0 件時の EmptyState 表示（フィルタ有無でメッセージ切替）、行クリックコールバック |
| `55_ui_component/screens/admin-all-reports.md §PageTitle` | コンポーネント単体テスト | タイトルテキストの描画 |
| `55_ui_component/state-management.md §useAllReports` | Hook 単体テスト | API 呼び出し・フィルタパラメータ送信・ページネーション・レスポンスのデータ変換 |
| `55_ui_component/state-management.md §useTenantMembers` | Hook 単体テスト | テナント内メンバー一覧の取得・キャッシュ |

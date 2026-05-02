# ダッシュボード -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/dashboard.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/dashboard.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-DASH-001` |
| 画面名 | ダッシュボード |
| URL パス | `/dashboard` |
| 画面詳細仕様 | `50_detail_design/screens/dashboard.md` |

---

## 2. コンポーネントツリー

```
DashboardPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    └── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── PageSkeleton variant="card"（← common-components.md、ローディング中のみ）
        ├── AppToast（← common-components.md、エラー発生時のみ）
        │
        ├── [Member / Approver / Accounting の場合]
        │   └── MyReportCountCards
        │       ├── CountCard（下書き）
        │       ├── CountCard（提出中）
        │       └── CountCard（却下）
        │
        ├── [Approver の場合]
        │   └── CountCard（承認待ち）
        │
        ├── [Accounting の場合]
        │   └── CountCard（支払待ち）
        │
        ├── [Admin の場合]
        │   ├── TenantStatusCards
        │   │   ├── CountCard（下書き）
        │   │   ├── CountCard（提出済み）
        │   │   ├── CountCard（承認済み）
        │   │   ├── CountCard（却下）
        │   │   └── CountCard（支払済み）
        │   └── CountCard（メンバー数）
        │
        ├── [Approver / Accounting / Admin の場合]
        │   └── MonthlySummaryTable
        │       ├── SectionHeading（Typography variant="h6" 「月別支出サマリー」）
        │       └── Table
        │           ├── TableHead（列見出し: 年月 / 合計金額）
        │           └── TableBody
        │
        └── [Member / Approver / Accounting の場合]
            └── RecentReportList
                ├── SectionHeading（Typography variant="h6" 「最近のレポート」）
                ├── Table
                │   ├── TableHead（列見出し: タイトル / 期間 / 金額 / ステータス）
                │   └── TableBody
                │       └── RecentReportRow（0〜5件）
                ├── EmptyState（← common-components.md、0件の場合）
                └── ViewAllLink
```

---

## 3. コンポーネント定義

### DashboardPage

- 配置: `pages/dashboard/DashboardPage.tsx`
- 責務: ダッシュボード画面のルートコンポーネント。`useDashboard` Hook でデータを取得し、`useCurrentUser` でロールを判定して、ロール別にセクションを出し分ける。ローディング中は PageSkeleton を表示し、エラー時は AppToast でメッセージを表示する
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;1〜&sect;9 全体
- **Admin 分岐の「メンバー数」カード**: Admin 分岐の `CountCard`（メンバー数）は `<Grid container spacing={2} data-testid="admin-member-count-cards">` + `<Grid size={{ xs: 12, sm: 4 }}>` でラップする。これにより MyReportCountCards（他ロール）と同じ視覚的幅基準（PC 幅で 1/3 幅相当）を共有する。ラップは `<Box sx={{ mt: 2 }}>` で余白を付与して TenantStatusCards と視覚的に分離する

```typescript
// Props なし（ページコンポーネント）
// 内部で useDashboard, useCurrentUser を呼び出す
```

---

### CountCard

- 配置: `components/dashboard/CountCard.tsx`
- 責務: 件数を大きなフォントで表示するカードコンポーネント。ラベル・件数・リンク先を受け取り、カードクリックで指定画面に遷移する。件数が 0 でも表示する
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.1, &sect;4.2, &sect;4.3, &sect;4.4, &sect;4.5

```typescript
interface CountCardProps {
  /** カードのラベル（例: 「下書き」「承認待ち」） */
  label: string;
  /** 表示する件数 */
  count: number;
  /** カードクリック時の遷移先パス。未指定の場合はクリック不可 */
  href?: string;
  /** アクセントカラー（Admin のステータス別件数カードで使用） */
  accentColor?: 'default' | 'info' | 'success' | 'error' | 'secondary';
  /** 件数の単位（デフォルト: 「件」。メンバー数カードでは「人」） */
  unit?: string;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `label` | `string` | Yes | カードのラベルテキスト | 固定文字列 |
| `count` | `number` | Yes | 表示する件数 | `useDashboard` レスポンスの各カウントフィールド |
| `href` | `string` | No | クリック時の遷移先パス | 固定パス + クエリパラメータ |
| `accentColor` | `'default' \| 'info' \| 'success' \| 'error' \| 'secondary'` | No | アクセントカラー | 固定値（ステータス色マッピング） |
| `unit` | `string` | No | 件数の単位（デフォルト:「件」） | 固定文字列 |

---

### MyReportCountCards

- 配置: `components/dashboard/MyReportCountCards.tsx`
- 責務: 自分のレポートのステータス別件数カード群（下書き・提出中・却下）をグリッドで配置する。Member / Approver / Accounting ロールで表示する
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.1

```typescript
interface MyReportCountCardsProps {
  /** 下書きレポート件数 */
  draftCount: number;
  /** 提出中レポート件数 */
  submittedCount: number;
  /** 却下レポート件数 */
  rejectedCount: number;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `draftCount` | `number` | Yes | 下書きレポート件数 | `useDashboard` -> `my_draft_count` |
| `submittedCount` | `number` | Yes | 提出中レポート件数 | `useDashboard` -> `my_submitted_count` |
| `rejectedCount` | `number` | Yes | 却下レポート件数 | `useDashboard` -> `my_rejected_count` |

---

### TenantStatusCards

- 配置: `components/dashboard/TenantStatusCards.tsx`
- 責務: テナント全体のレポートをステータス別に集計したカード群（下書き・提出済み・承認済み・却下・支払済み）をグリッドで配置する。Admin ロールのみで表示する。各カードにステータスバッジ色をアクセントとして使用する
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.4
- **Grid 配置**: 5 枚全てに `size={{ xs: 12, sm: 6, md: 4 }}` を適用する。PC 幅（md ≥ 900px）では 3 等分（`md: 4`）となり、5 カードのため **3 + 2 の 2 行レイアウト**になる（他ロール MyReportCountCards の `sm: 4` 基準と整合させるため `md: 2.4` → `md: 4` に変更）。タブレット幅（sm 600〜899px）では 2 列折返し、モバイル（xs < 600px）では縦積みとなる。`md: 'auto'` のような自然幅指定は使用しない（コンテンツ依存で幅が縮むため）。Grid container には `data-testid="tenant-status-cards"` を付与する
- **責務分離**: カード幅指定はコンテナ Grid 側（TenantStatusCards）で制御する。CountCard 共通基盤には幅指定を入れない

```typescript
interface TenantStatusCardsProps {
  /** テナント全体の下書き件数 */
  draftCount: number;
  /** テナント全体の提出済み件数 */
  submittedCount: number;
  /** テナント全体の承認済み件数 */
  approvedCount: number;
  /** テナント全体の却下件数 */
  rejectedCount: number;
  /** テナント全体の支払済み件数 */
  paidCount: number;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `draftCount` | `number` | Yes | テナント全体の下書き件数 | `useDashboard` -> `tenant_draft_count` |
| `submittedCount` | `number` | Yes | テナント全体の提出済み件数 | `useDashboard` -> `tenant_submitted_count` |
| `approvedCount` | `number` | Yes | テナント全体の承認済み件数 | `useDashboard` -> `tenant_approved_count` |
| `rejectedCount` | `number` | Yes | テナント全体の却下件数 | `useDashboard` -> `tenant_rejected_count` |
| `paidCount` | `number` | Yes | テナント全体の支払済み件数 | `useDashboard` -> `tenant_paid_count` |

---

### MonthlySummaryTable

- 配置: `components/dashboard/MonthlySummaryTable.tsx`
- 責務: テナント全体の直近 3 ヶ月の月別合計支出金額をテーブル形式で表示する。最新月が上（降順）。金額は `¥` プレフィックス + 3桁カンマ区切りで表示する。データが存在しない月は ¥0 として表示する。Approver / Accounting / Admin ロールで表示する
- ルート要素は `<Box>` で、上部にセクション見出し `<Typography variant="h6">月別支出サマリー</Typography>`（`sx={{ mb: 1 }}`）を配置し、その下に `<TableContainer>` を置く。0 件時もセクション見出しは表示する（0 件時は見出しの下に「データがありません」メッセージを表示）。
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.6

```typescript
interface MonthlySummaryItem {
  /** 年月（YYYY-MM 形式） */
  yearMonth: string;
  /** 月別合計金額（円） */
  totalAmount: number;
}

interface MonthlySummaryTableProps {
  /** 月別支出サマリーデータ（最大3件、降順） */
  items: MonthlySummaryItem[];
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `items` | `MonthlySummaryItem[]` | Yes | 月別支出サマリーデータ | `useDashboard` -> `monthly_summary` |

---

### RecentReportList

- 配置: `components/dashboard/RecentReportList.tsx`
- 責務: 自分が作成した直近 5 件のレポートを一覧表示する。レポートが 0 件の場合は EmptyState を表示する。一覧の下部に「すべてのレポートを見る」リンクを配置する。Member / Approver / Accounting ロールで表示する
- ルート要素は `<Box>` で、上部にセクション見出し `<Typography variant="h6">最近のレポート</Typography>`（`sx={{ mb: 1 }}`）を配置する。`<Table>` には `<TableHead>` を含み、列見出し（タイトル / 期間 / 金額 / ステータス）を表示する。金額列は右寄せ（`align="right"`）とし、`RecentReportRow` 側の `<TableCell>` の align と整合させる。0 件時もセクション見出しは表示する（見出しの下に EmptyState を表示）。
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.7

```typescript
interface RecentReport {
  /** レポートID（UUID） */
  id: string;
  /** レポートタイトル */
  title: string;
  /** 対象期間開始日（YYYY-MM-DD） */
  periodStart: string;
  /** 対象期間終了日（YYYY-MM-DD） */
  periodEnd: string;
  /** 合計金額（円） */
  totalAmount: number;
  /** ステータス */
  status: 'draft' | 'submitted' | 'approved' | 'rejected' | 'paid';
}

interface RecentReportListProps {
  /** 直近レポート一覧（最大5件） */
  reports: RecentReport[];
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reports` | `RecentReport[]` | Yes | 直近レポートデータ（最大5件） | `useDashboard` -> `recent_reports` |

---

### RecentReportRow

- 配置: `components/dashboard/RecentReportRow.tsx`（RecentReportList 内部で使用）
- 責務: 直近レポート一覧の 1 行分を表示する。タイトル（リンク付き）、対象期間、合計金額、ステータスバッジを横に並べる。タイトルクリックで SCR-RPT-004（レポート詳細）に遷移する
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.7

```typescript
interface RecentReportRowProps {
  /** レポートID（遷移先 URL 生成用） */
  id: string;
  /** レポートタイトル */
  title: string;
  /** 対象期間開始日（YYYY-MM-DD） */
  periodStart: string;
  /** 対象期間終了日（YYYY-MM-DD） */
  periodEnd: string;
  /** 合計金額（円） */
  totalAmount: number;
  /** ステータス */
  status: 'draft' | 'submitted' | 'approved' | 'rejected' | 'paid';
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `id` | `string` | Yes | レポートID | `RecentReportListProps.reports[n].id` |
| `title` | `string` | Yes | レポートタイトル | `RecentReportListProps.reports[n].title` |
| `periodStart` | `string` | Yes | 対象期間開始日 | `RecentReportListProps.reports[n].periodStart` |
| `periodEnd` | `string` | Yes | 対象期間終了日 | `RecentReportListProps.reports[n].periodEnd` |
| `totalAmount` | `number` | Yes | 合計金額 | `RecentReportListProps.reports[n].totalAmount` |
| `status` | `ReportStatus` | Yes | ステータス | `RecentReportListProps.reports[n].status` |

---

### ViewAllLink

- 配置: RecentReportList 内にインラインで実装
- 責務: 「すべてのレポートを見る」リンクを表示し、SCR-RPT-001（レポート一覧）に遷移する
- 対応セクション: `50_detail_design/screens/dashboard.md` &sect;4.7

```typescript
// RecentReportList 内で MUI Link + react-router-dom Link を使用してインライン実装
// 遷移先: /reports
// Props 不要（固定リンク）
```

---

## 4. データフロー

```
GET /api/dashboard
  → useDashboard()（← state-management.md §3）
    → DashboardPage
      → MyReportCountCards (props: draftCount, submittedCount, rejectedCount)
      │   └── CountCard x3 (props: label, count, href)
      → CountCard [承認待ち] (props: label, count, href)  ※Approver
      → CountCard [支払待ち] (props: label, count, href)  ※Accounting
      → TenantStatusCards (props: draftCount〜paidCount)              ※Admin
      │   └── CountCard x5 (props: label, count, href, accentColor)
      → CountCard [メンバー数] (props: label, count, unit)            ※Admin
      → MonthlySummaryTable (props: items)
      → RecentReportList (props: reports)
          └── RecentReportRow (props: id, title, periodStart, periodEnd, totalAmount, status)
              └── StatusChip (props: status)                          ← common-components.md

GET /api/auth/me
  → useCurrentUser()（← state-management.md §3）
    → DashboardPage（ロール判定に使用）
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useDashboard` | ダッシュボードデータのフェッチ（GET /api/dashboard） | `state-management.md` &sect;3 データフェッチ系 |
| `useCurrentUser` | ログインユーザー情報の取得（ロール判定に使用） | `state-management.md` &sect;3 認証系 |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| ダッシュボードデータ | サーバー状態 | `useDashboard`（TanStack Query、staleTime: 60秒） | 画面全体 |
| ユーザー情報（ロール判定） | サーバー状態 | `useCurrentUser`（TanStack Query、staleTime: 5分） | 画面全体（AppLayout 経由でも参照） |
| ローディング状態 | サーバー状態 | `useDashboard` の `isLoading` / `isPending` | 画面全体 |
| エラー状態 | サーバー状態 | `useDashboard` の `error` | 画面全体 |
| トースト表示状態 | UI 状態 | SnackbarContext（`showSnackbar`） | 画面全体 |

> ダッシュボード画面にはフォーム入力がないため、フォーム状態（React Hook Form + Zod）は不使用。
> ミューテーション系 Hook も不使用（読み取り専用画面）。

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md &sect;1） | DashboardPage のルートレイアウト | カスタマイズなし（`children` にダッシュボードコンテンツを渡す） |
| `AppToast`（← common-components.md &sect;3） | DashboardPage 内でエラー通知 | `severity="error"`, `autoHideDuration=null`（エラーは手動閉じ）。サーバーエラー / 認可エラーのメッセージ表示に使用 |
| `StatusChip`（← common-components.md &sect;3） | RecentReportRow 内のステータス表示 | カスタマイズなし（`status` Props をそのまま渡す） |
| `EmptyState`（← common-components.md &sect;3） | RecentReportList 内（レポート 0 件時） | `message="経費レポートはまだありません。レポートを作成して経費精算を始めましょう。"` |
| `PageSkeleton`（← common-components.md &sect;3） | DashboardPage 内（ローディング中） | `variant="card"`（カウントカードとテーブルのスケルトン表示） |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `DashboardPage` | 認証済みユーザー全ロール | 閲覧のみ（ミューテーションなし） | `authz.md` &sect;6.2: GET /api/dashboard は Member, Approver, Admin, Accounting 許可 |
| `MyReportCountCards` | `role` が `member`, `approver`, `accounting` のいずれか | カードクリックで SCR-RPT-001 に遷移可能 | `screens/dashboard.md` &sect;3.1, &sect;4.1 |
| `CountCard`（承認待ち） | `role === 'approver'` | カードクリックで SCR-WFL-001 に遷移可能 | `screens/dashboard.md` &sect;3.1, &sect;4.2 |
| `CountCard`（支払待ち） | `role === 'accounting'` | カードクリックで SCR-WFL-002 に遷移可能 | `screens/dashboard.md` &sect;3.1, &sect;4.3 |
| `TenantStatusCards` | `role === 'admin'` | カードクリックで SCR-ADM-001 に遷移可能 | `screens/dashboard.md` &sect;3.1, &sect;4.4 |
| `CountCard`（メンバー数） | `role === 'admin'` | クリック不可（MVP ではリンクなし） | `screens/dashboard.md` &sect;4.5 |
| `MonthlySummaryTable` | `role` が `approver`, `accounting`, `admin` のいずれか | 閲覧のみ | `screens/dashboard.md` &sect;3.1, &sect;4.6 |
| `RecentReportList` | `role` が `member`, `approver`, `accounting` のいずれか | タイトルクリックで SCR-RPT-004 に遷移可能。「すべてのレポートを見る」で SCR-RPT-001 に遷移可能 | `screens/dashboard.md` &sect;3.1, &sect;4.7 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 概要 | `DashboardPage` | ページコンポーネント全体 |
| &sect;2 レイアウト構成 | `DashboardPage` + `AppLayout` | 共通レイアウト（ヘッダー + サイドナビ + メインコンテンツ） |
| &sect;3 ロール別表示エリア | `DashboardPage`（条件分岐ロジック） | `useCurrentUser` のロール値で分岐 |
| &sect;4.1 カウントカード（自分のレポート） | `MyReportCountCards` -> `CountCard` x3 | Member / Approver / Accounting 向け |
| &sect;4.2 承認待ちカウントカード | `CountCard` | Approver のみ |
| &sect;4.3 支払待ちカウントカード | `CountCard` | Accounting のみ |
| &sect;4.4 ステータス別件数カード | `TenantStatusCards` -> `CountCard` x5 | Admin のみ。accentColor でステータス色を表現 |
| &sect;4.5 メンバー数カード | `CountCard`（unit="人", href なし） | Admin のみ。MVP ではクリック不可 |
| &sect;4.6 月別支出サマリー | `MonthlySummaryTable` | Approver / Accounting / Admin 向け。テーブル形式 |
| &sect;4.7 直近レポート一覧 | `RecentReportList` -> `RecentReportRow` + `EmptyState` + ViewAllLink | Member / Approver / Accounting 向け。最大5件、ページネーションなし |
| &sect;5 API レスポンスマッピング | `useDashboard` -> 各コンポーネント Props | DashboardResponse 型 -> Props 型へのマッピングは DashboardPage 内で実施 |
| &sect;6 画面遷移 | `CountCard.href`, `RecentReportRow`（Link）, ViewAllLink | react-router-dom の Link / useNavigate で遷移 |
| &sect;7.1 ローディング | `PageSkeleton variant="card"` | useDashboard の isLoading で表示制御 |
| &sect;7.2 エラー表示 | `AppToast` | 500: トーストエラー、401: リダイレクト、403: トーストエラー |
| &sect;7.3 キャッシュ | `useDashboard`（staleTime: 60秒） | state-management.md クエリキー設計準拠 |
| &sect;9 ロール別画面構成サマリ | `DashboardPage`（条件分岐全体） | Member / Approver / Accounting / Admin の4パターン |

---

## 9. テスト追跡用 設計識別子

テストケース（`60_test/test_cases/*.md`）からこの設計文書を一意に参照するための識別子規約。

**コンポーネント参照形式**: `55_ui_component/screens/dashboard.md §{ComponentName}`

**Hook 参照形式**: `55_ui_component/state-management.md §{useHookName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/dashboard.md §DashboardPage` | コンポーネント単体テスト | ロール別セクション表示分岐の検証（Member / Approver / Accounting / Admin の4パターン）。Admin 表示時の Grid 配置検証（tenant-status-cards 配下の 5 リンク枚数・admin-member-count-cards 配下のメンバー数カード存在）を含む |
| `55_ui_component/screens/dashboard.md §CountCard` | コンポーネント単体テスト | label / count / href / accentColor / unit の各 Props 組み合わせ描画検証 |
| `55_ui_component/screens/dashboard.md §MyReportCountCards` | コンポーネント単体テスト | 3枚のカウントカード描画、件数 0 のケース、カードクリック遷移先の検証 |
| `55_ui_component/screens/dashboard.md §TenantStatusCards` | コンポーネント単体テスト | 5枚のステータスカード描画、アクセントカラーの適用、カードクリック遷移先の検証。Admin 表示時の Grid 配置検証（3 等分 md:4 で 3+2 の 2 行レイアウト、Grid container の data-testid="tenant-status-cards" 存在）を含む |
| `55_ui_component/screens/dashboard.md §MonthlySummaryTable` | コンポーネント単体テスト | 3ヶ月分データ表示、金額フォーマット（¥ + カンマ区切り）、年月表示形式（YYYY年M月）、降順ソートの検証、セクション見出し「月別支出サマリー」の描画（0 件時も見出し表示） |
| `55_ui_component/screens/dashboard.md §RecentReportList` | コンポーネント単体テスト | 5件表示、0件時の EmptyState 表示、「すべてのレポートを見る」リンクの検証、セクション見出し「最近のレポート」の描画（0 件時も見出し表示）、列見出し（タイトル / 期間 / 金額 / ステータス）の描画 |
| `55_ui_component/screens/dashboard.md §RecentReportRow` | コンポーネント単体テスト | タイトルリンク遷移先、対象期間フォーマット、金額フォーマット、StatusChip 描画の検証 |
| `55_ui_component/state-management.md §useDashboard` | Hook 単体テスト | API 呼び出し・キャッシュ（staleTime: 60秒）・エラーハンドリングの検証 |
| `55_ui_component/state-management.md §useCurrentUser` | Hook 単体テスト | ユーザー情報取得・ロール判定の検証 |

# 092: 管理系2画面のコンポーネントツリーが AppLayout の責務と矛盾している

## 指摘概要
`admin-all-reports.md` と `admin-tenant.md` のコンポーネントツリーでは、`AppLayout` とメインコンテンツ要素が兄弟として記載されている。一方で `AppLayout` の正本定義は「AppHeader + AppSidebar + メインコンテンツ領域を提供し、children を描画する」責務であり、管理系2画面のツリー記法はその契約と矛盾している。実装者がツリーを正と見ると、`PageTitle` 以降を `AppLayout` の外に出す誤実装を誘発する。

## 根拠
- `55_ui_component/screens/admin-all-reports.md:29-44` では `AllReportsPage` 配下で `AppLayout` と `PageTitle` / `AllReportsFilterBar` / `AllReportsTable` / `AppPagination` が兄弟になっている。
- `55_ui_component/screens/admin-tenant.md:29-37` では `TenantPage` 配下で `AppLayout` と `PageTitle` / `TenantInfoCard` / `PhaseNotice` が兄弟になっている。
- `55_ui_component/common-components.md:17-27` では `AppLayout` が「AppHeader + AppSidebar + メインコンテンツ領域」を提供し、`children` を受け取るコンポーネントとして定義されている。

## 判定
高 / 内部整合性不備

## 修正方針案
両画面のコンポーネントツリーを他画面と同じ形式にそろえ、`AppLayout` の配下に `[メインコンテンツ]` を置いたうえで `PageTitle` 以降をネストして記載する。必要なら同一ファイル内のレイアウト説明文も `children` 契約に合わせて補足する。

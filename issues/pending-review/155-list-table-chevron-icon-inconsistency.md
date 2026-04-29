# 一覧テーブル末尾の ChevronRight アイコンが画面間で不整合

## 発見日
2026-04-28

## カテゴリ
ui-design / frontend

## 影響度
中（機能影響なし。同種の「行クリックで詳細遷移」UX を持つテーブル間で視覚表現が分裂、ユーザーに「違和感」として認知される）

## 発見経緯
user-report / Step 11-A SMK-011 検証時の Approver 承認待ち一覧操作で、ユーザーが「テーブル右端に > マークがある。レポート一覧にはないので違和感」と報告

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現状の二分法

| 画面 | URL | アクセス可能ロール | 「>」（ChevronRight）末尾列 |
|------|-----|------------------|---------------------------|
| 承認待ち一覧 | `/approvals` | Approver | **あり** |
| 支払待ち一覧 | `/payments` | Accounting | **あり** |
| レポート一覧 | `/reports` | Member / Approver / Accounting / Admin（自分のレポート） | なし |
| 全レポート | `/admin/reports` | Admin（テナント内全件） | なし |

### 該当箇所

- `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx:55-62`:
  ```tsx
  {
    field: 'navigate',
    headerName: '',
    width: 48,
    sortable: false,
    disableColumnMenu: true,
    renderCell: () => <ChevronRightIcon fontSize="small" color="action" />,
  },
  ```
- `expense-saas/frontend/src/pages/workflow/PaymentListPage.tsx`: 同等の列定義（要確認）
- `expense-saas/frontend/src/pages/reports/ReportListTable.tsx`: ChevronRight なし
- `expense-saas/frontend/src/pages/admin/AllReportsTable.tsx`: ChevronRight なし

### 観察

- 全 4 画面とも「行クリック → レポート詳細画面に遷移」という UX は共通（DataGrid の onRowClick で navigate）
- 「>」マークの有無は **ワークフロー操作対象（承認待ち / 支払待ち）** vs **レポート閲覧（一覧 / 全レポート）** で分岐している
- 設計書での意図確認が必要 — 意図的な使い分けか、実装時の不統一か

## 影響

- 同種の操作（行クリック → 詳細遷移）に対して視覚表現が分裂し、ユーザーが画面間を行き来する際に違和感を覚える
- 行が押下可能であることを示すアフォーダンスが画面によって異なり、UX 一貫性が崩れる

## 提案

### Step 1: 設計書での意図確認

- `dev-journal/deliverables/docs/55_ui_component/screens/approval-list.md` / `payment-list.md` / `report-list.md` / `admin-all-reports.md` を確認
- 「>」マーク（行末アイコン）の規定があるか、ある場合の意図（ワークフロー対象を強調する等）を確認

### Step 2: 統一方針の確定

**案A: 全画面で「>」を表示**
- 行クリックの押下可能性を視覚化する標準パターンとして全画面に適用
- 4 画面全ての DataGrid 列定義に末尾の ChevronRight 列を追加

**案B: 全画面で「>」を削除**
- DataGrid 自体の `cursor: pointer` / hover 効果で押下可能性は十分伝わるとし、視覚要素を最小化
- ApprovalListPage / PaymentListPage の末尾列を削除

**案C: 現状維持（意図的）**
- ワークフロー操作対象（承認待ち / 支払待ち）には「>」を付け、操作可能性を強調
- レポート閲覧画面（レポート一覧 / 全レポート）は「>」を付けない
- ただしこの場合、設計書側に明示的に「ワークフロー操作対象には末尾に ChevronRight を表示」と規定し、根拠を残す

### 推奨

設計書を確認した上で、**案A or 案B** で統一するのが UX 一貫性の観点で望ましい。案C を採用するなら設計書側で意図を明記すること。

## 修正対象ファイル（候補）

| ファイル | 変更種別 |
|---|---|
| `dev-journal/deliverables/docs/55_ui_component/screens/{approval-list,payment-list,report-list,admin-all-reports}.md` | 「>」マーク仕様の統一規定追加 |
| `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx` | 案B 採用時は ChevronRight 列削除 |
| `expense-saas/frontend/src/pages/workflow/PaymentListPage.tsx` | 案B 採用時は同上 |
| `expense-saas/frontend/src/pages/reports/ReportListTable.tsx` | 案A 採用時は ChevronRight 列追加 |
| `expense-saas/frontend/src/pages/admin/AllReportsTable.tsx` | 案A 採用時は同上 |

## 完了条件

- 設計書で「>」マークの仕様が統一的に規定されている
- 4 画面すべての DataGrid 列定義が設計書通りに統一されている
- 既存テストが通る + 必要に応じて新規テスト更新

## ラベル

- type: ui / inconsistency
- area: frontend / docs（設計確認）

## 関連

- 関連 SMK: SMK-011（二重押下防止）の副次発見

## MVP 区分

MVP（UX 一貫性、ユーザーが直接目にする）

---

## 解決内容

**採用方針**: 案 B（全画面で「>」削除）

**実装** (PR #107, commit be95274):
- `ApprovalListPage.tsx`: COLUMNS から `field: 'navigate'` 列定義を削除、`ChevronRightIcon` import 削除
- `PaymentListPage.tsx`: 同上
- `WFL-FE-016` / `WFL-FE-045` の `data-column-count` 期待値を 5→4 に修正

**設計書改訂** (commit 255d801):
- `55_ui_component/screens/{workflow-pending,workflow-payable,report-list,admin-all-reports}.md` の 4 画面に「行末 ChevronRight アイコンは表示しない（4 画面共通方針、行クリック + cursor:pointer のみで操作可能性を表現）」を追記

**注**: チケット指定ファイル名は `approval-list.md` / `payment-list.md` だったが、実ファイル名は `workflow-pending.md` / `workflow-payable.md` だった（dev-journal 60_test/test_cases/workflow.md でも実ファイル名が正本として参照されている）。

## 解決日

2026-04-30

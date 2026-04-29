# テナント全レポート一覧（DataGrid）スマホ幅で列幅省略により内容が読めない

## 発見日
2026-04-29

## カテゴリ
implementation / frontend / ui / responsive

## 影響度
中（機能影響なし。スマホ幅での閲覧時に DataGrid の列内容が ellipsis 省略され、業務上の情報判別が困難）

## 発見経緯
user-report / Step 11-A SMK-101（スマホ幅でのテナント全レポート一覧テーブル表示）検証時に、DevTools で iPhone SE サイズ（375px）に切り替えたところ、横スクロールは発生しないが列幅が短すぎて全列の文字が省略表示（ellipsis）になり、内容が読めないことを発見

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書の期待結果

`dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md:132`（SMK-101）:

> | チェックID | 確認観点 | 期待結果 |
> |-----------|---------|---------|
> | SMK-101 | スマホ幅でのテナント全レポート一覧テーブル表示 | テーブルが**横スクロール可能**になり、**列の内容が重なって読めなくなることがない**。**ハンバーガーメニュー**が表示される |

### 実装の挙動（DevTools iPhone SE 375px で確認）

- **横スクロール**: 発生しない（DataGrid 標準の `width: 100%` で親要素幅にフィット）
- **列内容**: 全列が ellipsis 省略表示で「申請者名 / タイトル / 合計金額 / ステータス / 提出日」のほとんどが読めない
- **ハンバーガーメニュー**: 表示あり（OK）

### 該当箇所

- `expense-saas/frontend/src/pages/admin/AllReportsTable.tsx` — `AppDataGrid` 経由で MUI X `DataGrid` を使用
- `expense-saas/frontend/src/components/ui/AppDataGrid.tsx` — DataGrid 共通ラッパー

### 影響範囲

`AppDataGrid` を使う他画面でも同種の問題を抱える可能性:

| 画面 | コンポーネント | スマホ幅検証状況 |
|------|--------------|----------------|
| テナント全レポート一覧 | `AllReportsTable.tsx` | **本 issue で FAIL 確認** |
| 承認待ち一覧 | `ApprovalListPage.tsx` | SMK で未検証（Phase 3 SMK-063 は詳細画面のボタン領域確認のみ） |
| 支払待ち一覧 | `PaymentListPage.tsx` | SMK で未検証 |

→ 修正方針次第で 1〜3 画面に波及。

### 過去経緯

- issue **#137**（PR #92 で解決）「スマホ幅でレポート一覧・明細一覧がページ全体横スクロール」で、`<Table>` 直置き系を `TableContainer` でラップして `overflow-x: auto` を付与する形で解決
- ただし AllReportsTable は MUI X `DataGrid` 使用で別アーキテクチャのため対象外。#137 ファイル L145 で「**SMK-101（テナント全レポート一覧のスマホ幅表示）: AllReportsTable は DataGrid 使用のため別挙動。Phase 6 Admin で検証**」と明記され残課題として織り込み済み
- 今回 SMK-101 で実際に検証 → 想定通り FAIL → 本 issue で対応

## 影響

- スマホでテナント全レポート一覧を確認する Admin / Accounting ロールが、各レポートの内容を判別できない
- 列幅省略により会社名・タイトル・金額・ステータス・提出日のほぼ全てが読めない状態
- 承認待ち一覧 / 支払待ち一覧（DataGrid 共通基盤）も同種の問題を抱える可能性が高い

## 提案

### 修正案A: 各列に minWidth 設定 + 親 Box に overflow-x: auto（横スクロール案、推奨）

`AllReportsTable.tsx` の `COLUMNS` に各列の `minWidth` を設定し、合計幅が画面幅を超える場合は親要素で横スクロール可能にする:

```tsx
const COLUMNS: GridColDef[] = [
  { field: 'submitter_name', headerName: '申請者名', flex: 1, minWidth: 120 },
  { field: 'title', headerName: 'タイトル', flex: 2, minWidth: 200 },
  { field: 'total_amount', headerName: '合計金額', flex: 1, minWidth: 100 },
  { field: 'status', headerName: 'ステータス', flex: 1, minWidth: 100 },
  { field: 'submitted_at', headerName: '提出日', flex: 1, minWidth: 110 },
];
```

DataGrid 親要素の overflow 設定:
```tsx
<Box sx={{ overflowX: 'auto' }}>
  <AppDataGrid ... />
</Box>
```

メリット: SMK-101 の期待結果（横スクロール可 + 列内容が読める）に整合
デメリット: 横スクロール UX は若干劣る（縦スクロール優先のスマホで横スクロール）

### 修正案B: columnVisibilityModel でスマホ幅では一部列を非表示

useMediaQuery でスマホ幅検出 → DataGrid の `columnVisibilityModel` で副次列（提出日 / 申請者名等）を非表示にする:

```tsx
const isSmallScreen = useMediaQuery('(max-width: 600px)');
const columnVisibilityModel = isSmallScreen
  ? { submitter_name: false, submitted_at: false }
  : {};
```

メリット: 横スクロールなしで読める
デメリット: 情報が一部隠れる（タップ展開等の補助 UI が必要）

### 修正案C: AppDataGrid 共通基盤で対応

`AppDataGrid` 自体に minWidth / overflow 設定を組み込み、全ての DataGrid 使用画面に波及効果を出す。

メリット: 1 箇所修正で承認待ち/支払待ち/全レポートが解消
デメリット: 共通基盤の影響範囲が大きい

### 推奨

**修正案A（各列 minWidth + 横スクロール）+ AppDataGrid 共通基盤側で overflow 対応**。

設計書 SMK-101 期待結果が「横スクロール可能」を明示しているため、案A が設計整合的。AppDataGrid 共通基盤で overflow を持たせれば承認待ち/支払待ちも自動的に解消（影響範囲を一気にカバー）。

## 修正対象ファイル（推定、推奨案）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/ui/AppDataGrid.tsx` | 親 Box の overflow-x: auto 設定 |
| `expense-saas/frontend/src/pages/admin/AllReportsTable.tsx` | 各列に minWidth 追加 |
| `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx` | 同上（並行で修正） |
| `expense-saas/frontend/src/pages/workflow/PaymentListPage.tsx` | 同上 |
| 各テスト | スマホ幅での横スクロール挙動テスト追加 |

## 完了条件

- スマホ幅（375px）で AllReportsTable の各列内容が省略されず読める（横スクロール経由）
- 横スクロールが DataGrid 領域内で完結する（ページ全体スクロールにならない）
- 承認待ち一覧 / 支払待ち一覧でも同様に動作する
- 既存テスト通過 + 新規テスト追加

## ラベル

- type: bug / ui / responsive
- area: frontend

## 関連

- 過去 issue: #137（PR #92 で解決、本 issue は #137 解決時に「Phase 6 で検証」と明記された残課題）
- 関連 SMK: SMK-101（スマホ幅でのテナント全レポート一覧テーブル表示）

## MVP 区分

MVP（業務上必要な情報判別、UX 影響大）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

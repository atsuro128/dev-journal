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

**採用方針**: 案 A（各列 minWidth 設定）+ AppDataGrid 共通基盤側で `overflowX: 'auto'` 対応

**実装** (PR #107, commits be95274 / fb09eb4):
- `AppDataGrid.tsx` ルート Box の sx に `overflowX: 'auto'` を追加 → 列幅合計が画面幅を超える場合に DataGrid 領域内で横スクロール可能（ページ全体スクロールにはしない）
- `AllReportsTable.tsx`: 各列に minWidth 追加（申請者名 120 / タイトル 200 / 合計金額 100 / ステータス 100 / 提出日 110）
- `ApprovalListPage.tsx`: 同上（提出日 110）
- `PaymentListPage.tsx`: 同上（承認日 110）
- `ReportListTable.tsx`: 各列に minWidth 追加（タイトル 200 / 対象期間 180 / 合計金額 100 / ステータス 100 / 作成日 110）— ユーザー判断でスコープ拡張、Member 用画面も同基盤対応

**スコープ拡張の経緯**: 当初は SMK-101 で報告された AllReportsTable のみだったが、ReportListTable も同じ AppDataGrid 共通基盤を使用しており構造的に同症状である可能性が高いため、予防的に対応範囲を 4 画面に拡張。

**テスト追加** (commit fb09eb4 で改善):
- ADG-006: AppDataGrid ルート Box に `overflowX: 'auto'` が実値で渡されていることを検証（Box mock を `data-overflow-x` 属性に展開）。codex 指摘で当初の実装（tagName === 'DIV' のみ検証）を強化、回帰防止性能を実証（一時削除で FAIL → 復元で PASS）

**設計書改訂** (commit 255d801):
- `55_ui_component/common-components.md` §AppDataGrid に「ルート Box は `overflowX: 'auto'` で列幅合計が画面幅を超える場合に横スクロール」を追記

---

## 再対応の解決内容（2026-04-30 第3バッチ）

**再対応理由**: SMK-101 再検証で「横スクロールしない / 列省略」が再確認され、初回修正の効果が効いていないことが判明（pending-review → open に再戻し）

**真因確定（Phase 1）**:
- 4 画面の columnDef で `flex: N` と `minWidth: M` が併用されていた
- MUI X DataGrid の仕様で `flex` 指定列はコンテナ幅を flex 比率で分配するため、スマホ幅（375px）では各列が minWidth 以下に縮小
- ルート Box の `overflowX: 'auto'` が設定されていても DataGrid がコンテナ内に収まるよう縮むため横スクロールが発火しない

**採用方針**: 候補 A（4 画面 columnDef の `flex` 削除 + `width` 絶対値固定）

**実装** (PR #114, commit `664efbc`):
- 4 画面（AllReportsTable / ReportListTable / ApprovalListPage / PaymentListPage）の columnDef から `flex: N` を全削除し `width` 絶対値で固定
- 列幅合計（540〜710px）が 375px を超えるため、ルート Box の `overflowX: 'auto'` が正常発火
- AppDataGrid 共通基盤の `overflowX: 'auto'`（Box ルート）は維持

**テスト**: 4 画面 columnDef 修正 + AppDataGrid.test.tsx ADG-006 維持で計 65 + 7 件 PASS

**設計書改訂** (commit `40ccb0f`):
- `55_ui_component/common-components.md` §AppDataGrid の columnDef 規定を「`flex` 併用禁止、`width` / `minWidth` 絶対値必須」に格上げ
- 標準値（申請者名 120 / タイトル 200 / 金額 100 / ステータス 100 / 日付 110）を明文化

**SMK 再検証要項目**: SMK-101

**残課題（後続判断）**: PC 幅で flex 削除によりデスクトップ幅 DataGrid の右側に空白が生じる可能性。視覚確認次第で最後尾列に `flex: 1` 追加対応の可能性

## 解決日

2026-04-30

---

## 再検証結果（2026-05-01 ローカル動作確認）

### 状態

**FAIL**（PR #114 の対応が両面で有効に機能していなかったことを確認、revert 実施・継続調査）

### 実機で判明した不備

1. **PC 全画面で過剰な余白**: 列幅合計（540〜710px）が lg コンテナ（最大 1152px）を大きく下回り、各画面で 442〜612px の右側空白が発生し UI 違和感が大きい
2. **スマホ幅でも横スクロール発火しない**: DevTools で 375px 幅に切り替えても、PR #114 が想定した「列幅合計 > 画面幅 → AppDataGrid Box の `overflowX: auto` 発火」が観測されない（MUI X v8 の挙動として、DataGrid 内部の virtualScroller が独自にコンテナ幅にフィットしているか、あるいは別の機序が働いている可能性）

### 対応（リバート）

PR #114 の columnDef 変更（4 画面）を **revert** する。

- `pages/reports/ReportListTable.tsx` / `pages/admin/AllReportsTable.tsx` / `pages/workflow/ApprovalListPage.tsx` / `pages/workflow/PaymentListPage.tsx` を PR #114 直前（commit `664efbc^`）の状態（`flex: N` + `minWidth: M`）に復元
- `components/ui/AppDataGrid.tsx` の Box `overflowX: 'auto'` は維持（無害、将来 columns が container を超えるケースの保険）
- `components/ui/AppDataGrid.tsx` の `--DataGrid-overlayHeight: 'auto'`（PR #118）は独立機能として維持
- 設計書 `55_ui_component/common-components.md` §AppDataGrid の規定を `flex` + `minWidth` 併用許容に戻す

### リバート後の状態

- PC 幅: flex で列が伸びるため右余白なし（PR #114 以前の挙動に戻る）
- スマホ幅: ellipsis 症状が再発する可能性が高い（issue #160 元の症状）→ **本 issue は引き続き未解決**として open 化

### 真因再分析の方向性（次セッション以降）

- MUI X v8 で `flex` + `minWidth` 併用時の実動作を再観測（v6 〜 v8 で挙動が変わった可能性）
- DataGrid 内部の `virtualScroller` のサイジングロジック調査
- 代替案: レスポンシブで列を間引く / カードビュー切替 / `responsive` 系 props の活用

### 起票日（再 open）

2026-05-01

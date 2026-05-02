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

### 観察された具体症状（2026-05-01）

> 「スマホ用画面に変わった瞬間、テーブル幅が変わらなくなる。画面を狭めると見切れる」

ブレークポイント遷移後、DataGrid の幅が viewport の縮小に追従せず、内容がクリップされる（横スクロールが発火しない）。

参考: ProcessedReportsPage（SCR-WFL-003）も他 4 画面と同じ `flex + minWidth` パターンだが、ユーザーから「UI としてきれい」との評。一方で本 issue（mobile での横方向見切れ）は全画面共通の症状であり、テーブル単体の columnDef ではなく **DataGrid のサイジング機構そのもの** に起因する可能性が高い。

### 調査ポイント（次セッション着手用メモ）

1. **DataGrid 内部 scrollbar の可視性確認**
   - DOM 上は `MuiDataGrid-scrollbar--horizontal` が render されている（PR #114 検証時に DevTools で確認済み）
   - `display` / `visibility` / `pointer-events` 等で隠れていないか CSS を観察

2. **AppDataGrid Box ラッパーの幅指定の見直し**
   - 現状: `<Box sx={{ width: '100%', overflowX: 'auto' }}>` が DataGrid を内包
   - `width: '100%'` のため Box 自身がコンテナ幅で止まり、DataGrid 内部 overflow が「Box の overflowX」として伝播していない可能性
   - 検証案: Box の width を `'auto'` または `'fit-content'` に変えて Box が DataGrid のサイズに追従するか確認
   - 検証案: Box ではなく親コンテナ側で `overflowX: 'auto'` を持たせる

3. **MUI X v8 の DataGrid 設定**
   - `disableColumnResize` / `columnBufferPx` 等の関連 props
   - v8 で virtualScroller の scroll が disable されるデフォルト条件があるか
   - `--DataGrid-rowWidth` / `--DataGrid-columnsTotalWidth` / `--DataGrid-hasScrollX` の CSS 変数とコンテナ幅の関係
   - `--DataGrid-hasScrollX: 1` が設定されていても可視 scrollbar が出ない条件

4. **代替アーキテクチャ**
   - viewport breakpoint で `columnVisibilityModel` を切り替え（mobile では status / 日付などを非表示）
   - mobile では DataGrid ではなく `<Card>` ベースのリスト表示に切替
   - MUI 公式の Responsive DataGrid パターン（v8 で提供されているか要調査）

### 関連コミット

- PR #114 (664efbc): width 絶対値固定で解消を試みたが PC 余白過剰 + mobile 横スクロール未発火
- PR #119 (5a12d33): PR #114 を revert、`flex + minWidth` パターンに復元

### 起票日（再 open）

2026-05-01

---

## 真因確定と採用方針（2026-05-02、実機計測ベース）

### 実機計測結果（DevTools iPhone SE 375px）

`.MuiDataGrid-root` の computed style:

```
--DataGrid-hasScrollX: 0
--DataGrid-hasScrollY: 0
--DataGrid-rowWidth: 690px
--DataGrid-columnsTotalWidth: 690px
```

階層別 width:

| 階層 | 要素 | computed width | display |
|---|---|---|---|
| 親 | `<Box css-k008qs>` (PageContainer 等) | 375px | **flex** |
| 子（AppDataGrid Box） | `<Box css-d9bexz>` (`width: '100%', overflowX: 'auto'`) | **726px** | block |
| 孫 | `.MuiDataGrid-root` | 726px | (DataGrid) |

### 真因

**CSS Flexbox の `min-width: auto` の罠**:

- AppDataGrid の Box ラッパー (`<Box sx={{ width: '100%', overflowX: 'auto' }}>`) は `width: '100%'` を指定している
- しかし親が `display: flex` のため、Box は flex item として扱われる
- Flex item のデフォルト `min-width: auto` は **コンテンツ（DataGrid 内部 726px）より縮まない**
- 結果、Box が `width: '100%'` でなく **content size の 726px に追従して膨張**
- DataGrid 内部判定（`columnsTotalWidth (690) > root (726)` ではない）で `hasScrollX = 0` となり scrollbar 不要と判定
- Box の `overflowX: 'auto'` は機能しているが、Box 自身が 726px に膨らむため overflow が発火しない

### architect 仮説（2026-05-01 レポート）の修正

architect レポートの仮説「`columnsTotalWidth` が `rootSize.width = 343px` に追従して縮む（min violation 後の再フロー）」は実機 `columnsTotalWidth = 690px` で否定された。MUI X v8 の computeFlexColumnsWidth 内部仕様の問題ではなく、**外側の CSS Flexbox レイアウト問題** が真因。

そのため architect 推奨の **案 B（columnVisibilityModel）/ 案 C（カードビュー）/ 案 E は不要**。テーブル構造を変えずに解決できる。

### 採用方針: 案 F（新規、Flexbox min-width 罠の解消）

**`AppDataGrid` の Box ラッパー sx に `minWidth: 0` を追加する**。

```tsx
// 修正前
<Box sx={{ width: '100%', overflowX: 'auto' }}>

// 修正後
<Box sx={{ width: '100%', minWidth: 0, overflowX: 'auto' }}>
```

これで:

- Box が flex container 内で 375px に縮まる（`min-width: auto` を 0 に上書き）
- Box は `overflowX: 'auto'` を持つので、内部 DataGrid (726px) がはみ出して **横スクロールバー発火**
- DataGrid 自体は触らない（columns / `flex` / `minWidth` はそのまま維持）
- PC 幅では Box が親コンテナ（lg 1152px）まで広がるので flex 列が分配されて右余白なし
- **テーブル構造を一切変えない**（SMK-101 期待結果「列の内容が重なって読めなくなることがない」+「横スクロール可能」を両立）

### 修正対象

| ファイル | 変更内容 |
|---|---|
| `expense-saas/frontend/src/components/ui/AppDataGrid.tsx` | Box sx に `minWidth: 0` 追加 |
| `expense-saas/frontend/src/components/ui/__tests__/AppDataGrid.test.tsx` | minWidth: 0 が Box に渡されていることを検証する新規テスト追加 |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` §AppDataGrid | Flexbox `min-width: auto` の罠と `minWidth: 0` が必須である根拠を追記 |

### 各画面の columnDef は変更不要

`flex + minWidth` パターン（PR #119 で復元済み）はそのまま維持。案 F は AppDataGrid 共通基盤側 1 行追加のみで全 5 画面（AllReportsTable / ReportListTable / ApprovalListPage / PaymentListPage / ProcessedReportsPage）に波及する。

### 並行発見事項（別 issue で対応）

ReportListTable (マイレポート) 画面で「ステータス / 開始日 / 終了日」のフィルタエリアが mobile 幅で横一列のまま画面幅を超える症状を観察。これは DataGrid の問題ではなく、**フィルタエリアのレイアウトが mobile 対応していない**別問題のため、別 issue として切り出して対応する。

### SMK 再検証要項目

SMK-101（テナント全レポート一覧 + 他 4 画面の横スクロール / 列内容判別性）

### 起票日（案 F 採用）

2026-05-02

---

## 真因再々確定と追加修正方針（2026-05-03、PR #122 マージ後の SMK 再検証 FAIL 受け）

### 状態

PR #122（AppDataGrid Box に `minWidth: 0` 追加）マージ後の SMK-101 再検証で **依然として FAIL**（横スクロール発火せず、Box が 726px のまま）を確認。実機計測で真因の場所が誤っていたことが判明。

### 計測対象の取り違え（前回の誤り）

「2026-05-02 真因確定」セクションで「Box (`css-d9bexz`) 726px = AppDataGrid の Box ラッパー」と特定したが、**実際の DOM 階層では `css-d9bexz` は `<main>` 要素**だった。HTML 構造を改めて読み解いた結果：

```
<main class="MuiBox-root css-d9bexz">    ← 726px、flex-grow: 1（これが本当の元凶）
  <div class="MuiToolbar-root">
  <div class="MuiContainer-root MuiContainer-maxWidthLg">
    <div class="MuiBox-root css-0">
      <div data-testid="report-list-table">
        <div class="MuiBox-root css-2t1ku1">  ← AppDataGrid Box（PR #122 で minWidth: 0 + overflowX: auto 追加済み、ここは正しく機能）
          <div class="MuiDataGrid-root">      ← 690px (columnsTotalWidth)
```

`<main>` が flex item として `min-width: auto` で content (DataGrid 690px + Container padding) に追従して 726px に膨張していたため、その内側の AppDataGrid Box が `minWidth: 0` を持っていても **既に外側で 726px に広がっており発火しない**状態だった。

### 真因の場所

`/root-project/expense-saas/frontend/src/components/layout/AppLayout.tsx` L95-102:

```tsx
<Box
  component="main"
  sx={{
    flexGrow: 1,
    width: { md: `calc(100% - ${DRAWER_WIDTH}px)` },  // mobile (xs) で width 未指定
    // minWidth: 0 が無いため flex item の min-width: auto で content に追従して膨張
  }}
>
```

mobile（xs）では `width` が未指定のため、`flexGrow: 1` + `min-width: auto` の組合せで content size に追従して膨張する CSS Flexbox の罠。

### PR #122 の評価（再評価）

**完全に無駄ではない**。AppLayout の `<main>` を 375px に縮めた後、内側の AppDataGrid Box で `overflowX: 'auto'` + `minWidth: 0` が「内部 DataGrid (690px) を overflow させて横スクロールバー発火」する **最後の砦** として機能する。ただし AppLayout 側の修正なしには発火しないため、**PR #122 単独では効果ゼロ** だった。

PR #122 の修正と設計書改訂自体は CSS Flexbox 仕様として正しく、レイアウト改善の保険として維持する。

### 採用方針: 案 F'（AppLayout の `<main>` にも `minWidth: 0` 追加）

`AppLayout.tsx` の `<Box component="main">` の sx に `minWidth: 0` を追加する：

```tsx
sx={{
  flexGrow: 1,
  minWidth: 0,  // 追加
  width: { md: `calc(100% - ${DRAWER_WIDTH}px)` },
}}
```

これで mobile でも `<main>` が viewport (375px) に収まり、内側の Container / Box / AppDataGrid Box が連鎖して 375px に縮み、AppDataGrid Box の `overflowX: 'auto'` で内部 DataGrid (690px) が overflow して横スクロールバー発火する。

### 修正対象ファイル

| ファイル | 変更内容 |
|---|---|
| `expense-saas/frontend/src/components/layout/AppLayout.tsx` | `<Box component="main">` の sx に `minWidth: 0` 追加 |
| 必要に応じて該当する AppLayout のテスト | minWidth: 0 が main に渡されていることを検証する新規テスト追加 |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` § AppLayout（または該当セクション） | `<main>` に `minWidth: 0` を必須とする根拠を追記。AppDataGrid §の規定（PR #122 で追加）と一貫性を取る |

### 影響範囲

`AppLayout` は全画面の基盤コンポーネントのため、修正は **全ページに波及**する。影響として：

- mobile では `<main>` が viewport に収まる（DataGrid 系に限らず、内部 content がはみ出すケース全般で改善）
- PC（md 以上）では既存の `width: calc(100% - DRAWER_WIDTH)` がそのまま効くため挙動変化なし
- 副作用リスク: `flexGrow: 1` と組み合わせた `minWidth: 0` は flex item の標準的なテクニックで、レイアウトが崩れる可能性は低い

### SMK 再検証要項目（変更なし）

SMK-101（テナント全レポート一覧 + 他 4 画面の横スクロール / 列内容判別性）

### 起票日（案 F' 追加修正）

2026-05-03

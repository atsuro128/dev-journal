# マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える

## 発見日
2026-05-02

## カテゴリ
implementation / frontend / ui / responsive

## 影響度
中（機能影響なし。スマホ幅でのフィルタ操作時に画面幅を超え、UI が破綻して操作不能になる可能性）

## 発見経緯
user-report / Step 11-A SMK 再検証中に issue #160 の真因調査の過程で発見。DevTools iPhone SE 375px でマイレポート画面 (ReportListTable) を確認したところ、上部のフィルタエリア（ステータス / 開始日 / 終了日）が **横一列のまま画面幅 375px に収まらず、画面外に溢れている**ことを観察。

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

`ReportListTable.tsx`（または上位ページ）の上部フィルタエリア:

- ステータス絞り込み select
- 開始日 date picker
- 終了日 date picker

がスマホ幅（375px）で **横一列の flex layout のまま画面幅を超えて溢れている**。

### 期待動作

スマホ幅では:

- フィルタが **縦積み**（または折りたたみ式）になり画面幅に収まる
- 各 input 要素が tap しやすい幅で表示される
- ハンバーガーメニュー / グローバルナビゲーションと干渉しない

### 該当箇所（推定）

- `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`（または `ReportListTable.tsx` の親コンポーネント）の上部フィルタエリア

レイアウトは恐らく `<Stack direction="row">` または `<Box sx={{ display: 'flex' }}>` で固定された row layout になっている。

### 影響範囲

マイレポート画面のみ確認済み。他の一覧画面（テナント全レポート / 承認待ち / 支払待ち / 処理済み）にも同種のフィルタエリアがあれば同症状の可能性があるため、修正時に横展開要否を確認すること。

## 影響

- スマホでマイレポート画面を開いた Member ロールが、フィルタを正常に操作できない
- 画面幅を超えた要素が右側に隠れ、ユーザーは存在に気付けない可能性
- SMK-101 の延長問題（テーブル幅問題と同様、mobile レスポンシブ対応の漏れ）

## 提案

### 修正案 A: Stack の direction をレスポンシブに切替（推奨）

```tsx
<Stack direction={{ xs: 'column', sm: 'row' }} spacing={2}>
  <StatusSelect ... />
  <DatePicker label="開始日" ... />
  <DatePicker label="終了日" ... />
</Stack>
```

メリット: 最小変更。MUI 公式 Responsive パターン
デメリット: mobile で縦積み 3 要素になり画面が縦に伸びる

### 修正案 B: アコーディオンで折りたたみ

mobile 幅では `<Accordion>` で折りたたんで「フィルタ」見出しタップで展開する形。

メリット: 画面領域を圧迫しない
デメリット: 1 アクション増える、実装やや複雑

### 推奨

**案 A**（Stack direction レスポンシブ）。最小変更で SMK-101 期待結果に沿う。Member 画面は頻繁にフィルタを使う想定のため、初期表示で見える状態（縦積み）の方が UX 良い。

## 修正対象ファイル（推定、案 A）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`（または該当コンポーネント） | フィルタエリアの Stack を `direction={{ xs: 'column', sm: 'row' }}` に変更 |
| 対応するテストファイル | mobile 縦積みを検証するテスト追加 |
| `dev-journal/deliverables/docs/50_detail_design/screens/report-list.md`（該当画面の設計書） | mobile レスポンシブ規定を追記 |

修正前に該当ファイルを特定すること（`grep` で「ステータス絞り込み」「開始日」「終了日」のラベル文言を検索）。

## 完了条件

- スマホ幅（375px）でマイレポート画面のフィルタエリアが画面幅に収まる（案 A なら縦積み）
- 各フィルタ要素が tap しやすい幅で表示される
- 既存テスト通過 + 新規テスト追加
- 横展開要否確認（テナント全レポート / 承認待ち / 支払待ち / 処理済みでも同種フィルタエリアがあれば同時対応）

## 追加対応ログ（再調査でスコープ修正）

### 確認日
2026-05-05

### 再調査の経緯

issue #165 起票後にユーザー手動確認したところ、**「フィルタエリアが画面外に溢れる」現象は解消済み**であることが判明（issue #160 の AppLayout `<main>` 375px 幅修正の副次効果と推定）。

ただし新たな問題を発見:

1. **マイレポート画面**: `<Box sx={{ display: 'flex' }}>` 横並び固定で、375px では各 input が圧縮されてラベル・パラメータが省略表示される（読めない）
2. **全レポート画面**: `<AllReportsFilterBar>` の各 `<AppSelect>` / `<AppDatePicker>` が `fullWidth=true` デフォルトで縦積みされ、PC 画面で過剰幅になる
3. **画面間レイアウト不整合**: 一覧 5 画面でフィルタエリア構造が統一されていない

### 各画面の現状

| 画面 | 構造 | 各要素 fullWidth | 結果 |
|------|------|-------------------|------|
| マイレポート (`ReportListPage`) | `<Box flex>` 横並び | AppSelect: `false` 明示, TextField: 指定なし | 横並び圧縮（375px で省略） |
| 全レポート (`AllReportsFilterBar`) | `<div>` 縦並び | AppSelect / AppDatePicker: **true（デフォルト）** | 過剰幅縦並び |
| 承認待ち (`ApprovalListPage`) | `<div>` | TextField: 指定なし | 適切幅単体（OK）|
| 支払待ち (`PaymentListPage`) | 同上 | 同上 | 同上（OK）|
| 処理済み (`ProcessedReportsPage`) | 同上推定 | 同上 | 同上（OK）|

### 共通コンポーネント側の問題

- `AppSelect`: `fullWidth = true` をデフォルト化（実装上）
- `AppDatePicker`: `fullWidth` をハードコード（Props で制御不可）

`AppDatePicker` の `fullWidth` ハードコードは AllReportsFilterBar での過剰幅問題の根本原因。Props 化が必要。

### 修正方針（A 案: 必須のみ、ユーザー判断）

#### 1. マイレポート (`ReportListPage.tsx`)

```tsx
<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2, mb: 2 }}>
  <AppSelect ... fullWidth={false} sx={{ width: 140 }} />
  <TextField ... sx={{ width: 170 }} />  {/* 開始日 */}
  <TextField ... sx={{ width: 170 }} />  {/* 終了日 */}
</Box>
```

#### 2. 全レポート (`AllReportsFilterBar.tsx`)

```tsx
<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2 }}>
  <AppSelect ... fullWidth={false} sx={{ width: 140 }} />        {/* ステータス */}
  <AppDatePicker ... fullWidth={false} sx={{ width: 170 }} />    {/* 開始日 */}
  <AppDatePicker ... fullWidth={false} sx={{ width: 170 }} />    {/* 終了日 */}
  <AppSelect ... fullWidth={false} sx={{ width: 200 }} />        {/* 申請者 */}
</Box>
```

#### 3. `AppDatePicker.tsx` の `fullWidth` を Props 化

```tsx
// 修正前: <TextField ... fullWidth ... />
// 修正後: 
interface Props {
  ...
  fullWidth?: boolean;  // 追加。後方互換のためデフォルト true
}
// JSX 内: <TextField ... fullWidth={fullWidth} ... />
```

#### 4. 承認待ち / 支払待ち / 処理済み

**touch 不要**（現状で適切幅）。

### スコープ外

- 一覧 5 画面のレイアウトパターン完全統一（B 案）。承認待ち系を flex-wrap パターンに揃える対応は本スコープ外（ユーザー判断: A 案採用）。

### 完了条件（更新）

- ✅ マイレポート画面で 375px・PC 双方で各フィルタ要素が適切幅で表示される
- ✅ マイレポート画面で flex-wrap により mobile では自動折り返しが効く
- ✅ 全レポート画面で各フィルタ要素が適切幅で表示される（過剰幅解消）
- ✅ 全レポート画面で flex-wrap により mobile では自動折り返しが効く
- ✅ AppDatePicker の `fullWidth` Props 化が後方互換維持で実装される
- ✅ 関連テスト追加 + 既存テスト通過
- ✅ 設計成果物（screens/report-list.md / admin-all-reports.md）にレイアウト規定追記

## ラベル

- type: bug / ui / responsive
- area: frontend

## 関連

- 関連 issue: #160（同じ Step 11-A 検証中に発見、テーブル mobile 対応の延長問題）
- 関連 SMK: 既存 SMK では未カバー（Step 11-A で目視発見）

## MVP 区分

MVP（業務上 mobile 操作の前提が崩れる UX 影響、Step 11-A 残課題のため）

## 解決ログ

### 解決日
2026-05-06

### PR
- PR #135（マージコミット: `6948f23`）: 主対応 — マイレポート + 全レポート flex-wrap 化、AppDatePicker fullWidth Props 化
- PR #136（マージコミット: `b8ddc1b`）: 改行位置調整 — 開始日・終了日を内側 Box でセット化 + 日付 width 170→160
- PR #137（マージコミット: `be2b4fe`）: デグレ修正 — AllReportsFilterBar に `mb: 2` 追加（フィルタバー下余白）

### 修正内容（最終）

#### 1. マイレポート (`ReportListPage.tsx`)
- フィルタコンテナを `<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2, mb: 2 }}>` に
- ステータス AppSelect: `sx={{ width: 140 }}`
- 開始日・終了日 TextField を内側 `<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2 }}>` でラップ
- 開始日・終了日 TextField: `sx={{ width: 160 }}`

#### 2. 全レポート (`AllReportsFilterBar.tsx`)
- フィルタコンテナを `<Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 2, mb: 2 }}>` に（`<div>` から変更）
- ステータス AppSelect: `fullWidth={false}` + `sx={{ width: 140 }}`
- 開始日・終了日 AppDatePicker を内側 Box でラップ + `fullWidth={false}` + `sx={{ width: 160 }}`
- 申請者 AppSelect: `fullWidth={false}` + `sx={{ width: 200 }}`
- `<p role="alert">` を削除し、エラー表示を AppDatePicker の `errorMessage`（helperText）に一本化

#### 3. 共通コンポーネント
- `AppDatePicker`: `fullWidth` を Props 化（デフォルト `true` で後方互換）+ `sx` Props 追加
- `AppSelect`: `sx` Props 追加

### 改行挙動（最終）

| 画面 | 375px 幅 | PC 幅 |
|------|----------|-------|
| マイレポート | ステータス / 改行 / 開始日 + 終了日 | 横並び |
| 全レポート | ステータス / 開始日 + 終了日 / 申請者（各単独行）| 横並び |

### 起票時観察の真因

issue 起票時の「フィルタエリアが画面外に溢れる」現象は、PR #135 着手前の手動再確認で解消済みと判明。真因は issue #160（DataGrid 列幅 + AppLayout `<main>` 375px 幅修正）の副次効果でフィルタも収まっていた。ただしコード上は flex row 固定で各 input の圧縮表示 / 過剰幅 / レイアウト不整合という別問題があったため、本 issue を再フォーカスして flex-wrap + 適切な width 指定に統一した。

### レビュー
- PR #135 内部レビュー: PASS（warning 3 → 全対応で再 PASS）
- PR #135 codex: 初回 REQUEST CHANGES（テストセレクタ / 二重表示 / テスト品質）→ 対応後 APPROVE 相当
- PR #136 codex: 指摘なし
- PR #137 codex: 指摘なし

### 完了条件の達成
- ✅ マイレポート画面で 375px・PC 双方で各フィルタ要素が適切幅で表示される（flex-wrap で意味単位の改行）
- ✅ 全レポート画面で各フィルタ要素が適切幅で表示される（過剰幅解消）
- ✅ AppDatePicker の `fullWidth` Props 化が後方互換維持で実装される
- ✅ 関連テスト追加（ADT-008/009, RPT-FE-117, TNT-FE-054）+ 既存テスト通過
- ✅ 設計成果物（screens/report-list.md / admin-all-reports.md）にレイアウト規定を追記
- ✅ 手動動作確認 PASS（PC + 375px の両方、エラー表示、デグレ修正後の余白）
- ⏭️ 承認待ち / 支払待ち / 処理済み画面は touch せず（A 案スコープ）

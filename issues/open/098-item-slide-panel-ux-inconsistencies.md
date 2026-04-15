# ItemSlidePanel の UX 不整合 4 件（横幅・閉じるボタン・アニメーション・閲覧モード disabled）

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / ui / ux

## 影響度
低〜中（機能には影響しないが、ポートフォリオ品質としては修正したい UX 課題）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-012（明細スライドパネル）の目視確認中、Member ロールで draft レポートを開き、明細追加 / 明細行選択 / 編集ボタンの 3 経路で Drawer を開いた際に複数の見た目の不整合に気付いた。

## 関連ステップ
Step 8-7（共通 UI コンポーネント実装）/ Step 10-E（明細）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 対象ファイル

- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx`
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`
- 起動元: `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx`（推定）

---

## 課題 1: Drawer 横幅が起動経路ごとにバラつく

### 観測
- 「明細追加」ボタン → Drawer の幅 X
- 明細行クリック（閲覧モード） → Drawer の幅 Y
- 明細行の編集ボタン → Drawer の幅 Z
- X / Y / Z がそれぞれ異なり統一感がない

### 原因
`ItemSlidePanel.tsx` L107-113 の `<Drawer>` で `PaperProps` に `width` 指定がない。

```tsx
<Drawer
  anchor="right"
  open={open}
  onClose={onClose}
  PaperProps={{ 'data-testid': 'item-slide-panel' } as PaperProps}
>
```

横幅が指定されていないため、内部コンテンツ（プリフィルされたフォームのテキスト長、添付ファイル一覧の有無、AttachmentArea のサイズ）によって自然幅が変動する。

### 修正案
`PaperProps` に固定幅を指定する。レスポンシブ対応も含めて以下の例:

```tsx
PaperProps={{
  'data-testid': 'item-slide-panel',
  sx: { width: { xs: '100%', sm: 480 } },
} as PaperProps}
```

幅の値（480px）は他の Drawer 系コンポーネント（あれば）と統一すること。

---

## 課題 2: 閉じるボタンの配置が一般的な UI 作法と乖離

### 観測
Drawer 内に「閉じる」テキストボタンがあるが、UI 作法的には**右上に ✕ アイコンボタン**を置く方が自然。

### 該当箇所
`ItemSlidePanel.tsx` L114-119:

```tsx
<div>
  <h2>{title}</h2>
  <Button variant="text" size="small" onClick={onClose}>
    閉じる
  </Button>
</div>
```

### 修正案

ヘッダー領域を flex で組み、右上に `<IconButton>` + `<CloseIcon>` を配置する。

```tsx
<Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between', px: 2, py: 1.5 }}>
  <Typography variant="h6" component="h2">{title}</Typography>
  <IconButton aria-label="閉じる" onClick={onClose} size="small">
    <CloseIcon />
  </IconButton>
</Box>
```

`<h2>` の生 HTML を `<Typography component="h2">` に置き換える点も併せて整える。

---

## 課題 3: スライドアニメーションが起動経路ごとに不揃い

### 観測
- 「明細追加」を押したとき: Drawer が右からスライドインするアニメーションが見える
- 明細行クリック・編集ボタンクリック: スライドしているように見えない（瞬間表示に近い）

### 推定原因
親コンポーネント（`ReportDetailPage.tsx` など）の状態管理パターンによる挙動差と推定。

- **明細追加**: `panelOpen` が一度 false → ボタン押下で true に遷移 → Drawer が `open=false → true` の transition を実行 → スライドアニメーション
- **明細行クリック / 編集ボタン**: `panelOpen` が true のまま `selectedItem` だけ差し替わるパス、または React の同期更新で false→true の transition がスキップされている

`<Drawer>` の `transitionDuration` は MUI デフォルトで適用されるが、open prop が真→真のまま中身だけ変わると transition は走らない。

### 確認すべき点
- `ReportDetailPage.tsx` で `panelOpen` と `selectedItem` の更新順を確認
- 同じセッションで複数の Drawer を開く場合、一度閉じてから開く設計にするか、もしくは初回の `open` 遷移を確実に false→true にするか

### 修正案
**案 A**: 明細クリック・編集ボタンで一度パネルを閉じてから開き直す（`setOpen(false)` → `setTimeout(() => setOpen(true))`）— 一貫性は出るが UX としてやや冗長
**案 B**: 親側の状態を `panelMode | null` の単一 state に集約し、null → 何らかの mode で必ず再 mount されるよう key を付け替える
**案 C**: アニメーションを統一せず、明細追加だけスライド、それ以外は瞬時表示に揃える（一貫性が逆方向）

推奨は **案 B**（アニメーションが常に発火し UX 一貫性が出る）。

---

## 課題 4: 閲覧モードでカテゴリ欄だけがグレー（disabled）表示

### 観測
明細行クリック（閲覧モード）で開いた Drawer 内、フォーム項目の表示状態が以下のように非対称:
- 金額・摘要・日付などの TextField: 編集不可だが**通常の文字色**
- カテゴリ Select: 編集不可で**グレー（disabled）表示**

### 原因
`ItemForm.tsx` L138-146 の `AppSelect` は `disabled={isView || (isPending && !isView)}` を渡しており、view モードで `disabled=true` になる。一方、TextField 系は `read-only` 風の扱い（推定）で disabled とは違う見た目になっている。

`AppSelect` の `disabled` は MUI FormControl の `disabled` に直結し、ラベルとフィールド全体がグレーになる。

### 修正案
view モードの「読み取り専用」表現を全フィールドで統一する。選択肢:

**案 A**: TextField 側に揃えて、`AppSelect` も disabled をやめて read-only 相当の表現にする（背景色を変えるだけなど）。MUI Select には ネイティブな `readOnly` 概念がないので独自実装が必要。

**案 B**: カテゴリ以外も view モードで `disabled=true` に統一し、全フィールドが灰色になるようにする。

**案 C**: view モードでは `<ItemForm>` を使わず、純粋な表示用コンポーネント（`<ItemView>` 等）を新設して描画する。設計書 §6 が「閲覧モードで全操作禁止」と謳っているため、別コンポーネント化の方が責務が明確。

推奨は **案 C**（責務分離 + 表示テスト容易化）だが規模が大きい。**案 B** が最小修正。

---

## 完了条件
- 3 経路で Drawer 横幅が同じ
- ヘッダー右上に ✕ ボタンが配置されている
- 3 経路すべてで Drawer のスライドアニメーションが一貫して再生される
- 閲覧モードで全フォーム項目の見た目（読み取り専用表現）が統一されている

## 関連
- 092: ItemSlidePanel が Drawer 未使用（PR #51 で対応済み） — 本 issue は PR #51 マージ後に発見された UX 課題で、別事象
- 097: AppSelect の outlined 切り欠き不整合 — 同じく ItemForm のカテゴリ欄まわりだが、見た目の症状は別

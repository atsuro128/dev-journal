# AttachmentUploader の UI 課題 2 件（重複ボタンとスピナー未実装）

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / ui / ux

## 影響度
中（UI が二重に見えており混乱を招く + SMK-012 の期待結果「スピナー or プログレス表示」を厳密には満たしていない）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-012（添付アップロード中のローディング）を実施中、ブラウザの Network Throttling（Slow 3G）で人為的に遅延させてアップロードを観察した際に発見。

## 関連ステップ
Step 8-7（共通 UI コンポーネント実装）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 対応前の合意確認（必須）

実装に着手する前に、以下の論点をユーザーと合意すること。UI/UX 系の課題は実装方針が複数あり、勝手に決め打ちすると後で手戻りが発生する。

### 論点 1: ボタンの見た目
- 案 A: MUI `<Button variant="outlined" startIcon={<AddIcon />}>` + visually-hidden input（推奨）
- 案 B: MUI `<Button variant="contained">` で目立たせる
- 案 C: アイコンのみの `<IconButton>`（コンパクト）
- 案 D: 既存の「+ ファイルを追加」テキストのまま（input だけ隠す最小修正）

→ 他の操作ボタン（保存、削除等）との視覚的優先度バランスをユーザーに確認する。

### 論点 2: ローディング表現
- 案 A: ボタン内に `<CircularProgress size={16} />` を表示（スピナー方式）
- 案 B: ボタンの下に `<LinearProgress />`（インジターミネートバー）
- 案 C: ボタンの下に `<LinearProgress variant="determinate" value={...} />`（実進捗を XHR の progress イベントから取得して表示）
- 案 D: テキスト「アップロード中...」のみ（現状維持）

→ 案 C は実装コストが他より高い（XHR への置き換えまたは fetch + ReadableStream 対応が必要）。SMK-012 期待結果との整合と、実装コストのバランスをユーザーに確認する。

### 論点 3: ドラッグ & ドロップエリアの扱い
現在の実装はボタンとは別に `<div onDrop>` で全体をドロップゾーンにしているが、視覚的にドロップ可能領域が示されていない。今回の修正でドロップゾーンも UI として整えるかどうかをユーザーに確認する。

### 論点 4: 文言
- 「ファイルを追加」「ファイルをアップロード」「+ 添付」等の表現候補
- アップロード中の文言「アップロード中...」「アップロードしています...」等

→ 他画面の文言と合わせて表記揺れがないか、用語集（`dev-journal/deliverables/docs/01_glossary.md`）も確認する。

**手順**: 上記 4 論点を持ってユーザーに「対応方針確認」のチケット起票 or 直接相談 → 合意後に実装着手 → PR レビュー時にこの相談ログへの参照を残す。

## 対象ファイル
- `expense-saas/frontend/src/pages/reports/AttachmentUploader.tsx`

---

## 課題 1: ファイル選択ボタンが二重に表示されている

### 観測
明細スライドパネルの添付エリアに、ファイル選択ボタンが 2 つ並んで見える:
- **左**: ブラウザネイティブの「ファイルを選択」ボタン（OS / ブラウザによって文言は変わる）
- **右**: 「+ ファイルを追加」というテキスト

両方を押しても同じファイル選択ダイアログが開く。明らかに同じ機能を重複して表示している。

### 原因
`AttachmentUploader.tsx` L113-125 で `<label>` の中に生の `<input type="file">` を入れているが、その input を CSS で hidden 化していない:

```tsx
<label>
  <input
    ref={fileInputRef}
    type="file"
    accept={ALLOWED_MIME_TYPES.join(',')}
    data-testid="attachment-file-input"
    disabled={isPending}
    onChange={handleChange}
  />
  <span data-testid="attachment-upload-button">
    {isPending ? 'アップロード中...' : '+ ファイルを追加'}
  </span>
</label>
```

`<input type="file">` はブラウザがネイティブの「ファイルを選択」ボタンを描画するため、隣に `<span>+ ファイルを追加</span>` も表示されることで二重表示になる。

本来の意図は「input を視覚的に隠してラベルクリックでファイル選択ダイアログを開く」パターン（HTML / MUI の標準テクニック）だが、隠し処理が抜けている。

### 修正案

**案 A（推奨）: visually-hidden で input を隠す**

input に `style={{ display: 'none' }}` または MUI の `visuallyHidden` ユーティリティを適用する。

```tsx
<label>
  <input
    ref={fileInputRef}
    type="file"
    accept={ALLOWED_MIME_TYPES.join(',')}
    data-testid="attachment-file-input"
    disabled={isPending}
    onChange={handleChange}
    style={{ display: 'none' }}
  />
  <Button component="span" variant="outlined" disabled={isPending} startIcon={<AddIcon />}>
    {isPending ? 'アップロード中...' : 'ファイルを追加'}
  </Button>
</label>
```

「+」記号も Material Icon (`<AddIcon />`) に置き換えて MUI の Button に統一すると見た目が他のボタンと揃う。

**案 B: MUI 公式パターン（VisuallyHiddenInput）に揃える**

MUI v5+ のドキュメントで紹介されている `styled('input')({ ...visuallyHidden styles })` パターンを使う。アクセシビリティ対応も含まれる。

---

## 課題 2: スピナー / プログレス表示が実装されていない

### 観測
Browser Network Throttling（Slow 3G）でアップロードを意図的に遅延させても、表示されるのは「+ ファイルを追加」というボタンテキストが「アップロード中...」に切り替わるだけ。視覚的なスピナー（`<CircularProgress />`）やプログレスバーは表示されない。

### SMK-012 の期待結果との照合

`dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-012:

> アップロード進行中はプログレス or スピナーが表示され、完了後に一覧へ即時反映される

「プログレス or スピナー」と明示されており、テキスト変化のみでは厳密には未充足。

### 原因
`AttachmentUploader.tsx` L122-124 で `isPending` の表現が文言差し替えだけになっている:

```tsx
<span data-testid="attachment-upload-button">
  {isPending ? 'アップロード中...' : '+ ファイルを追加'}
</span>
```

`<CircularProgress />` や `<LinearProgress />` の組み込みがない。

### 修正案

課題 1 の修正と合わせて、Button の `startIcon` を `isPending` 時だけ `<CircularProgress size={16} />` に切り替える:

```tsx
<Button
  component="span"
  variant="outlined"
  disabled={isPending}
  startIcon={isPending ? <CircularProgress size={16} /> : <AddIcon />}
>
  {isPending ? 'アップロード中...' : 'ファイルを追加'}
</Button>
```

または、Button の下に `<LinearProgress />` を出してプログレスバー風に見せる。

---

## 完了条件

- 添付エリアにファイル選択を起動するボタンが**1 個だけ**表示されている
- アップロード中、ボタン内（または周辺）に**スピナーまたはプログレスバーが視覚的に表示**される
- 既存のテスト（`AttachmentUploader.test.tsx` 等）が通過する。`data-testid` の継続性に注意して書き換える
- 必要に応じてテストに「アップロード中スピナー表示」のアサーションを追加

## 関連
- 098: ItemSlidePanel の UX 不整合 — 同じ Drawer 内のもうひとつの UX 課題群、合わせて対応するとレビュー効率が上がる
- 099: useDeleteAttachment の invalidation 漏れ — 同じ添付機能領域のバグ

## 解決

PR #58 でスカッシュマージ（2026-04-16）。
- VisuallyHiddenInput（MUI 公式パターン）でネイティブ file input を隠蔽、ボタン重複解消
- isPending 時に CircularProgress をボタン内に表示（SMK-012 対応）
- DnD ゾーンを点線ボーダーで視覚化 + dragover 時の色変化フィードバック
- isPending 中の DnD を無効化（codex blocker 対応）
- dragleave バブリング対策（子要素間のちらつき防止）
- テスト ID を ATT-FE-051〜053 に振り直し（設計書と整合）
- テスト 16 件全 PASS

## 解決日
2026-04-16

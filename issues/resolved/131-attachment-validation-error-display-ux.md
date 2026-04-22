# AttachmentUploader のクライアントサイドバリデーションエラー表示を整理する（エラースタイリング + 文言整合）

## 発見日
2026-04-21

## カテゴリ
implementation / ui-design

## 影響度
中（エラー表示としての視覚識別が成立しておらず、SMK-033 が FAIL 判定。文言整合の問題も同時に内包）

## 発見経緯
user-report（Step 11-A SMK-033 / SMK-035 実施中にユーザーから指摘。両方 FAIL 判定）

SMK-035（2026-04-21 実施）の実観察結果:
- ファイル: 5,242,881 バイトの PDF を選択
- アップロード非実行: ✓（機能要件は満たす）
- エラー表示: 「ファイルサイズが上限（5MB）を超えています（5.0MB）」が「対応形式: JPEG, PNG, PDF（最大 5MB）」直下に**黒字通常テキスト**で表示（SMK-033 と同じパターン）
- SMK-035 期待結果「ファイルサイズは5MB以下にしてください」とも文言不一致

## 関連ステップ
Step 11-A（ローカル動作確認）/ Step 10-G（添付ファイル機能実装）/ Step 6-D（smoke_check.md）

## ブロッカー
なし（後追い改善、SMK-033 は機能要件「アップロード非実行」は満たすため Step 11-A 自体のブロッカーではない）

## 問題

### 現状の実装

`expense-saas/frontend/src/pages/reports/AttachmentUploader.tsx` のクライアントサイドバリデーションエラー表示:

```tsx
// L76-84: validateFile
function validateFile(file: File): string | null {
  if (!ALLOWED_MIME_SET.has(file.type)) {
    return '許可されていないファイル形式です（対応: JPEG, PNG, PDF）';
  }
  if (file.size > MAX_FILE_SIZE_BYTES) {
    return `ファイルサイズが上限（5MB）を超えています（${(file.size / 1024 / 1024).toFixed(1)}MB）`;
  }
  return null;
}

// L324-328: エラー表示
{validationError && (
  <div data-testid="attachment-validation-error" role="alert">
    {validationError}
  </div>
)}
```

### 問題点

#### 1. エラー表示として視覚識別できない
エラーメッセージが MUI コンポーネントを使わず素の `<div>` + テキストで表示されている。色指定・アイコン・背景色なし、通常の黒字テキストで描画される。

配置場所がヘルパーテキスト「対応形式: JPEG, PNG, PDF（最大 5MB）」（L321-323）の直下のため、**視覚的にヘルパーテキストと見分けがつかない**。ユーザーは「エラーが発生した」ことを認識しにくい。

`role="alert"` はセマンティックには正しいが、視覚的なアフォーダンスが完全に欠けている。

#### 2. ヘルパーテキストと情報が重複
- ヘルパー: 「対応形式: JPEG, PNG, PDF（最大 5MB）」
- エラー（拡張子）: 「許可されていないファイル形式です（対応: JPEG, PNG, PDF）」

エラー文言内に「対応: JPEG, PNG, PDF」を再掲しており、ヘルパーと冗長。

#### 3. SMK-033 期待結果との文言不一致

| 観点 | smoke_check.md SMK-033 期待結果 | 実装（AttachmentUploader.tsx:78） |
|------|-------------------------------|----------------------------------|
| 拡張子エラー | 「JPEG, PNG, PDF のみアップロード可能です」 | 「許可されていないファイル形式です（対応: JPEG, PNG, PDF）」 |

SMK-035（5MB 超過）についても:

| 観点 | smoke_check.md SMK-035 期待結果 | 実装（AttachmentUploader.tsx:81） |
|------|-------------------------------|----------------------------------|
| サイズエラー | 「ファイルサイズは5MB以下にしてください」 | 「ファイルサイズが上限（5MB）を超えています（Xx.xMB）」 |

いずれも #128 系（smoke_check 文言整合）と同じパターンで、実装と期待の乖離がある。

#### 4. ドロップエリア外への表示位置
エラーはドロップエリア（L286-320）の外・ヘルパーテキストの下に表示される。ユーザーの視線動線（ドラッグ操作 → ドロップエリア）から外れた位置にある。

## 影響

- UX: 中（エラー表示として機能していない、ユーザーがエラーに気づきにくい）
- SMK 整合: 中（SMK-033 FAIL、SMK-035 も同様の FAIL 可能性）
- 実害: なし（機能としてのバリデーションは動作、アップロードは実行されない）

## 提案

### 方針

クライアントサイドバリデーションエラー表示を MUI の正規エラー UI（Alert or Snackbar）に統一し、文言は smoke_check.md の期待結果に実装を合わせる。

### 変更スコープ

#### 実装修正

| ファイル | 変更内容 |
|---------|--------|
| `frontend/src/pages/reports/AttachmentUploader.tsx:76-84` | エラー文言を smoke_check.md に合わせる:<br>・MIME: 「許可されていないファイル形式です（対応: JPEG, PNG, PDF）」→「JPEG, PNG, PDF のみアップロード可能です」<br>・サイズ: 「ファイルサイズが上限（5MB）を超えています（...）」→「ファイルサイズは5MB以下にしてください」 |
| `frontend/src/pages/reports/AttachmentUploader.tsx:324-328` | エラー表示を MUI `<Alert severity="error">`（インライン）または `<Snackbar severity="error">`（トースト）に置き換え。本 issue では MUI Alert を優先案とする（位置・視認性・実装コストのバランス） |

#### 設計判断が必要な論点

**論点 A: エラー表示方式（Alert インライン vs Snackbar トースト）**
- **案 A-1**: MUI `<Alert severity="error">` をドロップエリア直下にインライン表示
  - 利点: 操作位置の近くでフィードバック、自動消去の複雑性なし
  - 欠点: ヘルパーテキストとの情報重複は残る（文言整合で緩和）
- **案 A-2**: MUI `<Snackbar severity="error">` でトースト表示
  - 利点: ユーザーの要望「トーストが綺麗」に合致、ヘルパーと物理的に分離
  - 欠点: 自動消去タイミング設計が必要、既存の Snackbar 運用（`AppToast`）との整合確認が必要

→ **推奨: 案 A-1**（AttachmentUploader 内部に閉じた実装で、既存 AppToast の運用ポリシーに干渉しない。ユーザーの「トーストが綺麗」という直観は満たせないが、Alert でも視覚的識別は十分改善される）

**論点 B: 文言の正本**
- **案 B-1**: 実装を smoke_check に合わせる（smoke_check が正本）
- **案 B-2**: smoke_check を実装に合わせる（#128 系の先例に反するが、文言的には実装のほうが情報量が多い）

→ **推奨: 案 B-1**（#128 と同じ運用。smoke_check を正本とし、実装を書き換える）

#### テスト更新

| ファイル | 変更内容 |
|---------|--------|
| `frontend/src/pages/reports/__tests__/AttachmentUploader.test.tsx` | エラー文言の変更を反映。MUI Alert の DOM 検証（`role="alert"` + `severity="error"` の class や icon 存在を検証） |

#### 設計書

本 issue は **実装と smoke_check の整合のみ**で、`screens/report-detail.md` / `files.md` に変更なし（バリデーション文言は設計書レベルで規定されていない）。

### 却下案

- **C: インライン + 赤字化のみ（MUI Alert 未使用）**: エラー識別はできるが、将来のスタイル統一性を損なうため却下
- **D: Snackbar を全面採用**: 既存 Snackbar 運用（`AppToast`）との整合検証が必要で、スコープが膨らむ。本 issue では保留

## 完了条件

- `AttachmentUploader.tsx` のバリデーションエラーが MUI Alert `severity="error"` で表示される（視覚的にエラーと識別可能）
- エラー文言が smoke_check.md SMK-033 / SMK-035 の期待結果と完全一致する
  - MIME エラー: 「JPEG, PNG, PDF のみアップロード可能です」
  - サイズエラー: 「ファイルサイズは5MB以下にしてください」
- SMK-033 / SMK-035 を再実施して両方 PASS になる
- 既存テストが通過 + 新しいエラー UI のアサーションが追加されている
- issue #128 と同じ「smoke_check 整合」扱いで、progress.md / review-findings との整合が取れている

## 関連

- **#128**（resolved）: smoke_check.md 文言整合（SMK-021/023/025/027/036/084）。本 issue は SMK-033/035 を同じパターンで追加整合する
- **#125**（resolved）: SMK-028 文言整合の先例
- **#123**（resolved）: SMK-026 文言整合の先例
- **#100**（resolved）: AttachmentUploader UI 改善（VisuallyHiddenInput + CircularProgress）。本 issue はエラー表示側の UI 改善を追加

## 発見時の元メッセージ（参考）

> ワーニングもしくはエラートーストメッセージとして表示した方がUIとしてきれいに思いました。
> 現状動作はインライン赤字表示ではなく普通に黒字

---

## 解決内容

PR #85（commit e06a521）にて対応。`AttachmentUploader.tsx` のバリデーションエラー表示を MUI `<Alert severity="error">` に変更し視覚識別を改善。エラー文言を smoke_check.md 期待値に合わせ、MIME エラーは「JPEG, PNG, PDF のみアップロード可能です」、サイズエラーは「ファイルサイズは5MB以下にしてください」に統一。SMK-033 / SMK-035 再検証 PASS。

## 解決日

2026-04-22

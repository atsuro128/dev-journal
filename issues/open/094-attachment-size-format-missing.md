# 添付ファイル一覧のファイルサイズが生バイト数で表示される（KB/MB フォーマット未実装）

## 発見日
2026-04-14

## カテゴリ
implementation / frontend / bug

## 影響度
中（機能は動作するが UX 品質低下）

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 / SMK-030〜032 実施中、添付ファイル一覧に表示されるサイズ値が `1024`、`2859424`、`2860118`、`4808655` のような生バイト数で、ユーザーにとって意味不明な数値表示になっていることを確認

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（機能は動作、UX のみ）

## 概要

設計書 `report-detail.md` §7.1 L326 は添付ファイル一覧のサイズ表示形式を `XX KB` / `X.X MB` と定義しているが、実装 `AttachmentList.tsx` は API が返すバイト数（`att.file_size`）をそのまま表示している。

## 事実

### 設計書

`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §7.1 L321-328:

| # | 項目 | 表示形式 |
|---|------|---------|
| 1 | ファイル名 | テキスト。クリックでダウンロード |
| 2 | ファイルサイズ | `XX KB` / `X.X MB` |
| 3 | 削除ボタン | `[x]` アイコン |

### 実装

`expense-saas/frontend/src/pages/reports/AttachmentList.tsx` L51-53:

```tsx
<span data-testid={`attachment-size-${att.id}`}>
  {att.file_size}
</span>
```

→ `att.file_size` は `number`（バイト数）のまま表示。フォーマット処理なし。

### 観察された表示例

| 実際の値 | 期待表示 |
|---------|---------|
| `1024` | `1 KB` |
| `2859424` | `2.7 MB` |
| `2860118` | `2.7 MB` |
| `4808655` | `4.6 MB` |

## 修正方針

### 案 A: AttachmentList.tsx 内で inline フォーマット

```tsx
const formatFileSize = (bytes: number): string => {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${Math.round(bytes / 1024)} KB`;
  return `${(bytes / 1024 / 1024).toFixed(1)} MB`;
};
// ...
<span data-testid={`attachment-size-${att.id}`}>
  {formatFileSize(att.file_size)}
</span>
```

### 案 B: 共通ユーティリティとして切り出す

- `expense-saas/frontend/src/utils/formatFileSize.ts` を作成
- 他の場所（将来の明細一覧の添付数表示等）で再利用可能

### 推奨

**案 B**。共通ユーティリティ化してテストも付けるのが保守性上望ましい。既存プロジェクト内の `utils/` ディレクトリ構造に従う。

## 修正対象ファイル

- 新規: `expense-saas/frontend/src/utils/formatFileSize.ts`
- 新規: `expense-saas/frontend/src/utils/__tests__/formatFileSize.test.ts`
- 修正: `expense-saas/frontend/src/pages/reports/AttachmentList.tsx` L51-53
- 修正: `expense-saas/frontend/src/pages/reports/__tests__/AttachmentList.test.tsx`（サイズ表示のアサーション追加）

## 完了条件

- 添付ファイル一覧のファイルサイズが `XX KB` / `X.X MB` 形式で表示される
- 1 KB 未満は `XX B`、1 MB 未満は `XX KB`、それ以上は `X.X MB` のような境界ロジック
- フォーマット関数の単体テストが追加され通過する
- 既存の AttachmentList テストが通過する
- 設計書 §7.1 と実装が一致する

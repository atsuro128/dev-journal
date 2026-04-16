# 添付ファイル削除の確認ダイアログが未実装（設計書定義済みなのに実装漏れ）

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / bug / design-implementation-gap

## 影響度
中（誤クリックで即削除される。データロスのリスク + 設計書との明確な乖離）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-038（添付削除）を実施中、smoke_check.md の手順「2. 確認ダイアログで『削除』」に従おうとしたが、削除ボタン押下で**確認ダイアログが出ず即削除された**。設計書を確認したところ、ダイアログは設計時点で定義されていたことが判明。

## 関連ステップ
Step 5（詳細設計）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項。ただし誤クリックでデータが消えるので Step 11-A 完了前に修正したい）

## 事実

### 設計書（正本）

`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §4.6（L364-368）:

```
### 添付削除の確認ダイアログ（screens.md 4.6 準拠）

| メッセージ | 確認ボタン | 対応UC |
|-----------|-----------|--------|
| 「この添付ファイルを削除しますか?」 | 「削除する」 | UC-M03a |
```

設計書では明確に確認ダイアログが定義されている。

### 実装

`expense-saas/frontend/src/pages/reports/AttachmentArea.tsx` L77-93:

```tsx
// 削除コールバック。
const handleDelete = (attachmentId: string) => {
  setDeletingId(attachmentId);
  deleteAttachment.mutate(
    { reportId, itemId, attId: attachmentId },
    {
      onSuccess: () => {
        setDeletingId(null);
        showToast('success', '添付ファイルを削除しました');
      },
      ...
    },
  );
};
```

`<AttachmentList>` の削除ボタンが押されると、`handleDelete` から直接 `deleteAttachment.mutate()` が呼ばれる。**確認ダイアログを挟む処理がない**。`grep -n "confirm\|Dialog"` でも該当箇所なし。

### 観測される挙動
- Member ロールで draft レポートの明細を開く
- 添付ファイル横の「削除」ボタンを押下
- **即座に削除される**（成功トーストが表示される、確認なし）

### 比較: 他の削除操作
他の削除操作（明細削除、レポート削除）には確認ダイアログが実装されているはず。要確認だが、添付削除だけが抜けている可能性が高い。

## 修正方針

### 1. 確認ダイアログを追加

MUI `<Dialog>` コンポーネントを使用、または共通ダイアログラッパー（`ConfirmDialog` 等が既存実装にあればそれを使う）。

```tsx
const [confirmDeleteId, setConfirmDeleteId] = useState<string | null>(null);

const handleDelete = (attachmentId: string) => {
  setConfirmDeleteId(attachmentId);
};

const handleConfirmDelete = () => {
  if (!confirmDeleteId) return;
  setDeletingId(confirmDeleteId);
  deleteAttachment.mutate(
    { reportId, itemId, attId: confirmDeleteId },
    {
      onSuccess: () => {
        setDeletingId(null);
        setConfirmDeleteId(null);
        showToast('success', '添付ファイルを削除しました');
      },
      onError: () => {
        setDeletingId(null);
        setConfirmDeleteId(null);
        showToast('error', '削除に失敗しました');
      },
    },
  );
};

// ...

<ConfirmDialog
  open={confirmDeleteId !== null}
  title="添付ファイルの削除"
  message="この添付ファイルを削除しますか?"
  confirmLabel="削除する"
  cancelLabel="キャンセル"
  onConfirm={handleConfirmDelete}
  onCancel={() => setConfirmDeleteId(null)}
/>
```

文言は設計書（report-detail.md §4.6 / screens.md 4.6）と完全一致させること。

### 2. 既存の共通ダイアログコンポーネントを確認

`expense-saas/frontend/src/components/ui/` 配下に `ConfirmDialog`/`AppDialog` 等があれば使う。なければ MUI Dialog を直接使うか、共通化を検討（ただし共通化は本 issue のスコープ外、別 issue へ）。

### 3. テスト追加

- `AttachmentArea.test.tsx` または `AttachmentArea.integration.test.tsx`:
  - 削除ボタン押下 → 確認ダイアログ表示の検証
  - 「削除する」押下 → mutate 呼び出しの検証
  - 「キャンセル」押下 → mutate 非呼び出しの検証

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`
- `expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.test.tsx` または `.integration.test.tsx`
- 既存の共通ダイアログを使う場合は別途追加読み込みファイルなし

## 完了条件

- 添付ファイル削除ボタン押下時に確認ダイアログが表示される
- ダイアログ文言が設計書（report-detail.md §4.6 / screens.md 4.6）と一致している
- 「キャンセル」で削除されない
- 「削除する」で削除される（成功 / 失敗時のトーストは現状維持）
- テストで確認ダイアログ経由フローが検証されている

## 関連
- 099: useDeleteAttachment の invalidation 漏れ — 同じ削除フローのバグ。本 issue と同じ PR でまとめて修正すると効率的
- screens.md 4.6 / report-detail.md §4.6: 確認ダイアログの設計上の正本

---

## 解決

**解決日**: 2026-04-15
**解決 PR**: #57 (`e55b66c`)

### 対応内容

- `AttachmentArea.tsx` の削除操作を共通 `ConfirmDialog` 経由の 2 段階フローに変更
- `title="添付ファイルの削除"` / `message="この添付ファイルを削除しますか?"` で設計書 `screens.md` 4.6 L231 と完全一致
- 既存の明細削除ダイアログパターン（ReportDetailPage）と同型の title + message 2 段構成

### codex 指摘と対応

初回レビューで「設計書は message のみ記載だが実装は title + message 追加文言で不一致」と指摘。設計書完全一致を優先して `message="削除した添付ファイルは元に戻せません。"` の追加文言を削除。

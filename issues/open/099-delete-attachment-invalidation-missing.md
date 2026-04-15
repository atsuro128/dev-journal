# 添付ファイル削除後のキャッシュ無効化が漏れており UI に削除済みファイルが残る

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / react-query / bug

## 影響度
中（明確なバグ。削除後の画面状態が DB と不整合になる。再削除を試みると 404 エラートーストが出る誤誘導 UX）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-038（添付削除）を実施中、Member ロールで draft レポートに添付した PDF を削除した際に発見。

## 再現手順
1. Member ログイン (`test-member@example.com` / `TestPass1!`)
2. draft レポートを開き、明細を選択して Drawer を開く
3. 添付ファイルを 1 つアップロード（成功トーストが出る）
4. 同じ添付ファイルの「削除」ボタンを押下 → 確認ダイアログで「削除」
5. **症状①**: 「添付ファイルを削除しました」成功トーストは出るが、UI 上は **ファイル名・容量・削除ボタンがそのまま残る**
6. **症状②**: 残ったままの削除ボタンを再度押下 → 「削除に失敗しました」エラートーストが出る（実体は既に DB から消えているので 404）
7. Drawer を閉じて開き直すと、確かに削除されている

## 関連ステップ
Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（バグだが致命的ではない。ただし誤誘導 UX なので Step 11-A 中に修正したい）

## 根本原因

`expense-saas/frontend/src/hooks/useDeleteAttachment.ts` の `onSuccess` で**添付一覧クエリのキーが invalidate されていない**。

### 該当コード（useDeleteAttachment.ts L31-34）

```ts
onSuccess: (_data, { reportId }) => {
  // レポート詳細のキャッシュを無効化する（items 配列に添付一覧が含まれるため個別 invalidation は不要）。
  void queryClient.invalidateQueries({ queryKey: ['reports', 'detail', reportId] });
},
```

コメントには「items 配列に添付一覧が含まれるため個別 invalidation は不要」と書かれているが、これは事実と乖離している。`AttachmentArea` は **専用の Query Hook (`useAttachments`)** を使って独立したクエリキーで添付一覧を取得しているため、レポート詳細キャッシュを invalidate しても画面上の添付一覧は更新されない。

### 該当箇所（useAttachments.ts L20-29）

```ts
export function useAttachments({ reportId, itemId }: UseAttachmentsParams) {
  return useQuery({
    queryKey: ['reports', reportId, 'items', itemId, 'attachments'],
    queryFn: async () => {
      return api.get<ApiResponse<Attachment[]>>(
        `/api/reports/${reportId}/items/${itemId}/attachments`,
      );
    },
  });
}
```

### 比較: useUploadAttachment は両方を invalidate している（正しい実装）

`useUploadAttachment.ts` L36-43:

```ts
onSuccess: (_data, { reportId, itemId }) => {
  void queryClient.invalidateQueries({ queryKey: ['reports', 'detail', reportId] });
  void queryClient.invalidateQueries({
    queryKey: ['reports', reportId, 'items', itemId, 'attachments'],
  });
},
```

アップロードは正しく両方無効化している。**削除側だけ漏れている**。

## 修正方針

### 1. useDeleteAttachment.onSuccess を修正

`useDeleteAttachment.ts` の onSuccess に添付一覧のキー invalidation を追加する。

```ts
onSuccess: (_data, { reportId, itemId }) => {
  void queryClient.invalidateQueries({ queryKey: ['reports', 'detail', reportId] });
  void queryClient.invalidateQueries({
    queryKey: ['reports', reportId, 'items', itemId, 'attachments'],
  });
},
```

合わせてコメントの誤った記述（「個別 invalidation は不要」）を修正する。

### 2. テスト追加

`useDeleteAttachment.test.tsx` に「削除成功後に attachments クエリが invalidate されること」のアサーションを追加する。useUploadAttachment のテストに同等の検証があるはずなので、それを参考に書く。

### 3. AttachmentArea.integration.test.tsx でも UI 反映の回帰テストを追加

削除ボタン押下後、UI から該当行が消えることを assert する統合テスト。これがなかったので本バグが検知されなかった。

## 修正対象ファイル

- `expense-saas/frontend/src/hooks/useDeleteAttachment.ts`（修正）
- `expense-saas/frontend/src/hooks/__tests__/useDeleteAttachment.test.tsx`（テスト追加）
- `expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx`（テスト追加）

## 完了条件

- 削除成功直後、UI から該当添付ファイル行が消える（Drawer の開き直し不要）
- 同じ削除ボタンを連打しても 404 エラートーストが出ない
- 上記回帰テストが追加され通過する

## 関連
- 069: 添付ファイルアップロード後の invalidation が設計テーブルと乖離 — 同根のキャッシュ無効化問題だが、上流側（設計テーブルとの整合）の論点。本 issue は実装の単純な漏れに焦点を絞る
- PR #51: ItemSlidePanel Drawer 化 — このバグは PR #51 の修正前から存在していた可能性が高い（useDeleteAttachment は変更されていない）

# 添付アップロード中の並行操作の整合性

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / ui / ux / state-management

## 影響度
中（致命ではないが、UX 破綻・添付反映の取りこぼし・トーストの座礁リスクあり）

## 発見経緯
issue 100（AttachmentUploader の UI 改善）の論点議論中にユーザーから「アップロード中に編集確定などの操作を行った場合を想定できているか」との指摘を受け、現状の state 管理を調査した結果、並行操作の整合性が担保されていないことが判明した。issue 100 のスコープを広げずに別枠で議論・対応するため独立 issue として起票。

## 関連ステップ
Step 8-7（共通 UI コンポーネント実装）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 問題

### 現状の state 管理
`AttachmentUploader.tsx` L56 の `isPending` は `useUploadAttachment` hook のローカル state で、**親コンポーネント（ItemSlidePanel / ItemForm）に伝播していない**。

```tsx
// AttachmentUploader.tsx
const { mutate, isPending } = useUploadAttachment();  // 完全に内部
```

一方、`ItemForm` の `isPending` は ItemSlidePanel から props で渡される別系統:

```tsx
// ItemSlidePanel.tsx
isPending?: boolean;  // ← フォーム保存中フラグ（AttachmentUploader とは無関係）
<ItemForm isPending={isPending} ... />
```

2 つの `isPending` は完全に独立しており、**AttachmentUploader のアップロード状態は ItemForm の「保存」ボタン disable に反映されない**。

### 想定される問題ケース

1. **アップロード中に「保存する」を押下**
   - 現状: 保存 API が走り、編集内容が保存される
   - アップロードはバックグラウンドで継続、成功後に添付一覧へ反映
   - ユーザーからは「保存したのに添付が一時的に見えない」と見える可能性

2. **アップロード中にパネルを閉じる**（キャンセル or IconButton）
   - React Query mutation は AbortController 未設定のため継続実行
   - onUploadSuccess コールバックは unmount 後なので呼ばれず、添付一覧の invalidate が発火しない可能性
   - 成功/失敗トーストがパネル外の無関係な画面で鳴る可能性

3. **アップロード中に別の明細を開く**
   - 同上、バックグラウンドで upload 継続
   - 2 つの明細間で誤った紐付けが起こる心配は現状なし（itemId が mutate 引数に固定されているため）が、トースト座礁は発生

4. **アップロード中にフォームを閉じる／新規明細作成に切り替える**
   - 同上、state のクリーンアップが不明

### 設計書との対応

- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §AttachmentUploader に「アップロード中の並行操作」に関する記述は現時点で確認できていない
- UX 規約（`dev-journal/deliverables/docs/01_ui_guidelines.md`）にも該当記述なし
- **設計書側にも仕様の欠如がある可能性**（要確認）

## 対応方針の候補

### 案 α: 並行操作を抑止（state lifting）
- AttachmentUploader から親 (ItemSlidePanel) に uploading 状態を伝播
- ItemSlidePanel 側で `isPending || isUploading` を OR 演算
- 保存ボタン・閉じるボタン・キャンセルボタン・別明細切り替えを disable
- **実装規模**: 中（state lifting + prop drilling）
- **UX**: 堅牢だが、ユーザーから見ると「一時的に何もできない」

### 案 β: 並行操作を許可するがクリーンアップ強化
- AbortController をセットしてアップロードを mutation 単位で cancelable に
- unmount 時に upload を cancel
- トーストの出し方を mutation 紐付けから global に変える（パネル閉じた後も正しく鳴る）
- **実装規模**: 中〜大
- **UX**: 自然（他画面の操作がブロックされない）だが、cancel 時にユーザーに「アップロードが中断された」通知が必要

### 案 γ: 確認ダイアログで段階的に対応
- アップロード中に保存・閉じる等を押したら `ConfirmDialog` で「アップロードを中断してよいですか？」を確認
- 「中断」を選ぶと案 β の AbortController 経路、「続行」を選ぶと操作キャンセル
- **実装規模**: 中
- **UX**: 最も明示的だが、ダイアログが頻出すると煩わしい

### 案 δ: 設計書先行
- まず設計書（report-detail.md §AttachmentUploader + ui_guidelines.md §ローディング）に「アップロード中の並行操作ポリシー」を明記
- 実装はその後に合意された方針で
- **実装規模**: 設計先行

## 修正対象ファイル（候補）

- `expense-saas/frontend/src/pages/reports/AttachmentUploader.tsx`
- `expense-saas/frontend/src/hooks/useUploadAttachment.ts`
- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx`
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md`（設計追記）
- `dev-journal/deliverables/docs/01_ui_guidelines.md`（設計追記、該当ファイル名は要確認）

## 完了条件

- 方針（案 α〜δ）が合意され、本 issue に追記されている
- 設計書に「アップロード中の並行操作」の方針が明記されている
- 4 つの想定ケースが設計書どおりに動作することを手動確認 or test で担保
- 必要ならフロント自動テストで並行操作 UX が検証されている

## 議論ログ（2026-04-16）

### 追加で発見された問題

1. **即時アップロード方式の UX 問題**: アップロードした時点で S3 + DB に即時永続化される。フォームをキャンセルしても添付は残る。ユーザーの「保存するまで反映されない」という期待と乖離
2. **新規明細で添付できない**: API が `POST /api/reports/:id/items/:itemId/attachments` で itemId 必須のため、未保存の新規明細では AttachmentArea が非表示。画面仕様（report-detail.md）はこの制限を明記しておらず設計漏れ
3. **108 のスコープが当初想定より広い**: 案 α（disable で抑止）では根本問題（即時永続化 + 新規明細で使えない）は解決しない

### 方針

別セッションで実際の動作を確認してから対応方針を決定する。

## 関連

- 100: AttachmentUploader UI 改善 — 本 issue の発見起点。100 はボタン見た目＋スピナーに限定し、並行操作は本 issue で対応
- 098: ItemSlidePanel UX 不整合 — panelState 単一 state 化を PR #56 で実装中。本 issue の案 α を採用する場合、panelState の拡張で対応する可能性あり
- 099: useDeleteAttachment invalidation 漏れ — 同じ添付機能領域、別ミューテーションでも同様の並行操作問題が潜在する可能性あり

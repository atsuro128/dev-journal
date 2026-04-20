# フォーム編集中の操作整合性（添付アップロード並行操作 + ItemForm 破棄確認）

## 発見日
2026-04-15（並行操作）/ 2026-04-18（ItemForm 破棄確認スコープ追加）

## カテゴリ
implementation / frontend / ui / ux / state-management

## 影響度
中（致命ではないが、UX 破綻・添付反映の取りこぼし・トーストの座礁リスクあり / 編集中の意図しない破棄リスクあり）

## 発見経緯
issue 100（AttachmentUploader の UI 改善）の論点議論中にユーザーから「アップロード中に編集確定などの操作を行った場合を想定できているか」との指摘を受け、現状の state 管理を調査した結果、並行操作の整合性が担保されていないことが判明した。issue 100 のスコープを広げずに別枠で議論・対応するため独立 issue として起票。

2026-04-18 のセッションで方針議論を行い、以下のスコープ調整を実施:
- 「即時アップロード UX」「新規明細で添付不可」を本 issue から分離（issue 114 / 115 として独立起票）
- ItemForm のフィールド編集中の破棄確認ダイアログを本 issue に統合（同じファイル群を触るため）

## 関連ステップ
Step 8-7（共通 UI コンポーネント実装）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## スコープ

本 issue で対応する 2 つの課題:

1. **添付アップロード中の並行操作整合性**
2. **ItemForm 編集中の破棄確認ダイアログ**

**スコープ外**（独立 issue）:
- 添付の永続化タイミング仕様明示 → **issue 114**
- 新規明細での添付方法 → **issue 115**

---

## 課題 1: 添付アップロード中の並行操作整合性

### 現状の state 管理
`AttachmentUploader.tsx` L77 の `isPending` は `useUploadAttachment` hook のローカル state で、**親コンポーネント（ItemSlidePanel / ItemForm）に伝播していない**。

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

加えて、`useUploadAttachment` の mutation には AbortController が設定されておらず、コンポーネント unmount 後もアップロード API が走り続ける。

### 想定される問題ケース

1. **アップロード中に「保存する」を押下**
   - 現状: 保存 API が走り、編集内容が保存される
   - アップロードはバックグラウンドで継続、成功後に添付一覧へ反映
   - ユーザーからは「保存したのに添付が一時的に見えない」と見える

2. **アップロード中にパネルを閉じる**（×ボタン / キャンセルボタン / 明細外クリック）
   - React Query mutation は AbortController 未設定のため継続実行
   - onUploadSuccess コールバックは unmount 後なので呼ばれず、添付一覧の invalidate が発火しない
   - 成功/失敗トーストがパネル外の無関係な画面で鳴る

3. **アップロード中に別の明細を開く**
   - 同上、バックグラウンドで upload 継続
   - itemId が mutate 引数に固定されているため明細間の誤紐付けは起きないが、トースト座礁は発生

### 対応方針（合意済み）

**α 採用 + β cleanup 部分採用** のハイブリッド:

1. **アップロード中は保存ボタン disable**（α 方式の最小適用）
   - AttachmentUploader から親（ItemSlidePanel）に uploading 状態を伝播
   - ItemSlidePanel 側で `isPending || isUploading` を OR 演算
   - 保存ボタンのみ disable（閉じるボタン・キャンセルボタンは常時有効）

2. **パネル閉じ時に AbortController でアップロード中断**（β cleanup 部分）
   - useUploadAttachment に AbortController を組み込む
   - unmount 時に upload を cancel
   - 中断時は「アップロードを中止しました」トーストを表示（mutation のエラーハンドリング経由）

3. **削除 mutation も同様の対応**（issue 099 と同領域）
   - useDeleteAttachment にも同パターン適用（AbortController + 親への状態伝播）

### 想定外（採用しなかった案）

- **β 全面採用**（操作完全許可 + トーストを global 化）: トースト座礁の根本対策だが、実装規模が大きく MVP 範囲外
- **γ 確認ダイアログ**: アップロード中の他操作で毎回ダイアログ → 煩わしい

---

## 課題 2: ItemForm 編集中の破棄確認ダイアログ

### 現状の挙動
ItemForm（明細編集フォーム）でフィールド編集中にパネルを閉じると、編集内容が **無確認で破棄** される。閉じるパターンは 4 つ:

1. パネル右上の × ボタン
2. キャンセルボタン
3. 明細外（Drawer のオーバーレイ）クリック
4. F5 リロード / タブ閉じ / ブラウザ閉じ

### 問題
誤操作で編集内容を失う UX リスク。特に明細外クリックは誤発火しやすい。

### 対応方針（合意済み）

**dirty 判定 + 4 パターン分の破棄確認**:

1. **dirty 判定対象**: ItemForm のフィールド変更のみ
   - 対象: 金額・日付・カテゴリ・摘要等のフォームフィールド
   - **対象外**: 添付の追加・削除（即時保存方式のため Form の dirty とは独立、issue 114 で仕様明示）

2. **閉じパターン別の実装**:

   | 閉じパターン | 実装 |
   |------------|------|
   | × ボタン | MUI Dialog（カスタム文言「変更を破棄しますか？」+ 「破棄/キャンセル」ボタン） |
   | キャンセルボタン | 同上 |
   | 明細外クリック（Drawer の onClose） | 同上 |
   | F5 / タブ閉じ / ブラウザ閉じ | `beforeunload` で**ブラウザ標準ダイアログ**（dirty 時のみ登録） |

3. **F5 等の技術制約**:
   - `beforeunload` のカスタム文言は現代ブラウザで不可（Chrome 51 / Firefox 4 / Safari 9.1 で廃止、セキュリティ理由）
   - 表示有無のみ `event.preventDefault()` で制御可能
   - keydown F5 傍受は網羅性に欠ける（ツールバー再読み込み・タブ閉じが拾えない）ため不採用
   - 詳細根拠は本 issue 末尾「技術調査ログ（F5 ダイアログ）」参照

### 採用しない案

- **添付操作も dirty に含める**: 即時保存方式の整合性が崩れる。添付の永続化タイミングは issue 114 で仕様明示する方針
- **keydown F5 傍受でカスタム MUI Dialog**: 網羅性不足で UX がぶれる

---

## 設計書との対応

- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6（明細スライドパネル）/ §7（添付ファイル管理）に「アップロード中の並行操作」「破棄確認ダイアログ」の仕様追記が必要
- `dev-journal/deliverables/docs/01_glossary.md` の用語集に「破棄確認ダイアログ」追加検討

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/AttachmentUploader.tsx`
- `expense-saas/frontend/src/hooks/useUploadAttachment.ts`
- `expense-saas/frontend/src/hooks/useDeleteAttachment.ts`
- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx`
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`
- `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md`（設計追記）

## 完了条件

- アップロード中に保存ボタンが disable されている（フィールド編集・閉じる操作は可能）
- パネル閉じ時にアップロード中の場合、AbortController でアップロードが中断され、中断トーストが表示される
- ItemForm のフィールド変更がある状態で × / キャンセル / 外クリックで閉じようとすると MUI Dialog で破棄確認が表示される
- F5 / タブ閉じ等で dirty 時にブラウザ標準ダイアログが表示される
- 破棄確認で「キャンセル」を選ぶとパネルは閉じず、編集内容が保持される
- 破棄確認で「破棄」を選ぶとパネルが閉じ、編集内容が破棄される
- 設計書（report-detail.md §6 §7）に上記仕様が明記されている
- 既存テスト（AttachmentArea.integration.test.tsx 等）が通過する
- 新規テスト（並行操作・破棄確認の振る舞い）が追加されている

## 関連

- 098: ItemSlidePanel UX 不整合 — panelState 単一 state 化を PR #56 で実装。本 issue の状態管理は panelState の拡張で対応する可能性あり
- 099: useDeleteAttachment invalidation 漏れ — 同じ添付機能領域、削除 mutation でも同様の並行操作問題が潜在。本 issue で同パターン適用
- 100: AttachmentUploader UI 改善 — 解決済み（PR #58）。本 issue の発見起点
- 114: 添付の永続化タイミング仕様明示 — 本 issue から分離
- 115: 新規明細での添付方法 — 本 issue から分離

## 技術調査ログ（F5 ダイアログ）

2026-04-18 のセッションで codex により独立調査を実施。結論:

| 項目 | 結論 |
|------|------|
| `beforeunload` のカスタム文言 | **不可**（Chrome 51 / Firefox 4 / Safari 9.1 で廃止。乱用・詐欺的 UI 対策） |
| 標準ダイアログ表示有無の制御 | **可**（`event.preventDefault()` で発火） |
| F5/タブ閉じ/ブラウザ閉じに MUI Dialog | **不可**（spec で beforeunload 中の alert/dialog 禁止） |
| keydown F5/Ctrl+R 傍受 | 部分的に可だがツールバー再読み込み等が拾えず網羅性なし → **不採用** |
| Service Worker / PWA | 不可（DOM 操作権限なし） |
| React Router useBlocker | SPA 内遷移には有効、hard reload は止められない |

参考:
- MDN `beforeunload`: https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event
- Chrome dialogs policy: https://developer.chrome.com/blog/dialogs-policy/
- Firefox Bugzilla #588292: https://bugzilla.mozilla.org/show_bug.cgi?id=588292

**実装の二段構え**（業界標準）:
- SPA 内遷移は MUI Dialog（カスタム）
- ブラウザ離脱は `beforeunload` の標準ダイアログ + 自動保存/復元

本 issue では自動保存/復元はスコープ外（MVP 後検討）。

## 次セッション着手手順（別セッションでワークツリー対応想定）

1. `/issue 対応 108` で本 issue を読み込む
2. 上流確認: `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6 §7、`expense-saas/frontend/src/pages/reports/{AttachmentUploader,AttachmentArea,ItemForm,ItemSlidePanel}.tsx`、`expense-saas/frontend/src/hooks/{useUploadAttachment,useDeleteAttachment}.ts`
3. 方針案を提示しユーザー合意（実装の細部）
4. `/implement issue-108` で frontend-developer をワークツリー起動（ブランチ名例: `fix/108-form-edit-integrity`）
5. PR 作成 → /test → reviewer → codex → マージ
6. 設計書修正（report-detail.md §6 §7）は同じ PR に含める（dev-journal リポジトリは別途コミット）

並列化: 本 issue は AttachmentUploader/ItemForm 周辺を全面的に触るため、issue 114（設計書のみ）/ 115（AttachmentArea + 新規モード分岐）と**ファイル重複あり**。原則的には 108 → 114/115 の順で対応するか、114（設計書だけ）を先に独立対応する。

# 明細保存成功後も beforeunload リスナが残存し、F5 でリロード確認ダイアログが表示される

## 発見日
2026-04-21

## カテゴリ
implementation

## 影響度
中（保存したのにリロードで警告される UX バグ。ユーザーが「保存が成功したか不安になる」）

## 発見経緯
user-report（Step 11-A SMK-034 実施後のリロード動作確認中にユーザーから指摘）

## 関連ステップ
Step 11-A（ローカル動作確認）/ Step 10-G（添付ファイル機能実装）/ issue #108（フォーム編集中の操作整合性）

## ブロッカー
なし

## 問題

### 再現手順
1. Member で draft レポート → 「明細追加」または既存明細「編集」で ItemSlidePanel を開く
2. 必須フィールドを入力 → 「保存する」押下 → 保存成功トースト表示・Drawer 閉じる
3. レポート詳細画面（明細一覧が見えている状態）で **F5** を押下
4. → ブラウザ標準のリロード確認ダイアログが表示される（期待: そのまま再読込されるべき）

### 原因（コード特定済み）

`expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx:504-517`:

```tsx
useEffect(() => {
  if (!isDirty) return;
  const handleBeforeUnload = (event: BeforeUnloadEvent) => event.preventDefault();
  window.addEventListener('beforeunload', handleBeforeUnload);
  return () => window.removeEventListener('beforeunload', handleBeforeUnload);
}, [isDirty]);
```

`isDirty` の定義（L201-203）:
```tsx
const isDirty = mode === 'add' ? isFormDirty || pendingFiles.length > 0 : isFormDirty;
```

### リセット漏れ

破棄経路（`handleDiscard`、L463-473）では以下のリセットが走る:
```tsx
formResetRef.current?.();
setIsFormDirty(false);
setPendingFiles([]);
onClose();
```

**しかし保存成功経路（`handleAddModeSubmit` / `handleSubmit`）にはこれらのリセットがない**。`afterSubmit()`（= `onSaveSuccess`）を呼んで親に Drawer クローズを指示するだけで、ItemSlidePanel 内部の state (`isFormDirty`、`pendingFiles`) はそのまま残る。

MUI Drawer は `open=false` でもコンテンツをアンマウントしないため、state は残存し:
1. 保存成功 → Drawer 閉じる
2. `isDirty === true` のまま
3. `beforeunload` リスナ継続登録
4. F5 → ブラウザ標準のリロード確認ダイアログ表示

### 設計書との乖離

`screens/report-detail.md` §6（破棄確認ダイアログ仕様）および issue #108 の方針は「**dirty 時のみ** beforeunload を発動」としている。保存成功後は dirty ではないはずなので、本挙動は設計逸脱。

## 影響

- UX: 中（保存成功後にリロード警告が出ることで「保存が成功したか不安になる」）
- 設計整合: 中（#108 で合意した「dirty 時のみ」仕様からの逸脱）
- データ: なし（実害なし）
- セキュリティ: なし

### 副次的影響（ブラウザ anti-abuse 挙動）

ユーザーが同一セッション内で複数回 beforeunload を経験すると、Chromium 系ブラウザは「**このページで追加のダイアログを作成できないようにする**」チェックボックス付きのダイアログを表示する。これは本 issue の挙動によって beforeunload が意図せず頻発することが原因。

**このチェックボックス自体はブラウザの仕様**:
- 同一オリジンで `beforeunload` / `alert()` / `confirm()` / `prompt()` が短時間に繰り返し発生した際、Chromium 系（Chrome / Edge）が anti-abuse 目的で表示する
- ユーザーがチェックを入れると、当該ページからの以降のすべてのダイアログが抑制される（セッション終了まで）
- 仕様準拠であり、アプリケーション側から制御・抑制は不可能
- 参考: HTML Living Standard - [Unloading documents: prompt to unload a document](https://html.spec.whatwg.org/multipage/browsing-the-web.html#unloading-documents)

本 issue を修正すれば beforeunload の誤発動がなくなるため、このチェックボックスが表示される状況自体が解消される。

## 提案

### 修正方針

保存成功経路で `isFormDirty` と `pendingFiles` をリセットする。破棄経路と同等の cleanup を `afterSubmit()` 直前に実行する。

### 変更スコープ

| ファイル | 変更内容 |
|---------|--------|
| `frontend/src/pages/reports/ItemSlidePanel.tsx:300-417`（`handleAddModeSubmit`） | 全成功・部分失敗いずれの分岐でも `afterSubmit()` 呼び出し前に以下を実行:<br>・`setIsFormDirty(false)`<br>・`setPendingFiles([])`<br>・`formResetRef.current?.()` |
| `frontend/src/pages/reports/ItemSlidePanel.tsx:419-430`（`handleSubmit`、編集モード経路） | 編集モードでも保存成功後に `setIsFormDirty(false)` / `formResetRef.current?.()` を実行。ただし編集モードの保存は親側（`onItemSubmit` コールバック）経由で mutation するため、成功ハンドラをどこに配置するかは実装時に検討（親の `onSaveSuccess` 内 or ItemSlidePanel 側の props 追加）|
| `frontend/src/pages/reports/ItemSlidePanel.tsx:432-449`（`handleSaveAndContinue`） | 「保存して続けて追加」は保存成功後に続けて入力できるようリセットするが、`pendingFiles` / `isFormDirty` も含める必要あり（既存挙動を確認） |

### テスト追加

| ファイル | 追加テスト |
|---------|----------|
| `frontend/src/pages/reports/__tests__/ItemSlidePanel.integration.test.tsx` | - 追加モード: 保存成功後に `window.removeEventListener('beforeunload', ...)` が呼ばれる（`isDirty` が false になる）<br>- 編集モード: 同上<br>- 保存して続けて追加: リセット挙動が妥当か（再入力のために `isDirty` は一旦 false、再入力で true に戻る） |

### 想定修正規模
コード 10〜20 行、テスト 3 ケース前後。

## 完了条件

- 追加モードで保存成功後、F5 を押してもブラウザ標準のリロード確認ダイアログが表示されない
- 編集モードで保存成功後、F5 を押してもブラウザ標準のリロード確認ダイアログが表示されない
- 「保存して続けて追加」は引き続き想定どおり動作する（入力継続可能）
- 破棄経路（キャンセル・×・外クリック時の破棄ダイアログ「破棄」）は引き続き想定どおり動作する
- 既存テストが通過 + 回帰テスト追加

## 関連

- **#108**（resolved）: フォーム編集中の操作整合性・破棄確認ダイアログ。本 issue の beforeunload リスナはここで導入された。破棄経路のリセットは実装されたが保存経路のリセットが漏れている
- **#115**（resolved）: 新規追加モードの local buffer 方式。`pendingFiles` state はここで導入された
- **#129**（open）: 新規追加モードの添付 UX 改善。`pendingFiles` 周りの UI を変更するため、本 issue の修正と同時対応も可能
- **#130**（open）: パネル閉じ時のスライドアニメーション中にレイアウトが切り替わる問題。同じく ItemSlidePanel の state リセットタイミング系課題

---

## 解決内容

## 解決日

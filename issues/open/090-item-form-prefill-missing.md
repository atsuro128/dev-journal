# 明細編集/詳細時にフォームに既存値がプリフィルされない（ItemForm 再マウント漏れ）

## 発見日
2026-04-14

## カテゴリ
implementation / frontend / bug

## 影響度
**高 / blocker**

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 前提データ準備のため、明細を追加した後、既存明細を編集モードまたは詳細モードで開いた際に、フォームの各フィールドが空のまま表示されることを確認

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
**あり** — 明細編集機能が事実上機能していない。SMK-012 以降の明細関連 SMK（SMK-020〜023, 030〜038, 071 等）の検証がブロックされる

## 概要

明細行の「編集」ボタン押下時、または明細行クリック（閲覧モード）時に、スライドパネルのフォームに既存値（日付・金額・カテゴリ・摘要）がプリフィルされず、空のまま表示される。設計（`report-detail.md` §5 L209）は「既存データをプリフィルして開く」を明示しており、仕様乖離。

## 根本原因

1. `ItemSlidePanel.tsx:105` は `style={{ display: open ? 'block' : 'none' }}` で表示切替のみ、**コンポーネントを再マウントしない**
2. `ItemForm.tsx:89-98` の `useForm({ defaultValues })` は react-hook-form の仕様上、**初回マウント時のみ** 初期値を読む
3. `ReportDetailPage.tsx:517` の `<ItemSlidePanel key={formKey}>` の `formKey` は「保存して続けて追加」成功時（L454）のみインクリメントされる。`handleItemClick`（L348-354）や `handleEditItem`（L359-365）では `formKey` を変えない
4. その結果、`selectedItem` state が変わっても ItemForm が再マウントされず、フォームは初期値（空）のまま

## 事実

- 実装箇所:
  - `expense-saas/frontend/src/pages/reports/ItemForm.tsx` L89-98 — `useForm({ defaultValues })` は初回のみ評価
  - `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx` L102-130 — `display: none/block` で再マウントなし
  - `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` L348-365 — handleItemClick / handleEditItem が formKey を更新しない
  - `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` L517 — `<ItemSlidePanel key={formKey}>` が「保存して続けて追加」時のみ再マウント
- 仕様: `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` L209「明細編集 … スライドパネルに既存データをプリフィルして開く」

## 修正方針

### 案 A: handleItemClick / handleEditItem で formKey をインクリメント（最小変更）

```tsx
const handleItemClick = (itemId: string) => {
  const item = report.items.find((it) => it.id === itemId) ?? null;
  setSelectedItem(item);
  setPanelMode('view');
  setItemApiError(null);
  setFormKey((prev) => prev + 1);  // ← 追加
  setPanelOpen(true);
};

const handleEditItem = (itemId: string) => {
  const item = report.items.find((it) => it.id === itemId) ?? null;
  setSelectedItem(item);
  setPanelMode('edit');
  setItemApiError(null);
  setFormKey((prev) => prev + 1);  // ← 追加
  setPanelOpen(true);
};
```

### 案 B: ItemForm 内で useEffect + reset で対応

```tsx
useEffect(() => {
  if (defaultValues) {
    reset(defaultValues);
  }
}, [defaultValues]);
```

### 推奨

**案 A** を推奨。シンプルで、react-hook-form の `useForm` 初期化戦略を変えない。formKey の役割が「フォーム状態をリセットしたいとき」に統一される。

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` — handleItemClick / handleEditItem で formKey 更新
- `expense-saas/frontend/src/pages/reports/__tests__/ReportDetailPage.test.tsx` — 明細クリック / 編集時にフォームが既存値でプリフィルされるテストを追加

## 完了条件

- 明細行をクリック（閲覧モード）したとき、フォームに既存値が表示される
- 明細の「編集」ボタンを押下したとき、フォームに既存値が表示される
- 別の明細行に切り替えたとき、フォームが新しい明細の値に更新される
- 追加モードから編集モードに切り替えたとき、追加モードのフォーム状態が引きずられない
- 既存テストが通過し、新規テスト（プリフィル検証）が追加されて通過する
- SMK-012 以降の明細関連 SMK が実施可能になる

## 関連 issue

- 089（カテゴリラベル重複）— 同じ `ItemForm.tsx` 周辺。バンドル PR 推奨
- 091（draft 所有者の明細行クリック時の挙動未定義）— 本 issue の修正で行クリック時の挙動が変わるため、091 の設計判断を先行させるか同時に対応するか要検討

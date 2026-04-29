# ワークフロー確認ダイアログ閉じる際に「支払完了を記録しますか？」が一瞬ちらつく（三項演算子フォールバック起因）

## 発見日
2026-04-28

## カテゴリ
implementation / frontend / ui

## 影響度
中（機能影響なし。ユーザーが「別のメッセージが一瞬見える」違和感を覚え、UX バグとして認識される）

## 発見経緯
user-report / Step 11-A SMK-011 検証時、Approver で承認確認ダイアログを開いた状態からキャンセル（または外側押下）でダイアログを閉じると、閉じるアニメーション中に一瞬「このレポートの支払完了を記録しますか？」というメッセージがちらつく現象をユーザーが報告

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 該当箇所

`expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx:621-629`:

```tsx
{/* ワークフロー操作確認ダイアログ（承認・却下・支払完了） */}
<ConfirmDialog
  open={workflowDialogAction !== null}
  message={
    workflowDialogAction === 'approve'
      ? 'このレポートを承認しますか？'
      : workflowDialogAction === 'reject'
        ? 'このレポートを却下しますか？'
        : 'このレポートの支払完了を記録しますか？'
  }
  ...
/>
```

`workflowDialogAction` の型は `'approve' | 'reject' | 'pay' | null`（同ファイル L39）。

### 根本原因

三項演算子の最後の分岐（pay 用文言）が **`'pay'`** だけでなく **`null`** にもフォールバックする実装になっている:

- `workflowDialogAction === 'approve'` → 「承認しますか？」
- `workflowDialogAction === 'reject'` → 「却下しますか？」
- それ以外（`'pay'` **または** `null`）→ 「支払完了を記録しますか？」

ダイアログを閉じる際の流れ:
1. ユーザーがキャンセル / 外側クリック
2. `onCancel` ハンドラで `setWorkflowDialogAction(null)` を実行
3. React 再レンダリングで `open={workflowDialogAction !== null}` が `false` になり、MUI Dialog が閉じるアニメーション開始
4. アニメーション中（数百ミリ秒）はダイアログ DOM が残ったまま、`message` プロパティは **「支払完了を記録しますか？」**（null フォールバックの結果）に更新済み
5. ユーザーには元の文言（例: 「承認しますか？」）から「支払完了を記録しますか？」に切り替わったように見える

### 影響範囲

- 承認ダイアログ → キャンセル: 「支払完了」が一瞬見える ✓ ユーザー報告通り
- 却下ダイアログ → キャンセル: 同様に「支払完了」が一瞬見える（推定、要検証）
- 支払完了ダイアログ → キャンセル: 元から「支払完了」のためちらつき不可視（バグは存在するが目立たない）

## 影響

- ユーザーが意図せず「支払完了」というワークフロー終端のメッセージを目にし、操作の文脈と矛盾するため UX バグとして認識される
- Approver は支払完了操作の権限がないため「自分には関係のないメッセージが出た」と混乱する可能性

## 提案

### 修正案A: pay を明示的に分岐 + null 時は空文字（最小変更）

```tsx
message={
  workflowDialogAction === 'approve'
    ? 'このレポートを承認しますか？'
    : workflowDialogAction === 'reject'
      ? 'このレポートを却下しますか？'
      : workflowDialogAction === 'pay'
        ? 'このレポートの支払完了を記録しますか？'
        : ''
}
```

メリット: 変更が局所的で副作用なし。`null` 時に空文字を出してもアニメーション中は視覚上ほぼ問題なし
デメリット: 空文字によるレイアウト揺れの可能性（要 DOM 検証）

### 修正案B: 表示用文言を別 state で保持（推奨）

```tsx
const [dialogMessage, setDialogMessage] = useState('');

// 各ハンドラで開く際に文言を設定
const handleApprove = () => {
  setDialogMessage('このレポートを承認しますか？');
  setWorkflowDialogAction('approve');
};
// reject, pay も同様

// ダイアログ message には dialogMessage を直接渡す（state なので閉じても保持される）
<ConfirmDialog message={dialogMessage} ... />
```

メリット: 閉じるアニメーション中も最後に開いた際の文言が保持され、ちらつきが完全に消える
デメリット: 状態管理が増える（軽微）

### 修正案C: ConfirmDialog 共通コンポーネント側で対応

`ConfirmDialog` 内部で `message` prop を `useDeferredValue` または `usePrevious` 相当のロジックで保持し、`open=false` の間は前回値を維持する。

メリット: 同種のちらつきを他箇所でも防げる
デメリット: 共通コンポーネント側の変更で影響範囲が広がる

### 推奨

修正案B が最もシンプルかつ確実。修正案C は他箇所での同種バグが既に複数あれば検討する価値あり。

## 修正対象ファイル

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` | 修正案 A/B/C のいずれかを実装 |
| `expense-saas/frontend/src/pages/reports/__tests__/ReportDetailPage.test.tsx` | 閉じるアニメーション中の表示文言保持テストを追加（可能な範囲で） |

## 完了条件

- 承認ダイアログをキャンセルしても「支払完了」メッセージが一瞬も表示されない
- 却下ダイアログをキャンセルしても同上
- 既存ダイアログ操作テストが通る + 可能であれば新規テストでちらつき防止を検証

## ラベル

- type: bug / ui
- area: frontend

## 関連

- 関連 SMK: SMK-011（二重押下防止）の副次発見
- 関連設計書: `dev-journal/deliverables/docs/55_ui_component/common-components.md` の `ConfirmDialog` 仕様

## MVP 区分

MVP（UX バグ、ユーザーが直接目にする）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

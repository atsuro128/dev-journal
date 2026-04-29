# 却下確認ダイアログの未入力バリデーションエラー文言が表示されない（設計 D4 規定の「却下理由を入力してください」未実装）

## 発見日
2026-04-29

## カテゴリ
implementation / frontend / ui

## 影響度
中（機能影響なし。送信は防がれている。ただし「なぜ押せないか」がユーザーに伝わらず、設計書のフォームバリデーション規定と乖離）

## 発見経緯
user-report / Step 11-A SMK-096 検証時、却下理由を空のまま入力欄をブラー（フォーカスアウト）しても「却下理由を入力してください」のエラー文言が表示されず、「却下する」ボタンが disabled になるのみであることを発見

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書の規定

`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md:159`:

| ダイアログ | 項目 | バリデーション | エラー文言 | 関連テストID |
|-----------|------|--------------|----------|------------|
| D4（却下） | 却下理由 | 空でないこと | **「却下理由を入力してください」** | WFL-012 |
| D4（却下） | 却下理由 | 1000文字以内 | 「却下理由は1000文字以内で入力してください」 | WFL-012 |

→ 必須項目未入力時に赤字エラー文言を表示する規定。

加えて `report-detail.md:669`:

| サーバーエラー | UI 表示 | ダイアログ状態 |
|--------------|--------|------------|
| MissingRejectionReason（422） | 却下ダイアログ内に「却下理由を入力してください」 | ダイアログを開いたまま |

→ サーバー 422 時にも同じ文言を表示する規定。

### 実装の挙動（ユーザー手動検証で確認）

- 却下ダイアログを開き、入力欄を空のままフォーカス・ブラー → **エラー文言なし**
- ボタンが disabled になっているのみ
- 「却下理由を入力してください」のエラー文言は画面のどこにも出ない

### 影響範囲

- 却下ダイアログ（必須項目）— 本 issue の対象
- 承認コメント / 支払完了確認は任意項目のためエラー文言不要 → 対象外

### 該当箇所（推定）

- `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx`（workflow ダイアログの `inputField` 設定。required: true で却下理由を渡している箇所）
- `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx`（共通ダイアログ。`inputField.required` 時のバリデーション・エラー表示ロジック）

`55_ui_component/common-components.md:225-231` でも `inputField.required` の規定はあるが、エラー文言表示の実装が伴っていない可能性。

## 影響

- ユーザーが「なぜ却下するボタンが押せないのか」を画面から判断できない
- スクリーンリーダー使用者には disabled の理由が一切届かない（aria 属性のみで、ライブリージョン等のアナウンスなし）
- 設計書のフォームバリデーション規定（V1/V2 等の他フォームでは正しく実装されている）との一貫性が崩れる
- 過去の同種フォーム（レポート作成・明細追加等）はエラー文言を表示しており、却下ダイアログだけ未実装

## 提案

### 修正案A: ConfirmDialog 共通コンポーネントで対応（推奨）

`ConfirmDialog` の `inputField.required: true` 時、以下の挙動を実装:

1. 入力欄を一度フォーカスして空のままブラー（onBlur）した場合、`helperText`（赤字 / `error: true`）として `errorMessage` または規定文言「却下理由を入力してください」を表示
2. 確認ボタン（disabled）の維持はそのまま
3. 入力が始まった瞬間にエラー文言を消す（`onChange` で error 状態解除）

```tsx
// ConfirmDialog.tsx の inputField 部分（推定）
const [touched, setTouched] = useState(false);
const isInvalid = inputField?.required && touched && !value.trim();

<TextField
  value={value}
  onChange={(e) => { setValue(e.target.value); }}
  onBlur={() => setTouched(true)}
  error={isInvalid}
  helperText={isInvalid ? inputField.errorMessage ?? '却下理由を入力してください' : ''}
  required={inputField.required}
  multiline
  rows={3}
/>
```

メリット: 共通ダイアログ側で対応 → 他必須項目（将来追加）にも自動適用
デメリット: ConfirmDialog の API 仕様変更（`errorMessage` プロパティ追加）

### 修正案B: ReportDetailPage 側でカスタムバリデーション実装

`ReportDetailPage.tsx` で却下ダイアログのみ専用 state を持ち、エラー表示制御を行う。

メリット: ConfirmDialog の責務を増やさない
デメリット: 各画面でバリデーション制御を再実装する必要

### 修正案C: ボタン enable + 送信時エラー（設計書通り）

ボタンを常に enable に戻し、押した時にバリデーションを発火 → エラー文言表示。

メリット: 設計書 L159 の意図に最も近い
デメリット: 実装の安全策（disabled）を捨てる形になる

### 推奨

**修正案A**。「ボタン disabled + ブラー時エラー文言」の二段構えが UX 上ベスト:
- ボタン disabled で物理的に誤送信を防止
- エラー文言で「何が問題か」をユーザーとスクリーンリーダーに伝達

設計書側も「ボタン disabled も併用してよい」と追補することで、設計と実装を整合させる。

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx` | inputField.required 時の onBlur バリデーション + helperText エラー表示追加 |
| `expense-saas/frontend/src/components/ui/__tests__/ConfirmDialog.test.tsx` | required + ブラー時のエラー文言表示テスト追加 |
| `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` | inputField に errorMessage 追加（必要に応じて） |
| `expense-saas/frontend/src/pages/reports/__tests__/ReportDetailPage.test.tsx` | 却下ダイアログのバリデーション挙動テスト更新 |
| `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` | D4 規定にボタン disabled の併用を追補（設計確認後） |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` | ConfirmDialog inputField の errorMessage / required 挙動追補 |

## 完了条件

- 却下ダイアログで却下理由欄を空のままブラーすると「却下理由を入力してください」エラー文言が表示される
- 入力すると同時にエラー文言が消える
- ボタン disabled は併用維持（誤送信防止）
- 既存テスト通過 + 新規テスト追加
- 設計書（report-detail.md / common-components.md）と実装が整合している

## ラベル

- type: bug / ui / validation
- area: frontend / docs（設計書整合）

## 関連

- 設計書: `50_detail_design/screens/report-detail.md` §フォームバリデーション §バリデーションエラー対応
- 関連 SMK: SMK-096（却下確認ダイアログの理由必須バリデーションと却下実行）の手順 4-5 部分が FAIL → 本 issue 起票

## MVP 区分

MVP（フォームバリデーションの基本要件、ユーザーに「なぜ押せないか」を伝える UX 影響）

---

## 解決内容

**採用方針**: 案 A（ConfirmDialog 共通コンポーネント側で onBlur + helperText 対応） + D4 規定対応（却下ダイアログのみ apiError パターン）

**実装** (PR #109, commits 683c076 / 0f23fc0 / c8167d0):
- `ConfirmDialog` の `InputFieldConfig` に `errorMessage?: string` オプショナル prop を追加
- 内部 state `touched`、onBlur で touched=true、`!value.trim()` のとき helperText に赤字エラー文言（`inputField.errorMessage`）を表示
- onChange で値が空でなくなったら touched=false 解除
- ボタン disabled は併用維持（誤送信防止の二段構え）
- `ReportDetailPage` 却下 inputField に `errorMessage: '却下理由を入力してください'` を指定

**D4 規定（MissingRejectionReason 422）対応**:
- 却下ダイアログのみ apiError パターン採用（`rejectDialogApiError` state、`apiError` prop で FormAlert 表示、`open=true` 維持）
- 422 サーバーエラー時もダイアログ内に文言表示し再試行可能
- 他ダイアログ（承認/支払/明細削除/添付削除/提出/レポート削除）はアプリ全体で established な旧トーストパターンを維持

**設計書改訂** (commits de49d82 / d2a90f8):
- `55_ui_component/common-components.md` ConfirmDialog 内部仕様 §必須入力バリデーション 新設、§API エラー表示 を「却下ダイアログのみ」に限定明記
- `50_detail_design/screens/report-detail.md` D4 行に「ボタン disabled 併用」追補、§11 冒頭ノートで「却下のみ apiError、他は旧トースト」明記

**スコープ調整の経緯**: 当初「全 7 ダイアログ apiError 統一」に拡大していたが、ユーザー判断（案 C）で却下のみに巻き戻し。設計書 D4 規定の本来のスコープ（required input validation）に整合させた。

**追加 W1 対応** (commit 5120cf9): ConfirmDialog の `loading` prop に各 mutation の `isPending` を連動、確認/キャンセル/外側クリックを止めることで多重送信防止 + SMK-011 規定準拠。

**テスト追加**:
- WFL-FE-064-H/I: 却下ダイアログ空ブラー時のエラー文言表示・入力で解除
- WFL-FE-064-J/K: 却下 422 時のダイアログ open 維持 + apiError 表示
- SMK-011-W1-A1〜E2 / ATT1/ATT2: 全ダイアログの loading 連動による二重押下防止

## 解決日

2026-04-30

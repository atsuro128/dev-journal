# issue #162: 却下確認ダイアログ初期表示時のバリデーションエラー / ボタン disabled regression

- 担当: frontend-developer（FE 実装） + 指揮役（設計書改訂を dev-journal/master 直接編集）
- 依存: なし。第3バッチ G1（#154+#160 AppDataGrid）/ G3（#164 DashboardPage）と並列実行可能（修正対象ファイルが独立）
- ブランチ: `step11/162-reject-dialog-initial-state-fix`（expense-saas）
- 出力先:
  - 設計書改訂: `dev-journal/deliverables/docs/55_ui_component/common-components.md` §ConfirmDialog 内部仕様（dev-journal/master 直接編集）
  - FE 実装: `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx`（worktree 隔離ブランチ）
  - テスト追加・更新: `expense-saas/frontend/src/components/ui/__tests__/ConfirmDialog.test.tsx`
- 元資料: `dev-journal/issues/open/162-reject-dialog-initial-validation-error-and-disabled-button.md`（採用方針: 「責務再分離」推奨方針）
- テンプレート: `ai-dev-framework/templates/ticket-template.md`

## 入力

| 資料 | パス | 参照箇所 | 用途 |
|------|------|----------|------|
| issue 本文 | `dev-journal/issues/open/162-reject-dialog-initial-validation-error-and-disabled-button.md` | 全文（特に「現象」「期待動作」「推奨方針」「完了条件」） | 採用方針・受け入れ条件の確定 |
| 既存実装（FE） | `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx` | L109-122（state 初期化と `useEffect` reset）、L144-147（`isInvalid` の `touched` 依存）、L164-166（`isConfirmDisabled` の disabled 条件）、L201-207（`error`/`helperText` の描画条件） | 修正対象。`isConfirmDisabled` が `touched` を介していないため初期から disabled になっている、および `usePrevious` で保持される `prevInputField` 経由で initial render に旧 inputField 状態が混入する可能性の確認 |
| 既存実装（FE 呼び出し側） | `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` | L284-289（却下ボタン押下処理 `setRejectDialogApiError(null)` リセット）、L676-732（ConfirmDialog 呼び出し、却下時 `apiError={rejectDialogApiError}`） | 呼び出し側の `apiError` 受け渡しが正しく `null` 初期化されていることの確認（変更不要であれば触らない） |
| 既存テスト | `expense-saas/frontend/src/components/ui/__tests__/ConfirmDialog.test.tsx` | 全体 | 既存テストの破壊的変更回避 + 初期表示挙動テストの追加 |
| 設計書（共通コンポーネント） | `dev-journal/deliverables/docs/55_ui_component/common-components.md` | §ConfirmDialog L208-269（特に L218-222「必須入力バリデーション」、L224「API エラー表示」） | 初期状態仕様（初期表示でエラー文言なし・ボタン押下可能）の明示追加 |
| 設計書（画面詳細） | `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` | §11 D4 MissingRejectionReason 規定 | 初期表示挙動の補足が必要かの確認（API パターン側の規定と矛盾しないか） |
| 過去 PR / コミット | PR #109, commit `5120cf9` / `c8167d0` / `d2a90f8` | 差分（apiError 拡張、disabled 条件追加、usePrevious 拡張） | regression の発端コミット特定。`isConfirmDisabled` に `inputValue.trim() === ''` を入れた経緯の把握 |
| 関連 issue #159（解決済み） | `dev-journal/issues/resolved/159-...` または PR #109 本文 | 「責務再分離」前の整理 | apiError パターン採用の意図確認 |
| 共通実装ルール | `.claude/rules/implementation-workflow.md` | worktree 作業ルール / コメント言語 / FE エラーハンドリング | worktree 隔離での作業手順遵守 |

## 責務

### 含めること

- ConfirmDialog の **`isConfirmDisabled` の責務再分離**: `inputField.required && inputValue.trim() === ''` 条件を **削除** し、disabled 条件を `loading`（= `mutation.isPending`）のみに絞る
- ConfirmDialog の **`isInvalid` の責務維持**: `touched && inputValue.trim() === ''` を維持しつつ、確認ボタン押下時にも `touched=true` を立てて helperText 赤字表示を発火させる（クライアントバリデーションのトリガを onBlur + onSubmit の二点に拡張）
- ConfirmDialog の **`apiError` 責務の明確化**: prop の意味は「サーバー応答エラー」のみ。`null` のとき `FormAlert` は何も描画せず、disabled も発火させない（既存の `isConfirmDisabled` には `apiError` 条件がそもそも入っていないため変更なし。確認のみ）
- 既存ロジックで保たれている `usePrevious` ちらつき防止・useEffect による open=false 時の `inputValue`/`touched` リセット は **維持**（regression を新たに発生させない）
- ConfirmDialog テストに以下を追加:
  - 初期表示（`open=true`, `inputField.required=true`）で helperText に `errorMessage` が表示されないこと（文字数カウンタ `0 / 1000` のみ）
  - 初期表示で確認ボタンが enabled であること
  - 空のまま確認ボタンを押下したとき → helperText に `errorMessage` 赤字表示・`onConfirm` が呼ばれない
  - 入力すると helperText がエラーから文字数カウンタに戻り、ボタン押下で `onConfirm(inputValue)` が呼ばれる
  - `loading=true` のとき確認ボタンが disabled になる（既存）
  - `apiError` が文字列のとき FormAlert に表示される（既存維持）/ `null` のとき非表示（既存維持）
- 設計書 `common-components.md` §ConfirmDialog 内部仕様に「初期表示挙動」節を追加（`isConfirmDisabled` の条件、`isInvalid` の発火タイミング、`apiError` 初期値の責務分離を明示）

### 含めないこと

- **他ダイアログ（承認・支払完了・提出・削除等）の挙動変更**: これらは `inputField` を持たないか `required=false`（承認コメント任意）のため、本修正の影響を受けない設計だが、ロジック共通化の都合で disabled 条件が変わるため **既存テストでの回帰検出のみ** 行い、振る舞いは変えない
- **react-hook-form / zod の導入**: ConfirmDialog は `useState` ベースの軽量実装で十分。issue 推定原因に「react-hook-form の `mode` 設定」が挙がっているが、現実装は react-hook-form 未使用のため該当しない（推定原因の B 案は不採用）
- **`apiError` 自動クリアロジックの変更**（onChange で apiError をクリアする等）: ReportDetailPage 側の責務であり、本チケットでは触らない（既存 L284-289 で却下ボタン押下時に `setRejectDialogApiError(null)` 済み）
- **`50_detail_design/screens/report-detail.md` §11 D4 の規定変更**: 既存 D4 規定（サーバー 422 時にダイアログ内エラー表示）と本修正は整合する。文言変更は不要（必要に応じて補注のみ）
- ConfirmDialog の prop シグネチャ変更（型変更・新規 prop 追加など）: 内部実装のみで完結する
- 期間外警告 ConfirmDialog（ITM-007）・編集中変更破棄 ConfirmDialog 等の `inputField` を持たない使用箇所への波及修正

## 完了条件

issue 本文「完了条件」5 項目に以下のマッピングで対応する（DoD 7 項目に分解）。

- [ ] **DoD-1（issue 完了条件 1 前半）**: 却下確認ダイアログを開いた直後に「却下理由を入力してください」のエラー文言が表示されない（`ConfirmDialog.test.tsx` で `screen.queryByText('却下理由を入力してください')` が `null` であることを検証）
- [ ] **DoD-2（issue 完了条件 1 後半）**: 却下確認ダイアログを開いた直後の確認ボタンが enabled（`screen.getByRole('button', { name: '却下する' }).disabled` が `false`）
- [ ] **DoD-3（issue 完了条件 2）**: 空のまま「却下する」を押下した時点で初めて helperText に `errorMessage`（`却下理由を入力してください`）が赤字表示され、`onConfirm` が **呼ばれない**（`fireEvent.click` 後 `mock onConfirm` が呼ばれていないことを検証）
- [ ] **DoD-4（issue 完了条件 3）**: 理由を入力すると helperText から `errorMessage` が消え（文字数カウンタ表示）、再度ボタン押下で `onConfirm(inputValue)` が呼ばれる
- [ ] **DoD-5（issue 完了条件 4）**: SMK-096 が PASS する（`11-A-local-verification.md` の SMK-096 #2 シナリオを再検証）。SMK-011 #4 も PASS する
- [ ] **DoD-6（issue 完了条件 5）**: 既存 ConfirmDialog テスト全件 + 既存 ReportDetailPage テスト + ItemSlidePanel / AttachmentArea / ItemForm（ConfirmDialog 呼び出し側）の既存テストが全て PASS（他ダイアログへの非回帰）
- [ ] **DoD-7**: 設計書 `common-components.md` §ConfirmDialog に「初期表示挙動」節が追記され、`isConfirmDisabled` / `isInvalid` / `apiError` の責務分離が明示されている
- [ ] PR 本文に DoD 7 項目のチェックボックスを記載し、レビュー時に確認可能な状態にする

---

## 影響範囲

### コード（expense-saas）

- `frontend/src/components/ui/ConfirmDialog.tsx`
  - L164-166: `isConfirmDisabled` を `loading` のみに変更（`inputField?.required === true && inputValue.trim() === ''` 条件を削除）
  - L149-155: `handleConfirm` 内で `displayInputField?.required === true && inputValue.trim() === ''` を判定し、true なら `setTouched(true)` だけ実行して `onConfirm` を呼ばない（クライアントバリデーション発火）
  - 修正履歴コメント（L6-13）に `#162` の対応内容を追記
- `frontend/src/components/ui/__tests__/ConfirmDialog.test.tsx`
  - 既存テストの中で「初期表示で helperText にエラーが表示されるか」を前提としたケースがあれば修正
  - 新規ケース 4 件（DoD-1〜DoD-4 に対応）を追加

### 設計書（dev-journal/master 直接編集）

- `deliverables/docs/55_ui_component/common-components.md` §ConfirmDialog 内部仕様（L216-224 周辺）
  - 「必須入力バリデーション」節（L218-222）に **初期表示挙動** を明示
    > 初期表示時（`open=true` への遷移直後）は `touched=false` のため `helperText` にエラー文言は出さず、文字数カウンタのみ表示する。確認ボタンも `loading=false` のとき初期は enabled とする。`onBlur` または **確認ボタン押下時** に `touched=true` を立て、未入力なら `helperText` に赤字エラーを表示し `onConfirm` を呼ばない（クライアントバリデーション = 二段構えのうち UI 側）
  - 「API エラー表示」節（L224）に責務分離を補強（既存記述を維持しつつ）
    > `apiError` は **サーバー応答エラー専用**（422 MissingRejectionReason 等）。クライアントサイドの未入力検出は `inputField.errorMessage` + `helperText` 経路で行い、`apiError` には流し込まない。`apiError === null` のとき `FormAlert` は何も描画せず、確認ボタン disabled の発火条件にも入れない

### 設計書（参照のみ・改訂なし想定）

- `deliverables/docs/50_detail_design/screens/report-detail.md` §11 D4: サーバー側 422 時の規定。`apiError` パターン側の挙動は維持されるため改訂不要。本修正は **クライアント側** バリデーションの責務再分離

### 関連 issue / PR

- 関連 PR: #109（前回セッションでの apiError パターン導入。本 issue が指摘する regression の発端）
- 関連 issue（解決済み）: #159（却下バリデーション エラー文言が表示されない、本 issue の前修正）
- 関連 SMK: SMK-011 #4 / SMK-096 #2（ともに本修正で PASS する）

---

## 設計書改訂方針

### `55_ui_component/common-components.md` §ConfirmDialog 内部仕様

L218-222「必須入力バリデーション」節を以下に書き換える（追記箇所のみ示す）:

```diff
 - **必須入力バリデーション**（issue #159 対応）: `inputField.required === true` の場合、入力フィールドは以下の挙動を取る:
+  - **初期表示時（`open=true` 遷移直後 / 入力前）**: `touched=false` のため `helperText` にエラー文言は出さず、文字数カウンタ（`0 / maxLength`）のみ表示する。確認ボタンも `loading=false` のとき初期は enabled とし、ユーザーが空のまま押下したときに初めてバリデーションを発火させる（issue #162 対応）
   - フォーカスアウト（`onBlur`）で「touched 状態」を立てる
+  - **確認ボタン押下時** にも `touched=true` を立てる。未入力（`!value.trim()`）なら `onConfirm` を呼ばず `helperText` に `inputField.errorMessage` を赤字表示する（クライアントバリデーション二段構えの UI 側。issue #162 対応）
   - touched 状態かつ `!value.trim()` のとき、`inputField.errorMessage` が指定されていれば `helperText` に赤字エラー文言として表示する。未指定時はエラー文言を表示せず、文字数カウンタのみを表示する
   - 文字数超過（`value.length > maxLength`）時も同様に赤字エラー文言を `helperText` に表示する
-  - 確認ボタン（`onConfirm`）の `disabled` 制御も併用してよい
+  - 確認ボタン `disabled` 条件は **`loading` のみ**（多重送信防止）。「未入力時 disabled」は採用しない（issue #162 で「初期から disabled」regression を招くため）。誤送信防止は確認ボタン押下時の `touched` 発火 + `onConfirm` ガードで行う
```

L224「API エラー表示」節の末尾に以下を追記:

```diff
 - **API エラー表示**（issue #159 D4 規定対応）: ...
+  - **責務分離（issue #162）**: `apiError` は **サーバー応答エラー専用**。クライアントサイドの未入力検出は `inputField.errorMessage` + `helperText` 経路で行い、`apiError` には流し込まない。`apiError === null` のとき `FormAlert` は何も描画せず、確認ボタン disabled の発火条件にも含めない（disabled は `loading` のみ）
```

### `50_detail_design/screens/report-detail.md`

改訂不要。§11 D4 のサーバー 422 時の挙動規定は既存のままで本修正と整合する。クライアントバリデーションの整理はコンポーネント仕様（55_ui_component）側に閉じる。

---

## 実装方針

### 全体方針

issue 本文「推奨方針」（責務再分離）に従い、`ConfirmDialog` の `isConfirmDisabled` から「未入力時 disabled」条件を **削除** し、disabled 条件を `loading` のみに絞る。代わりに「確認ボタン押下時に未入力なら `touched` を立てて `onConfirm` を呼ばない」ガードを `handleConfirm` に追加する。これにより:

- 初期表示: `touched=false` → `helperText` はカウンタのみ / `loading=false` → ボタン enabled
- 空のまま押下: `handleConfirm` が `setTouched(true)` だけ実行し `onConfirm` を呼ばない → `isInvalid=true` で helperText 赤字
- 入力 + 押下: `inputValue.trim() !== ''` なので通常通り `onConfirm(inputValue)` 実行

`apiError` パターンはサーバー応答エラー専用として責務を分離し、`isConfirmDisabled` には一切関与させない（既に関与していないが、設計書で明文化する）。

### ConfirmDialog.tsx 改修パターン

**Before（L149-166）**:

```tsx
const handleConfirm = () => {
  if (displayInputField) {
    onConfirm(inputValue);
  } else {
    onConfirm();
  }
};

// ... handleClose ...

// 入力必須フィールドが空の場合は確認ボタンを無効化する。
const isConfirmDisabled =
  loading || (displayInputField?.required === true && inputValue.trim() === '');
```

**After**:

```tsx
const handleConfirm = () => {
  // 必須フィールドが未入力の場合はクライアントバリデーションを発火させ、onConfirm を呼ばない (#162)。
  if (displayInputField?.required === true && inputValue.trim() === '') {
    setTouched(true);
    return;
  }
  if (displayInputField) {
    onConfirm(inputValue);
  } else {
    onConfirm();
  }
};

// ... handleClose ...

// 多重送信防止のため loading 中のみ disabled にする (#162: 未入力時 disabled は廃止し、押下時バリデーションに切替)。
const isConfirmDisabled = loading;
```

### コメント追記（L6-13 の修正履歴）

```tsx
// 修正履歴:
//   ...
//   #162 - 初期表示時のバリデーションエラー / ボタン disabled regression 修正。
//          isConfirmDisabled から未入力条件を削除し loading のみに絞る。
//          代わりに handleConfirm 内で未入力なら setTouched(true) のみ実行して onConfirm を呼ばない
//          ガードを追加（クライアントバリデーション = onBlur + onSubmit の二点発火）。
```

---

## 作業手順

### Phase 1: 設計書改訂（dev-journal/master 直接編集）

指揮役が dev-journal の master ブランチで直接編集する（worktree 不要）。

1. `dev-journal/deliverables/docs/55_ui_component/common-components.md` §ConfirmDialog 内部仕様を改訂
   - 「必須入力バリデーション」節に初期表示挙動 + 確認ボタン押下時の touched 発火を追記
   - 「API エラー表示」節に責務分離（サーバー専用）を追記
2. dev-journal master に直接コミット
   ```
   docs(common-components): ConfirmDialog 初期表示挙動と responsibilities を明示 (#162)
   ```
3. push

### Phase 2: FE 実装（expense-saas の worktree 隔離ブランチ）

frontend-developer エージェントに以下を依頼する。

1. worktree 内に入り、ブランチをリネーム: `git branch -m step11/162-reject-dialog-initial-state-fix`
2. `ConfirmDialog.tsx` を改修（上記実装パターン）
   - L149-155: `handleConfirm` に未入力ガード追加
   - L165-166: `isConfirmDisabled` を `loading` のみに変更
   - L6-13: 修正履歴コメントに `#162` の項目追加
3. テスト追加・更新（`ConfirmDialog.test.tsx`）
   - 既存ケースの中で「初期描画時に helperText にエラー文言がある」「未入力時にボタン disabled」を期待しているものがあれば修正（あるはずだが、現実装の touched=false 初期化を踏まえると既存は「未入力時ボタン disabled」を検証している可能性が高い）
   - 新規 4 ケース追加:
     - 初期表示（required=true）で `errorMessage` が helperText に存在しない（`queryByText` で検証）
     - 初期表示でボタン enabled
     - 空のまま押下 → helperText に `errorMessage` 表示 / `onConfirm` 未呼び出し
     - 入力後 → helperText がカウンタに戻り、押下で `onConfirm(inputValue)` 呼び出し
4. ローカルで `npm run lint --workspace=frontend` と該当テストファイルのみ実行（フルスイートは CI に任せる）
   ```bash
   npm run test --workspace=frontend -- src/components/ui/__tests__/ConfirmDialog.test.tsx
   ```
5. 既存呼び出し側テストへの非回帰を念のため確認:
   ```bash
   npm run test --workspace=frontend -- src/pages/reports/__tests__/ReportDetailPage.test.tsx
   ```
   （他は CI 任せでよい）
6. コミット
   ```
   fix(frontend): 却下確認ダイアログ初期表示で error / disabled 発火する regression を修正 (#162)
   ```
7. push: `git push -u origin step11/162-reject-dialog-initial-state-fix`
8. PR 作成: `gh pr create --title "fix(frontend): #162 却下確認ダイアログ初期表示の regression 修正" --body <DoD チェックリスト + 変更内容>`
9. PR URL を指揮役に返す

### Phase 3: PR レビュー → マージ → SMK 再検証

- 指揮役が PR をセルフレビュー（既存テストの破壊的変更の妥当性 + 新規テストのカバレッジ確認）
- マージ後、issue を `dev-journal/issues/open/` から `pending-review/` に移動
- Step 11-A の SMK-011 #4 / SMK-096 #2 を再検証 → 両方 PASS で issue を `resolved/` に移動
- `11-A-local-verification.md` の発行 issue テーブルを更新（状態を resolved に）

---

## テスト方針

### 追加・更新するテスト

| ID | 対象 | 内容 | 種別 |
|----|------|------|------|
| CFD-FE-NEW-1 | `ConfirmDialog.test.tsx` | `inputField.required=true` 初期表示で `errorMessage` が helperText に表示されない（`queryByText('却下理由を入力してください')` が null）。文字数カウンタ `0 / 1000` が表示される | 新規 |
| CFD-FE-NEW-2 | `ConfirmDialog.test.tsx` | `inputField.required=true` + `loading=false` 初期表示で確認ボタンが enabled | 新規 |
| CFD-FE-NEW-3 | `ConfirmDialog.test.tsx` | 空のまま確認ボタン押下 → `errorMessage` が helperText に表示される / `onConfirm` mock が呼ばれない / ダイアログは open のまま | 新規 |
| CFD-FE-NEW-4 | `ConfirmDialog.test.tsx` | テキスト入力後 → helperText がカウンタに戻る / 確認ボタン押下で `onConfirm(inputValue)` が呼ばれる | 新規 |
| CFD-FE-既存 | `ConfirmDialog.test.tsx` | `loading=true` のとき確認ボタン disabled（既存テスト維持） | 既存維持 |
| CFD-FE-既存 | `ConfirmDialog.test.tsx` | `apiError` 文字列のとき FormAlert に表示 / `null` のとき非表示（既存テスト維持） | 既存維持 |
| CFD-FE-既存 | `ConfirmDialog.test.tsx` | 既存「未入力時 disabled」を検証するテストがある場合は **削除** または「初期表示で enabled」へ書き換え | 既存改修 |
| CFD-FE-既存 | `ConfirmDialog.test.tsx` | onBlur で touched が立ち、helperText にエラー表示される（既存挙動維持） | 既存維持 |

### 確認コマンド

```bash
# 該当テストファイルのみ実行（worktree 内）
npm run test --workspace=frontend -- src/components/ui/__tests__/ConfirmDialog.test.tsx

# 呼び出し側非回帰確認
npm run test --workspace=frontend -- src/pages/reports/__tests__/ReportDetailPage.test.tsx

# Lint（worktree 内）
npm run lint --workspace=frontend
```

> フルテストスイートは CI（GitHub Actions）で実行する。サブエージェントにローカル全件実行はさせない（feedback_no_local_test_run）。

### 手動再検証（マージ後 / SMK 再実行）

1. Approver でログイン → 提出済みレポート詳細 → 「却下する」押下
2. 却下確認ダイアログが開いた直後に:
   - エラー文言「却下理由を入力してください」が **表示されない**（カウンタ `0 / 1000` のみ）
   - 「却下する」ボタンが **enabled**
3. 空のまま「却下する」押下:
   - エラー文言「却下理由を入力してください」が helperText に赤字表示
   - ダイアログは閉じない / API 呼び出しなし
4. 理由を入力（例: 「金額誤り」）:
   - エラー文言が消えカウンタ `4 / 1000` 表示
5. 「却下する」押下 → 成功トースト + ダイアログ閉じる
6. SMK-011 #4 / SMK-096 #2 再実行 → PASS 記録

---

## 依存・並列性

- **依存なし**: ConfirmDialog の内部修正のみ。既存 prop シグネチャを変更しないため呼び出し側修正不要
- **第3バッチ並列**: G1（#154+#160 AppDataGrid）/ G3（#164 DashboardPage）と並列実行可能（修正対象ファイル重複なし）
  - #154+#160: `frontend/src/components/ui/AppDataGrid.tsx` 系
  - #164: `frontend/src/pages/dashboard/DashboardPage.tsx`
  - #162: `frontend/src/components/ui/ConfirmDialog.tsx`
- **後続作業**: マージ後、`11-A-local-verification.md` の発行 issue テーブルを更新し、SMK-011 / SMK-096 の備考を「副次 issue #162 → resolved」に更新する

---

## リスク・注意点

1. **共通 ConfirmDialog の修正が他ダイアログ（承認・支払・提出・削除等）に波及する可能性**
   - 他ダイアログは `inputField` を持たないか、または `required=false`（承認コメント任意）のため、`isConfirmDisabled` 条件変更（`loading` のみ）の影響を受けない設計
   - ただし、既存テストで「未入力時 disabled」を期待しているケースがあれば必ず PR 差分で確認する
   - 既存呼び出し側（`ReportDetailPage`, `ItemSlidePanel`, `AttachmentArea`, `ItemForm`）のテストを念のため実行する
2. **既存テスト「未入力時 disabled」の処理**
   - 仕様変更のため、対応する既存テストは **削除** または「初期表示で enabled」に書き換える必要がある
   - PR 説明にこの仕様変更の理由（issue #162 で発覚した regression の根本対応）を明記する
3. **`usePrevious` による prevInputField の混入**
   - `displayInputField = open ? inputField : (prevInputField ?? inputField)` の `usePrevious` 経路で、open=true 直後の初回 render に旧 inputField 状態が混入する可能性がある
   - `useEffect(() => { if (!open) { setInputValue(''); setTouched(false); } }, [open])` で close 時にリセットされるため理論上問題ないが、テストで「閉じて再度開く」シナリオも検証する
4. **MUI の autoFocus による blur 連鎖の可能性**
   - `<TextField autoFocus />` でフォーカスが入った直後に何らかの blur が発生して `setTouched(true)` が走る可能性は低いが、初期表示テストで `screen.queryByText` が null であることを確認することで担保する
5. **react-hook-form / zod は使われていない**
   - issue 本文の推定原因 B 案（`mode: 'onChange'` / `mode: 'onMount'`）は現実装に該当しない（`useState` ベースのため）。修正方針には含めない
6. **設計書 D4 規定との整合**
   - サーバー応答 422 時の `apiError` パターンは維持される。本修正はクライアントバリデーションの責務分離のみで、D4 規定文言の変更は不要
7. **dev-journal master 直接コミット**
   - 設計書改訂は dev-journal の master 直接コミット（PR 経由ではない）。指揮役が責任を持つ
8. **MVP スコープ厳守**
   - ConfirmDialog 内部の `handleConfirm` / `isConfirmDisabled` 修正 + テスト + 設計書注記のみ。`apiError` 自動クリア・onChange 連動・prop シグネチャ変更等は含めない

---

## 後続への引き継ぎ

- マージ後、本 issue を `dev-journal/issues/open/162-...md` → `pending-review/` に移動し、SMK-011 #4 / SMK-096 #2 副次として再検証
- 再検証 PASS 後、`resolved/` に移動
- `11-A-local-verification.md` の発行 issue テーブルを更新（状態を resolved に）
- 本チケット（11-A-issue-162.md）を完了マーク → progress.md の課題セクションから #162 を「対応完了」に更新

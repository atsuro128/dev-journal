# 却下確認ダイアログを開いた瞬間にバリデーションエラーが表示され、「却下する」ボタンが disabled 状態になる（#159 修正の regression）

## 発見日
2026-04-30

## カテゴリ
ui-design / frontend / regression

## 影響度
高（業務影響: Approver が却下操作を実行できない可能性。空のままボタン押下時のエラー表示も期待通り動作しない）

## 発見経緯
user-report / Step 11-A SMK-011 #4 / SMK-096 #2 再検証時に、却下確認ダイアログを開いた瞬間（理由未入力の初期状態）から「却下理由を入力してください」のバリデーションエラー文言が赤字で表示されている、かつ「却下する」ボタンが disabled になっていることをユーザーが発見

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（詳細 URL 直打ちで承認/却下フロー自体は別経路で迂回不可だが、UI 上の表示は機能している）

## 問題

### 現象

却下確認ダイアログ（ConfirmDialog with apiError パターン、SCR-RPT-004 → 「却下する」押下時に開く）を開くと:

1. **ダイアログ初期表示時点で**「却下理由を入力してください」のエラー文言がフィールド下に赤字で表示される（理由は未入力 = 初期状態にもかかわらず）
2. 「却下する」ボタンが **最初から disabled** になっており、空のまま押下することができない

### 期待動作（SMK-096 仕様 + 設計書 D4 規定）

- ダイアログを開いた直後は **エラー文言なし**、「却下する」ボタンは押下可能
- 空のまま「却下する」を押下した時に初めて「却下理由を入力してください」がフィールド下に赤字で表示され、送信されない
- 理由を入力すると **エラーが消え**、ボタン押下で確認・送信される

### 推定原因

#159 修正（apiError パターン採用、commit `5120cf9` / `c8167d0` / `d2a90f8` 周辺）で導入された RejectConfirmDialog（または ConfirmDialog with required input）の初期化ロジックに不備がある。具体的には:

- `apiError` プロパティの初期値が `""`（空文字）または何らかの truthy 値になっていて、初期表示時点で「エラーあり」状態として描画されている可能性
- または、フォームバリデーション（zod / react-hook-form）の `mode: 'onChange'` / `mode: 'onMount'` 等の設定で初期値（空文字）に対してすぐにバリデーションを走らせている可能性
- ボタン disabled の制御が「`apiError` が truthy」または「formState.isValid === false」を初期から見ており、ユーザー入力前に disabled になっている

### 影響範囲

- **発生**: Approver の却下操作（apiError パターン採用 = 却下ダイアログのみ）
- **発生しない**: 承認 / 支払完了 / 提出 / レポート削除 / 明細削除 / 添付削除（旧トーストパターン維持）

`feedback_critical_review_of_codex` で前回セッションに「設計書 D4 規定の本来のスコープ（required input validation 422 = MissingRejectionReason）に整合」として案 C（apiError パターンを却下ダイアログのみに限定）を採用したが、実装側で初期状態の挙動がずれている。

## 影響

- Approver が却下操作を空のまま押下しても期待のバリデーションエラー表示が見えない（ボタン disabled で押下できないため）
- 「却下理由を入力してください」が常に表示されるため、入力欄を見る前に「自分が何か誤っているのでは」と感じる UX の悪化
- SMK-096 仕様（フィールド下に赤字でエラー表示）が満たせていないため Step 11-A の SMK が PASS しない

## 提案

### Step 1: 実装の現状把握

- `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx` の `apiError` prop の初期値・受け渡し
- `expense-saas/frontend/src/components/workflow/RejectConfirmDialog.tsx`（または同等）の初期化ロジック
- フォームバリデーション設定（zod schema / react-hook-form の `mode` 等）
- ボタン disabled 条件

### Step 2: 想定修正

- `apiError` prop の初期値を **`null`** に明確化し、`null` のときは描画も disabled も発火させない（truthy / null 判定の徹底）
- フォームバリデーションを `mode: 'onSubmit'` または `mode: 'onTouched'` にして、ユーザー操作前にエラーを発火させない
- ボタン disabled は `mutation.isPending` のみ（または送信中のみ）に絞り、初期状態では押下可能にする

### 推奨方針

apiError パターンの責務は「サーバー応答エラー（422 MissingRejectionReason 等）」をフィールド下に描画することであり、**クライアントサイドバリデーションは別レイヤー**で扱うべき。前回セッションの apiError 拡張時にクライアントバリデーションと混在した可能性があるため、責務を再分離する:

- 初期表示: エラーなし、ボタン押下可能
- 空のまま送信ボタン押下 → サーバー応答 422 → apiError prop に「却下理由を入力してください」が入る → フィールド下に赤字表示
- ユーザーが入力 → onChange で apiError をクリア（または変更検知でクリア）

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx` | apiError prop の初期値・描画条件・ボタン disabled 条件の見直し |
| `expense-saas/frontend/src/components/workflow/RejectConfirmDialog.tsx`（または同等） | apiError 受け渡し・onChange 時のクリア処理追加 |
| `expense-saas/frontend/src/components/ui/__tests__/ConfirmDialog.test.tsx` | 初期表示時に apiError 文言が描画されないテスト追加、ボタン押下可能状態を検証 |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` §ConfirmDialog | apiError の初期状態仕様を明示（初期値 null / undefined → 描画なし） |
| `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` D4 規定周辺 | 必要に応じて初期表示挙動を補足 |

## 完了条件

- 却下確認ダイアログを開いた直後はエラー文言なし、「却下する」ボタンが押下可能
- 空のまま「却下する」を押下したときに初めて「却下理由を入力してください」がフィールド下に赤字表示
- 理由を入力するとエラー文言が消え、ボタン押下で確認・送信される
- SMK-096 が PASS する
- 既存テスト（承認/支払完了/削除等の他ダイアログ）に影響がないこと

## ラベル

- type: bug / regression / ui
- area: frontend

## 関連

- 関連 issue: #159（却下バリデーション エラー文言が表示されない、本 issue の前修正）
- 関連 PR: #109（前回セッション、commit `683c076` / `adc5017` / `0f23fc0` / `5120cf9` / `c8167d0` / `d2a90f8`）
- 関連 SMK: SMK-011 #4 / SMK-096 #2

## MVP 区分

MVP（業務上必要な却下フローの UX 不具合、Step 11-A SMK FAIL のため）

---

## 解決内容

**採用方針**: ConfirmDialog 共通基盤の disabled 条件を `loading` のみに絞る + 未入力ガード追加

**真因確定**:
- `ConfirmDialog.tsx` L165-166 の `isConfirmDisabled = loading || (displayInputField?.required === true && inputValue.trim() === '')` が初期 inputValue=`""` でボタン disabled にしていた
- issue 推定 B（react-hook-form mode）は不該当（useState ベース実装のため）
- 「初期表示で赤字エラー」は理論上 `touched=false` の初期状態では出ない（自動テスト + 手動再検証で確認）。現象は「ボタン disabled」が実体

**実装** (PR #113, commit `dceec05`):
- `ConfirmDialog.tsx` L165-166 の disabled 条件を **`isConfirmDisabled = loading` のみに絞る**
- `handleConfirm` 内で「未入力なら `setTouched(true)` を発火し `onConfirm` を呼ばない」未入力ガード追加
- `apiError` の責務を「サーバー応答エラー専用」として明確化

**テスト改修**:
- 既存「未入力時 disabled」を「初期表示で enabled」に書き換え
- NEW-1〜4 追加（初期表示エラー非表示・ボタン enabled、空送信エラー、入力後クリア、`setTouched` 発火）
- ConfirmDialog.test.tsx 29 件 + ReportDetailPage.test.tsx 40 件 PASS

**設計書改訂** (commit `fa21368`):
- `55_ui_component/common-components.md` §ConfirmDialog に責務分離（apiError = サーバー専用）と disabled 条件 = `loading` のみを明記

**SMK 再検証要項目**: SMK-011 #4 / SMK-096 #2

## 解決日

2026-04-30

---

## 再対応（2026-05-02、Step 11-A SMK-011 #4 再検証で再発確認）

### 状態

**FAIL**（PR #113 の修正は 2/3 で残り 1 経路を取り逃したことを確認、再 open）

### 再現症状

却下確認ダイアログを開いた瞬間（理由未入力の初期状態）から:

- 「却下理由を入力してください」のエラー文言がフィールド下に赤字で表示される
- 「却下する」ボタンは押下可能（PR #113 の disabled 修正は効いている）
- ただし押下しても何も起きない（`handleConfirm` 内の未入力ガードで return される、これも PR #113 の正常動作）

リロード後・初回オープン時から発生（state 引き継ぎではない）。

### 真因（再分析）

PR #113 で取りこぼした経路:

1. `TextField` の `autoFocus` (ConfirmDialog.tsx L194) でフォーカスが当たる
2. MUI Dialog の FocusTrap 機構で Dialog 内の他要素にフォーカスが移動
3. TextField から **blur イベント発火**
4. `onBlur={() => setTouched(true)}` (L212) で `touched=true`
5. 初回 paint 時点で既に `isInvalid=true` → 赤字エラー表示

ユーザー観察「ダイアログを開いてもフォーカスされていない」= autoFocus が一瞬当たった直後に FocusTrap でフォーカスが他に移っている状態を裏付け。

### 前回検証の不備

- jsdom 環境では autoFocus → FocusTrap の blur 連鎖が発火しないため、自動テスト（ConfirmDialog.test.tsx 29 件）では検出不可
- 前回の手動再検証も「現象は disabled 化のみ」と誤認、エラー文言の経路を確認していなかった

### 採用方針

**案 A: TextField の `autoFocus` 削除**（最小変更で根本解決）

- 検討した代替案 D（`TransitionProps.onEntered` + `useRef` でプログラム的にフォーカス、UX 維持）はコード変更が増える + transition タイミングの v8 エッジケースリスクがあるため不採用
- UX デメリット: ダイアログを開いた後 TextField を 1 クリックする手間が増える（却下は Approver の意識的操作なので影響軽微）
- `onBlur` は維持: ユーザーが手動でフィールドに触ってから他に移動した場合の touched 発火は UX 的に有用（押下前に未入力エラーを誘導）

### 修正対象

| ファイル | 変更内容 |
|---|---|
| `expense-saas/frontend/src/components/ui/ConfirmDialog.tsx` | TextField の `autoFocus` 削除、修正履歴コメントに #162 再対応を追記 |
| `expense-saas/frontend/src/components/ui/__tests__/ConfirmDialog.test.tsx` | autoFocus 削除を確認するテスト追加（initial render で touched=false / error 非表示を実機相当で確認） |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` §ConfirmDialog | autoFocus 規定があれば削除、autoFocus 不採用の理由を記載 |

### SMK 再検証要項目

SMK-011 #4 / SMK-096 #2

### 起票日（再 open）

2026-05-02

---

## 解決確認（2026-05-04）

### 状態
**解決（resolved 移動）**

### 解決根拠
PR #121（ConfirmDialog.tsx の TextField `autoFocus` 削除）マージ後、2026-05-02 セッションで SMK-011 #4 / SMK-096 #2 の両項目が PASS。autoFocus → FocusTrap blur 連鎖によるエラー初期表示が根本解消された。

### 関連 PR / コミット
- PR #113（commit `dceec05`）: disabled 条件を `loading` のみに絞る（初回対応）
- PR #121: TextField `autoFocus` 削除（再対応・根本解決）

### 備考
jsdom 環境では autoFocus → FocusTrap の blur 連鎖が再現しないため、自動テストで初回対応時に検出漏れが発生した。

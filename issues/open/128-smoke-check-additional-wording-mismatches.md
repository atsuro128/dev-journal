# smoke_check.md 文言整合監査 残件（6 件集約）

## 発見日
2026-04-20

## カテゴリ
docs / consistency / test-specification

## 影響度
中〜低（docs 整合性の問題。実装の正本は state-management.md / 画面設計書）

## 発見経緯

issue 123（SMK-026）/ 125（SMK-028）対応時の smoke_check.md 全体監査（§4.3 エラー表示・§4.6 日本語 UI の全文突合）で、正本設計書（state-management.md §6.5 SERVER_ERROR_MESSAGES / 各 screens/*.md）と期待文言が乖離している項目を追加で 6 件発見。運用ルール（11-A: 発見分は都度単独 issue 起票）に従い本 issue に集約起票する。

## 関連ステップ
Step 6-D（テスト設計: smoke_check.md 作成）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題（不整合一覧）

### 乖離度「高」（文言が完全に別物、実装との齟齬リスク大）

#### SMK-027: CONFLICT 文言乖離

- **smoke_check.md 期待**: 「他のユーザーが既に処理しています。一覧を再読込してください」
- **正本（state-management.md L468 `CONFLICT`）**: 「他のユーザーがこのレポートを更新しました。ページを再読み込みしてください。」
- **乖離点**: 主語（「既に処理」vs「このレポートを更新」）、語尾（「一覧を再読込」vs「ページを再読み込み」）の双方

#### SMK-036: 添付 422 エラー文言

- **smoke_check.md 期待**: 「ファイル形式が不正です」
- **正本（state-management.md L490 `INVALID_FILE_TYPE`）**: 「JPEG, PNG, PDF のみアップロード可能です」
- **乖離点**: 文言ごと別物（SMK-033 では同じ `INVALID_FILE_TYPE` が正本通り記載されており、SMK-036 のみ乖離）

### 乖離度「中」

#### SMK-021: INVALID_AMOUNT（負数入力）

- **smoke_check.md 期待**: 「正の金額を入力してください」
- **正本（state-management.md L488 `INVALID_AMOUNT`）**: 「金額は正の整数で入力してください」
- **乖離点**: 整数制約が抜けている

#### SMK-023: INVALID_AMOUNT（小数入力）

- **smoke_check.md 期待**: 「円単位の整数で入力してください」
- **正本（state-management.md L488 `INVALID_AMOUNT`）**: 「金額は正の整数で入力してください」
- **乖離点**: 文言が正本と別物

#### SMK-084: 空状態メッセージ

- **smoke_check.md 期待**: 「レポートがありません」のメッセージと「新規作成」への導線
- **正本（report-list.md L217 / dashboard.md L337）**: 「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」 + 「レポート作成」ボタン
- **乖離点**: 文言が設計書と不一致、action ラベルも「新規作成」vs「レポート作成」で異なる

### 乖離度「低」

#### SMK-025: FORBIDDEN 句点欠落

- **smoke_check.md 期待**: 「この画面にアクセスする権限がありません」（句点なし）
- **正本対応**: 画面遷移 403 のトースト文言として「この画面にアクセスする権限がありません。」（WFL-FE / TNT-FE テストケース、issue-088 対応）
- **乖離点**: 句点「。」の欠落のみ

## 修正方針

### 1. 各項目の期待文言を正本に揃える

smoke_check.md の SMK-021 / SMK-023 / SMK-025 / SMK-027 / SMK-036 / SMK-084 の期待結果を、上記「正本」欄の文言に書き換える。

### 2. SMK-036 の手順見直し

SMK-036 の手順が「ファイル形式が不正」を期待しているが、実装上は「JPEG/PNG/PDF 以外のファイル（例: .txt, .exe）をアップロード」することで `INVALID_FILE_TYPE` が発火するため、操作手順にファイル形式の具体例を追記する。

### 3. SMK-084 の action ラベル確認

空状態のボタンラベル「レポート作成」と smoke_check.md の「新規作成」が別物のため、実装（ReportListPage / DashboardPage）のボタンラベルを確認し、正本と実装が一致していることを確認したうえで smoke_check.md を正本に揃える。実装側で乖離していた場合は別途実装 issue を起票する。

## 修正対象ファイル

- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`
  - SMK-021 / SMK-023 / SMK-025 / SMK-027 / SMK-036 / SMK-084 の期待結果・一部手順の修正

## 対応条件

- SMK-021 / SMK-023 / SMK-025 / SMK-027 / SMK-036 / SMK-084 の期待文言が state-management.md / 各 screens/*.md と一致
- SMK-036 の操作手順に非対応ファイル形式の具体例が明記されている
- SMK-084 の action ラベルが実装（ReportListPage / DashboardPage）と正本設計書の双方と整合（乖離があれば実装 issue を別途起票）

## 関連

- issue 123: SMK-026 修正（実施済、本 issue と同じ監査プロセスで発見）
- issue 125: SMK-028 修正（実施済、本 issue と同じ監査プロセスで発見）
- issue 122: SMK-024 対応（post-MVP UX 改善と一緒に管理）
- 将来同種の不整合を追加発見した場合は都度単独 issue 起票（11-A 運用ルール継続）

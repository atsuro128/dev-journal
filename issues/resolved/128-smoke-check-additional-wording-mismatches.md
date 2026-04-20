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

---

## レビューコメント（2026-04-20）

判定: 差し戻し

### 未解消の問題

- `smoke_check.md` の SMK-021 / SMK-023 / SMK-025 / SMK-027 / SMK-036 の期待文言、および SMK-036 の操作手順は正本に揃っている。
- ただし SMK-084 の action ラベル確認条件が未完了。`dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` は「+ レポート作成」ボタンを期待している一方、実装の `expense-saas/frontend/src/pages/reports/ReportListTable.tsx` は空状態 action を `レポートを作成` としており、設計書・スモークチェックとの表記ゆれが残っている。
- issue ファイル自体にも `解決内容` / `解決日` が未記載で、pending-review へ移す前提を満たしていない。

### 追加で修正すべき成果物

- `expense-saas/frontend/src/pages/reports/ReportListTable.tsx`
  - 空状態 action ラベルを設計・smoke_check と一致させる、または実装方針として別ラベルにするなら設計書と smoke_check を同じ判断で更新する。
- 必要に応じて `expense-saas/frontend/src/pages/reports/__tests__/ReportListTable.test.tsx`
  - 空状態 action ラベルの期待を追加または更新する。
- 本 issue ファイル
  - 対応後に `解決内容` / `解決日` を追記する。

### 上流修正の要否

現時点では上流判断の修正は不要。`50_detail_design/screens/report-list.md` と `55_ui_component/screens/report-list.md` は「レポート作成」系のラベルで揃っているため、実装修正または明示的な設計変更で閉じられる。

### そのまま進めた場合の影響

Step 11-A の確認者が smoke_check の期待結果と実画面のボタンラベル差分を不具合として扱うか判断に迷う。スモーク確認の合否基準が固定されないため、Issue 128 の対応条件を満たしたとは判定できない。

---

## 解決内容（2026-04-20）

本 issue の 6 件を smoke_check.md で修正完了：

- **SMK-021**: 「金額は正の整数で入力してください」に整合（state-management.md L488 `INVALID_AMOUNT`）
- **SMK-023**: SMK-021 と同文言に統一
- **SMK-025**: 句点追加「この画面にアクセスする権限がありません。」
- **SMK-027**: 「他のユーザーがこのレポートを更新しました。ページを再読み込みしてください。」に整合（state-management.md L468 `CONFLICT`）
- **SMK-036**: 「JPEG, PNG, PDF のみアップロード可能です」に整合、手順に具体例（`.txt` や `.exe` を `.pdf` にリネーム）追記
- **SMK-084**: 空状態文言「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」+ EmptyState 内の「レポートを作成」アクション導線に整合

コミット: ef46060（smoke_check.md）、fe43934 後の branch

### SMK-084 action ラベルの判断

初回修正で「+ レポート作成」ボタン（ReportListHeader.tsx L20）を参照したが、codex 指摘で SMK-084 の観点「空状態表示」が参照すべきは EmptyState の action（ReportListTable.tsx L83 「レポートを作成」）であることを確認。設計書（report-list.md L217 / dashboard.md L337）も「レポート作成ボタン」系の記述のみで「+」記号は仕様外。EmptyState 内のアクション「レポートを作成」に修正した。

ヘッダーの「+ レポート作成」ボタンのラベルは別観点（常時表示ボタン）として実装にあるが、SMK-084 のスコープ外。

## 解決日

2026-04-20

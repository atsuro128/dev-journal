# レポート一覧の空状態メッセージが DataGrid の高さ不足で表示されない

## 発見日
2026-04-28

## カテゴリ
implementation / frontend / ui

## 影響度
中（機能影響なし。空状態でユーザーへの誘導文言が見えず、新規ユーザー / 該当データがないロールでの操作導線が分かりづらい）

## 発見経緯
user-report / Step 11-A SMK-006 後の Approver でレポート一覧（`/reports`）を開いた際、件数 0 だが空状態メッセージ「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」が画面上に見えないことをユーザーが報告

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

レポート一覧画面（`/reports`、SCR-RPT-001）で件数が 0 のとき、AppDataGrid 内に表示されるはずの空状態メッセージ「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」が画面に見えない。テーブルヘッダーは表示されるが、その下のコンテンツ領域の高さが不足しており、`NoRowsOverlay`（または同等の空状態描画）が隠れていると推定される。

### 推定原因（要 DOM 検証で確定）

- AppDataGrid（`expense-saas/frontend/src/components/ui/AppDataGrid.tsx` 周辺）が、件数 0 時に最小高さを確保していない
- MUI X DataGrid のデフォルト動作では、`autoHeight` 未指定時に固定高さ（または高さ 0）になり、`NoRowsOverlay` が描画されてもコンテナサイズで隠れる

### 影響範囲

ロール非依存と推定される（DataGrid の汎用挙動）。同じ AppDataGrid 基盤を使う以下の画面でも同事象の可能性:

- レポート一覧（`/reports`、Member / Approver / Accounting / Admin）
- 全レポート（`/admin/reports`、Admin）
- 承認待ち一覧（`/approvals`、Approver）
- 支払待ち一覧（`/payments`、Accounting）

各画面の空状態を Phase 検証で確認すべき。

## 影響

- 件数 0 のロール / 新規ユーザーで「データがない」のか「読み込み中」のか「エラー」なのかが視覚的に判別できない
- 「レポートを作成して経費精算を始めましょう」という新規ユーザー向け導線文言が機能しない
- 設計書で空状態文言が規定されているのに UI 上で消費されていない（実装と表示の乖離）

## 提案

### Step 1: DOM 検証で原因確定

ブラウザ DevTools でレポート一覧（件数 0 状態）の DOM を確認:

- AppDataGrid のルート要素 / コンテナ要素の computed style `height`, `min-height`
- `NoRowsOverlay`（または `<div class="MuiDataGrid-overlay">` 等）の存在 / 表示状態 / overflow 状態
- 親要素（PageContainer / Layout）の高さ制約

### Step 2: 原因に応じた修正

**ケースA: AppDataGrid 自体に最小高さがない場合**

AppDataGrid の sx に `minHeight: 200px`（または適切な値）を設定。または `autoHeight` を有効化してコンテンツ高さに自動追従させる。

**ケースB: DataGrid の overlay が描画されているが parent overflow で隠れている場合**

親コンテナ側で overflow / height 制約を見直す。

**ケースC: 空状態の文言コンポーネント自体が条件分岐で出ていない場合**

`useReports` の戻り値判定（loading 終了 + 件数 0）で空状態専用の Box / Card を AppDataGrid と独立して描画する。

### 推奨方針

AppDataGrid を共通コンポーネントとして使っている画面が複数あるため、**AppDataGrid 側で空状態時の最小高さ確保**（ケースA）が波及効果が大きく望ましい。各画面個別で対応すると統一性が崩れる。

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/ui/AppDataGrid.tsx` | 件数 0 時の最小高さ確保（minHeight or autoHeight） |
| `expense-saas/frontend/src/components/ui/__tests__/AppDataGrid.test.tsx` | 空状態時の高さ検証テスト追加 |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` | AppDataGrid の空状態仕様記述追加（必要に応じて） |

## 完了条件

- レポート一覧（件数 0）で空状態メッセージが画面上に表示される
- 影響範囲の他画面（全レポート / 承認待ち / 支払待ち）でも空状態が正しく表示される
- AppDataGrid 共通テストで空状態の高さ・描画が検証されている

## ラベル

- type: bug / ui
- area: frontend

## 関連

- 関連 SMK: SMK-006（Approver グローバルナビ確認）の副次発見

## MVP 区分

MVP（新規ユーザー導線が機能しない、UX 影響大）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

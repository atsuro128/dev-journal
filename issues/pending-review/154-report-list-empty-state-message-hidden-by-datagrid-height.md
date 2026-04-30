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

**採用方針**: 推奨ケース A（AppDataGrid 共通基盤側で最小高さ確保）

**実装** (PR #107, commit be95274):
- `AppDataGrid.tsx` の DataGrid sx に `minHeight: 200` を追加（共通基盤側で件数 0 時の最小高さを確保）
- `autoHeight` は採用しない（issue #147 のフッター常時表示と干渉するため固定 minHeight 方式）
- 各画面側で個別に minHeight 設定する必要なし（4 一覧画面で統一挙動）

**テスト追加**:
- ADG-005: rows=[] 時に DataGrid の sx.minHeight が 200 で適用されることを検証

**設計書改訂** (commit 255d801):
- `55_ui_component/common-components.md` §AppDataGrid に「件数 0 時の最小高さ（minHeight: 200）を共通基盤側で確保」を追記

**残課題** (本 issue スコープ外):
- minHeight: 200 は本対応の選択値。EmptyState のアクションボタン付き（ReportListTable）で実機確認後に値の妥当性を再評価する余地あり

## 解決日

2026-04-30

---

## 再検証結果（2026-04-30 SMK-006 再実施）

### 状態

**FAIL**（再対応が必要、`pending-review/` から `open/` に戻して継続対応）

### 検証で判明した不備

minHeight: 200（一律適用）が **2 つの問題を同時に発生** させている:

1. **0 件時**: minHeight 200px では `EmptyState` の「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」（アクションボタン込み）の高さに不足し、空状態文言が **依然として画面上に見えない**
2. **1 件時**: 1 行コンテンツに対して minHeight 200px は過剰で、テーブル末尾に **不自然な余白**が生まれる

### 再対応方針案

| 案 | 内容 | 評価 |
|----|------|-----|
| A（推奨） | minHeight を **0 件時のみ条件付き適用**（300px 等の十分な値）。`rows.length === 0 ? 300 : undefined` のような条件分岐 | 根本対応 |
| B | minHeight を一律で 300px 等に上げる | × 1 件時の余白が更に悪化 |
| C | NoRowsOverlay を AppDataGrid 外で条件分岐レンダー（DataGrid とは独立した EmptyState 描画） | 設計大幅変更だが本来の責務分離としては最も綺麗 |

→ 推奨: **案 A**（共通基盤側で件数 0 を判定して minHeight 切り替え）。

### 想定変更

- `expense-saas/frontend/src/components/ui/AppDataGrid.tsx` で `rows.length === 0` 時のみ `sx.minHeight` を適用（値は `EmptyState` 全体が見える 300px 程度）
- `expense-saas/frontend/src/components/ui/__tests__/AppDataGrid.test.tsx` ADG-005 を改修: 0 件時は minHeight 適用、1 件以上は minHeight が undefined を確認
- `dev-journal/deliverables/docs/55_ui_component/common-components.md` §AppDataGrid の minHeight 規定を「件数 0 時のみ条件付き適用」に更新

### 検証結果（再 PASS 条件）

- 0 件時に EmptyState 全文（アクションボタン含む）が画面上に視認できる
- 1 件時にテーブル末尾に不要な余白がない
- 2 件以上のロード済み一覧でも違和感なし

### 起票日（再 open）

2026-04-30

---

## 再対応の解決内容（2026-04-30 第3バッチ）

**採用方針**: 案A（minHeight 条件付き適用）

**実装** (PR #114, commit `664efbc`):
- `AppDataGrid.tsx` sx の `minHeight: 200` を `minHeight: rows.length === 0 ? 300 : undefined` に変更
- 0 件時: 300px（EmptyState のアクションボタン込みで見切れない値）
- 1 件以上: undefined（DataGrid 自然高さに任せて余白を防ぐ）

**テスト改修**:
- ADG-005 を ADG-005a（0 件時 minHeight 300）/ ADG-005b（1 件以上 minHeight undefined）の 2 ケースに分解
- AppDataGrid.test.tsx 7 件 PASS

**設計書改訂** (commit `40ccb0f`):
- `55_ui_component/common-components.md` §AppDataGrid の minHeight 規定を「件数 0 時のみ条件付き適用（300px）、1 件以上は適用しない」に更新

**SMK 再検証要項目**: SMK-006 #4

## 再解決日

2026-04-30

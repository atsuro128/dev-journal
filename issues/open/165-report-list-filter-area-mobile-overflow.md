# マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える

## 発見日
2026-05-02

## カテゴリ
implementation / frontend / ui / responsive

## 影響度
中（機能影響なし。スマホ幅でのフィルタ操作時に画面幅を超え、UI が破綻して操作不能になる可能性）

## 発見経緯
user-report / Step 11-A SMK 再検証中に issue #160 の真因調査の過程で発見。DevTools iPhone SE 375px でマイレポート画面 (ReportListTable) を確認したところ、上部のフィルタエリア（ステータス / 開始日 / 終了日）が **横一列のまま画面幅 375px に収まらず、画面外に溢れている**ことを観察。

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

`ReportListTable.tsx`（または上位ページ）の上部フィルタエリア:

- ステータス絞り込み select
- 開始日 date picker
- 終了日 date picker

がスマホ幅（375px）で **横一列の flex layout のまま画面幅を超えて溢れている**。

### 期待動作

スマホ幅では:

- フィルタが **縦積み**（または折りたたみ式）になり画面幅に収まる
- 各 input 要素が tap しやすい幅で表示される
- ハンバーガーメニュー / グローバルナビゲーションと干渉しない

### 該当箇所（推定）

- `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`（または `ReportListTable.tsx` の親コンポーネント）の上部フィルタエリア

レイアウトは恐らく `<Stack direction="row">` または `<Box sx={{ display: 'flex' }}>` で固定された row layout になっている。

### 影響範囲

マイレポート画面のみ確認済み。他の一覧画面（テナント全レポート / 承認待ち / 支払待ち / 処理済み）にも同種のフィルタエリアがあれば同症状の可能性があるため、修正時に横展開要否を確認すること。

## 影響

- スマホでマイレポート画面を開いた Member ロールが、フィルタを正常に操作できない
- 画面幅を超えた要素が右側に隠れ、ユーザーは存在に気付けない可能性
- SMK-101 の延長問題（テーブル幅問題と同様、mobile レスポンシブ対応の漏れ）

## 提案

### 修正案 A: Stack の direction をレスポンシブに切替（推奨）

```tsx
<Stack direction={{ xs: 'column', sm: 'row' }} spacing={2}>
  <StatusSelect ... />
  <DatePicker label="開始日" ... />
  <DatePicker label="終了日" ... />
</Stack>
```

メリット: 最小変更。MUI 公式 Responsive パターン
デメリット: mobile で縦積み 3 要素になり画面が縦に伸びる

### 修正案 B: アコーディオンで折りたたみ

mobile 幅では `<Accordion>` で折りたたんで「フィルタ」見出しタップで展開する形。

メリット: 画面領域を圧迫しない
デメリット: 1 アクション増える、実装やや複雑

### 推奨

**案 A**（Stack direction レスポンシブ）。最小変更で SMK-101 期待結果に沿う。Member 画面は頻繁にフィルタを使う想定のため、初期表示で見える状態（縦積み）の方が UX 良い。

## 修正対象ファイル（推定、案 A）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`（または該当コンポーネント） | フィルタエリアの Stack を `direction={{ xs: 'column', sm: 'row' }}` に変更 |
| 対応するテストファイル | mobile 縦積みを検証するテスト追加 |
| `dev-journal/deliverables/docs/50_detail_design/screens/report-list.md`（該当画面の設計書） | mobile レスポンシブ規定を追記 |

修正前に該当ファイルを特定すること（`grep` で「ステータス絞り込み」「開始日」「終了日」のラベル文言を検索）。

## 完了条件

- スマホ幅（375px）でマイレポート画面のフィルタエリアが画面幅に収まる（案 A なら縦積み）
- 各フィルタ要素が tap しやすい幅で表示される
- 既存テスト通過 + 新規テスト追加
- 横展開要否確認（テナント全レポート / 承認待ち / 支払待ち / 処理済みでも同種フィルタエリアがあれば同時対応）

## ラベル

- type: bug / ui / responsive
- area: frontend

## 関連

- 関連 issue: #160（同じ Step 11-A 検証中に発見、テーブル mobile 対応の延長問題）
- 関連 SMK: 既存 SMK では未カバー（Step 11-A で目視発見）

## MVP 区分

MVP（業務上 mobile 操作の前提が崩れる UX 影響、Step 11-A 残課題のため）

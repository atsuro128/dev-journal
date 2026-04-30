# Admin ダッシュボードを PC 画面サイズで開くとカードの幅が他ロールより短い（共通幅未適用）

## 発見日
2026-04-30

## カテゴリ
ui-design / frontend / regression-candidate

## 影響度
中（機能影響なし。Admin で見た目の違和感、ポートフォリオデモで「ロール別に整合性が取れていない」印象）

## 発見経緯
user-report / Step 11-A SMK 再検証中、Admin でダッシュボードを PC 画面サイズで開いたところ、カードの幅が他ロール（Member / Approver / Accounting）と比較して短く、共通幅（Grid レイアウト想定）が適用されていないとユーザーが発見

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

PC 画面サイズ（広画面）で Admin としてダッシュボード（SCR-DASH-001）を開くと、カウントカード（または ActionCountCard）の **幅が他ロールより明らかに短い**。他ロールでは発生しない（Member / Approver / Accounting 表示は共通幅で整っている）。

### 推定原因

| 推定 | 内容 |
|------|------|
| A | DashboardPage のレイアウトで Admin 表示時のみ Grid `xs/md` プロパティが異なる |
| B | Admin で表示されるカード数が少ない（テナント全体カードのみ等）ため、Grid 内で flex 縮小して幅が縮む |
| C | ロール別の `useDashboard()` レスポンスで Admin 用のカードコンポーネントだけ独自スタイル（幅指定）を持っている |
| D | 前回セッション #153 修正（PR #108 / commit `6916ed0`）の Approver 向けカード幅統一が、Admin の表示パターンに副作用を与えた |

### 検証で確認すべきこと

- `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx` のロール別カード表示分岐
- 各カウントカード / ActionCountCard コンポーネントの幅指定（sx / Grid props）
- Admin 表示時のカード数（1〜2 枚と他ロールで枚数が違う場合、Grid の `xs={6}` 等が flex で縮小される）
- DevTools で computed style の `width` を Member/Approver/Accounting/Admin で比較

### 影響範囲

- **発生**: Admin ダッシュボード PC 幅
- **発生しない**: Member / Approver / Accounting

カード共通基盤（CountCard / ActionCountCard）が **Approver 等で再描画されることが多い**ため、レイアウトコンテナ側（Grid / Box）の Admin 用分岐が原因の可能性が高い。

## 影響

- Admin ダッシュボードで「画面が中途半端に見える」UX 違和感
- 他ロールで整っている見た目との一貫性欠如
- ポートフォリオデモで Admin ロールを見せたときに違和感を与える

## 提案

### Step 1: 実装の現状把握

- DashboardPage のロール別レイアウト分岐を全件読む
- 各 CountCard / ActionCountCard の幅指定を確認
- Admin で表示されるカード数（枚数）の確認 → 1〜2 枚なら `xs={12} md={6}` 程度に変更してフル幅 / 半分幅で揃える

### Step 2: 修正

| ケース | 修正 |
|--------|------|
| Admin だけカード数が少なく Grid の幅が縮む | Admin 表示時に Grid `xs/md` プロパティを大きくして他ロールと同じ視覚的幅に揃える |
| カードコンポーネント自体に幅指定がある | sx の `width` / `maxWidth` を見直す |
| #153 修正の副作用 | 前回コミット差分を確認し、影響範囲を限定 |

### 推奨方針

- **DashboardPage 側で Admin の Grid 配置を他ロールと同じ視覚的幅に揃える**（カード枚数が少ない場合は `xs={12}` でフル幅 or `xs={12} md={6}` で半分幅）
- カードコンポーネント（CountCard / ActionCountCard）共通基盤側に幅指定を入れず、コンテナ（DashboardPage の Grid）で制御する責務分離を維持

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx` | Admin ロール表示時の Grid `xs/md` 調整 |
| `expense-saas/frontend/src/pages/dashboard/__tests__/DashboardPage.test.tsx` | Admin 表示時のカード幅検証テスト追加（DOM スナップショット or `style.width` 確認） |
| `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` §ロール別レイアウト | Admin 表示時の Grid 配置仕様を明文化（他ロールと同じ視覚的幅） |
| `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` §4.2 / §9.4（Admin） | 同上 |

## 完了条件

- Admin ダッシュボード PC 幅でカードの幅が他ロールと視覚的に同等
- DevTools の computed `width` で Admin と他ロールが同等の値（または用途上整合する値）に揃う
- 設計書（dashboard.md）に Admin 表示時の Grid 配置が明文化される
- 既存テスト（DashboardPage / 各 CountCard）に影響なし

## ラベル

- type: bug / regression-candidate / ui / responsive
- area: frontend

## 関連

- 関連 issue: #153（ダッシュボード「承認待ち」カードの見た目幅が他カードより狭く描画される、Approver で発生・修正済み）
- 関連 PR: #108（前回セッションの Dashboard Badge / カード幅修正、commit `6916ed0`）
- 関連 SMK: なし（SMK 再検証中の副次発見）

## MVP 区分

MVP（ロール別整合性、ポートフォリオデモ印象に直結）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

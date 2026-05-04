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

**採用方針**: TenantStatusCards Grid 配置変更（5 等分）+ Admin メンバー数カードの Grid ラップ追加

**真因確定**:
- 真因 1: `TenantStatusCards.tsx` L34-72 の Grid `size={{ xs: 12, sm: 4, md: 'auto' }}` の `md: 'auto'` がコンテンツ自然幅扱いで縮小
- 真因 2: `DashboardPage.tsx` L156-160 の Admin「メンバー数」CountCard が Grid ラップなしで block 要素剥き出し
- **PR #108（commit `6916ed0`）は無関係**（issue 推定 D は false、本問題は PR #108 以前から潜在）

**実装** (PR #112, commit `bc208b0`):
- `TenantStatusCards.tsx`: Grid `size={{ xs: 12, sm: 6, md: 2.4 }}`（5 等分）に変更
- `DashboardPage.tsx`: Admin メンバー数カードを `<Grid container>` + `<Grid size={{ xs: 12, sm: 4 }}>` でラップ
- カード共通基盤（CountCard）に幅指定は入れず、コンテナ Grid 側で制御する責務分離維持

**テスト**:
- DSH-FE-NEW-A1（TenantStatusCards 5 枚 Grid 配置検証）+ DSH-FE-NEW-A2（Admin メンバー数 Grid ラップ検証）追加
- DashboardPage.test.tsx + TenantStatusCards 関連 47 件 PASS

**設計書改訂** (commit `9edf4a5`):
- `55_ui_component/screens/dashboard.md` §3 / §9 に Grid 配置仕様と責務分離規定を追記
- `50_detail_design/screens/dashboard.md` §4.4 / §4.5 / §9.4 に同等の追記

**注意**: jsdom 制約により実 width 値検証は不可。視覚的な幅検証は SMK 再検証時 / DevTools セルフレビューで実施

**SMK 再検証要項目**: 新規（既存 SMK では未カバー、UAT で確認）

## 解決日

2026-04-30

---

## 再対応（2026-05-03、DevTools 視覚確認 FAIL）

### 状態

**FAIL**（PR #112 マージ後の DevTools 視覚確認で、Admin だけ 1 行目に 5 カードが詰まっており他ロールと幅が揃っていないことを確認、再 open）

### 観察した挙動

| ロール | 1 行目構成 | 2 行目以降 |
|---|---|---|
| **Member** | MyReportCountCards 3 枚（1/3 幅、`sm: 4`） | （なし、MonthlySummary は表示しない） |
| **Approver** | MyReportCountCards 3 枚（1/3 幅、`sm: 4`） | 承認待ち 1 枚 + MonthlySummary + RecentReportList |
| **Accounting** | MyReportCountCards 3 枚（1/3 幅、`sm: 4`） | 支払待ち 1 枚 + MonthlySummary + RecentReportList |
| **Admin** | **TenantStatusCards 5 枚（1/5 幅、`md: 2.4`）** ← 他ロールと幅が違う | メンバー数 1 枚 + MonthlySummary |

`TenantStatusCards.tsx` L33-34 の規定:
> PC 幅（md ≥ 900px）で 5 等分（12 / 5 = 2.4）、タブレット幅（sm）で 2 列折返し、モバイルで縦積み。

つまり「Admin の 5 カードを 1 行に並べる」という設計が PR #112 で意図的に採用されたが、**他ロールが 1/3 幅基準で構築されているため、Admin だけ視覚的に細く見える不整合**が残っていた。

### 真因

PR #112 の修正方針（Admin の 5 カードを 1 行 5 等分にする）は、**他ロールの 1/3 幅基準と整合しない**設計判断だった。カード数の差（5 vs 3）から来る幅の差を、他ロールに合わせて折返しで解消するのが正しい方向。

### 採用方針

`TenantStatusCards.tsx` の Grid size を `md: 2.4` → `md: 4` に変更し、**5 カードを 3 + 2 の 2 行レイアウト** に変更する。

```tsx
// 修正前
<Grid size={{ xs: 12, sm: 6, md: 2.4 }}>

// 修正後
<Grid size={{ xs: 12, sm: 6, md: 4 }}>
```

これで Admin 表示時：

| ロール | 1 行目 | 2 行目 | 3 行目 |
|---|---|---|---|
| **Admin** | TenantStatus 3 枚（1/3 幅） | TenantStatus 残 2 枚 + 空き 1 枚分 | メンバー数 1 枚（1/3 幅） |

他ロールと **カード幅が完全に揃う**（全画面で 1/3 幅基準）。

### 修正対象ファイル

| ファイル | 変更内容 |
|---|---|
| `expense-saas/frontend/src/components/dashboard/TenantStatusCards.tsx` | Grid size を `md: 2.4` → `md: 4` に変更（5 箇所） + コメント L33-34 の規定更新 |
| `expense-saas/frontend/src/components/dashboard/__tests__/TenantStatusCards.test.tsx`（または相当） | レイアウト関連テストがあれば 5 等分 → 3 等分に期待値更新 |
| `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` §TenantStatusCards | レイアウト規定を「PC 幅で 5 等分」から「PC 幅で 3 等分（3 + 2 の 2 行）」に変更 |

### 設計コメント（参考）

ユーザーから「本来はカード表示はコンポーネント化して、ドメインによって表示するカードを変える作りならこんなこと起こらなかった」との指摘あり。MyReportCountCards / TenantStatusCards という別コンポーネント分割が今回の不整合の温床になった。MVP 内では現アーキテクチャを維持するが、Phase 2 以降のリファクタリングで「共通カードコンテナ + props で表示内容を差別化」する設計を検討する余地がある（post-MVP）。

### SMK 再検証要項目

DevTools 視覚確認（既存 SMK では未カバー）: PC 幅で Admin / Member / Approver / Accounting のダッシュボードを順に開き、カード幅が揃っていること

### 起票日（再 open）

2026-05-03

---

## 解決確認（2026-05-04）

### 状態
**解決（resolved 移動）**

### 解決根拠
PR #125（TenantStatusCards Grid size を `md: 2.4` → `md: 4` に変更し 5 カードを 3 + 2 の 2 行レイアウトにして他ロール 1/3 幅と統一）マージ後、2026-05-03 に DevTools 視覚再確認を実施して PASS。Admin / Member / Approver / Accounting のカード幅が揃っていることを確認。

### 関連 PR / コミット
- PR #112（commit `bc208b0`）: TenantStatusCards を 5 等分（md: 2.4）に変更（初回対応）→ 視覚 FAIL で再対応
- PR #125: TenantStatusCards Grid を md: 4（3 + 2 レイアウト）に変更（再対応・解決）

### 備考
Admin のカード 5 枚を 5 等分 1 行にする方針（PR #112）は他ロールの 1/3 幅基準と整合しなかったため、折返し 2 行レイアウトに変更して解決。

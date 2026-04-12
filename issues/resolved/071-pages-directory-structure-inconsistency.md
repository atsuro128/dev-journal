# pages ディレクトリ構成が3パターン混在している

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 10（機能実装）/ Step 11（システムテスト・UAT — 11-A ローカル動作確認）

## ブロッカー
070（LoginPage ルーティング修正）のマージ後に着手

## 問題

`frontend/src/pages/` 配下のディレクトリ構成が、Step 10 の並列実装で3パターン混在している。

### パターンA: サブディレクトリに全部入り
- `login/` — LoginPage.tsx, LoginForm.tsx, `__tests__/`
- `signup/` — SignupPage.tsx, SignupForm.tsx, `__tests__/`
- `password-reset/` — 各ページ・フォーム・`__tests__/`
- `admin/` — TenantPage.tsx, AllReportsPage.tsx, 子コンポーネント, `__tests__/`

### パターンB: Page 直置き + 子コンポーネントだけサブディレクトリ
- `ReportListPage.tsx`, `ReportCreatePage.tsx` 等 — pages 直下
- `reports/` — 子コンポーネント（ReportInfoCard 等）+ `__tests__/`
- `pages/__tests__/` — Page のテスト

### パターンC: Page 直置き + テストだけサブディレクトリ
- `DashboardPage.tsx` — pages 直下
- `dashboard/__tests__/` — テストのみ

### パターンBとCのワークフロー系
- `ApprovalListPage.tsx`, `PaymentListPage.tsx` — pages 直下
- `pages/__tests__/` — テスト

## 影響

- 開発者がファイルを探す際に構成が予測できない
- 新規ページ追加時にどのパターンに従うか判断できない
- architecture.md の設計（フラット配置）と実態が乖離

## 提案

パターン A（サブディレクトリに全部入り）に統一する。テスト配置はコロケーション方式（issue 059 で確定済み）。

### 移動計画

```
pages/
  dashboard/
    DashboardPage.tsx          ← pages/ から移動
    __tests__/                 ← 既存
  reports/
    ReportListPage.tsx         ← pages/ から移動
    ReportCreatePage.tsx       ← pages/ から移動
    ReportDetailPage.tsx       ← pages/ から移動
    ReportEditPage.tsx         ← pages/ から移動
    ReportInfoCard.tsx         ← 既存
    ...（既存の子コンポーネント）
    __tests__/                 ← 既存 + pages/__tests__/Report*.test.tsx を移動
  workflow/
    ApprovalListPage.tsx       ← pages/ から移動
    PaymentListPage.tsx        ← pages/ から移動
    __tests__/                 ← pages/__tests__/Approval*.test.tsx, Payment*.test.tsx を移動
  login/                       ← 既存（変更なし）
  signup/                      ← 既存（変更なし）
  password-reset/              ← 既存（変更なし）
  auth/                        ← 既存（変更なし）
  admin/                       ← 既存（変更なし）
```

### 付随作業
- App.tsx の import パスを全て更新
- 各テストファイルの import パスを更新
- architecture.md のディレクトリツリーを更新（dev-journal 側）

---

## 解決内容

PR #42 でパターン A（サブディレクトリに全部入り）に統一。全ページをサブディレクトリに移動し、App.tsx・テストファイルの import パスを更新。vi.mock パス未更新による CI 失敗を修正後マージ。

## 解決日
2026-04-11

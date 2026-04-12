# App.tsx に管理系画面のルートが未登録

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
高

## 発見経緯
review

## 関連ステップ
Step 10（機能実装）/ Step 11（システムテスト・UAT — 11-A ローカル動作確認）

## ブロッカー
071（pages ディレクトリ整理）のマージ後に着手

## 問題

`screens.md` で定義されている以下2画面の Route が `App.tsx` に存在しない。

| 画面 ID | 画面名 | URL パス | 実装ファイル |
|---------|--------|---------|------------|
| SCR-ADM-001 | 全レポート一覧 | `/reports/all` | `pages/admin/AllReportsPage.tsx` |
| SCR-ADM-002 | テナント設定 | `/settings/tenant` | `pages/admin/TenantPage.tsx` |

コンポーネント自体は `pages/admin/` に実装済みだが、App.tsx にルートが登録されていないためブラウザからアクセスできない。

## 影響

- 管理者（admin）が全レポート一覧・テナント設定画面にアクセスできない
- issue 070（LoginPage）と同種の問題（実装済みだがルーティング未接続）

## 提案

1. `App.tsx` に以下のルートを追加:
   - `<Route path="/reports/all" element={<AllReportsPage />} />`
   - `<Route path="/settings/tenant" element={<TenantPage />} />`
2. import パスを追加

---

## 解決内容

PR #43 で AllReportsPage（/reports/all）と TenantPage（/settings/tenant）のルートを App.tsx に追加。master とのコンフリクト解消後にマージ。

## 解決日
2026-04-11

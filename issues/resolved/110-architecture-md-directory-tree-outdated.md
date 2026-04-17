# architecture.md のフロントエンドディレクトリツリーが実装と乖離

## 発見日
2026-04-16

## カテゴリ
detail-design

## 影響度
低

## 発見経緯
proactive — 過去セッションログの引き継ぎ漏れ調査中に発見。

## 関連ステップ
Step 3（アーキテクチャ設計）

## ブロッカー
なし

## 問題

`dev-journal/deliverables/docs/30_arch/architecture.md` L226-234 のフロントエンドディレクトリツリーが実装と乖離している。

### 設計書の記載（旧構造）
```
pages/
├── LoginPage.tsx
├── DashboardPage.tsx
├── ReportListPage.tsx
├── ReportDetailPage.tsx
├── ReportCreatePage.tsx
├── ReportEditPage.tsx
├── ApprovalListPage.tsx
└── PaymentListPage.tsx
```

### 実装の構造（現行）
```
pages/
├── admin/
├── auth/
├── dashboard/
├── error/
├── login/
├── password-reset/
├── reports/
├── signup/
└── workflow/
```

また `components/`、`hooks/`、`api/`、`stores/` の記載も簡略化されたままで、実装で追加されたファイル群が反映されていない。

## 影響

設計書を参照して実装構造を把握しようとすると誤解を招く。

## 提案

architecture.md の §4.1 フロントエンドディレクトリツリーを、実装の現行構造に合わせて更新する。全ファイルを列挙する必要はなく、ディレクトリ階層とその役割の記述を現行に合わせれば十分。

## 完了条件

- architecture.md のフロントエンドディレクトリツリーが実装と一致している
- 新規追加されたディレクトリ（error/ 等）が反映されている

---

## 解決内容

## 解決日

### 2026-04-16 Codex Issue 解決再レビュー

- `dev-journal/deliverables/docs/30_arch/architecture.md` §4.1 を確認し、`pages/` が現行のドメイン別ディレクトリ構成（`login/`, `signup/`, `password-reset/`, `auth/`, `dashboard/`, `reports/`, `admin/`, `workflow/`, `error/`）へ更新されていることを確認。
- `components/`, `hooks/`, `api/`, `stores/`, `contexts/`, `lib/`, `test/` とフロントエンド直下の `theme.ts`, `vite.config.ts`, `package.json` なども現行構成に沿って記載されており、Issue 本文で指摘した旧フラット構造は解消されている。
- 追加の不整合は見当たらず、設計書を参照した実装構造の把握を妨げない状態になっているため PASS と判定。

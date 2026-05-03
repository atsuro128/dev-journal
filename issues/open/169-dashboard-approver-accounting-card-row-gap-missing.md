# Approver / Accounting ダッシュボードでカード行間の余白が不在（MyReportCountCards と ActionCards がくっつく）

## 発見日
2026-05-03

## カテゴリ
ui-design

## 影響度
中（視覚的整合性、機能影響なし）

## 発見経緯
user-report / issue #168（カード border 高さズレ）対応の SMK 視覚再検証中、ユーザーが「Admin とそれ以外でダッシュボードカードのマージン？が異なる。マージンではないが、カード同士の余白が Admin では設定されているのに対して、他のドメインでは 1 行目と 2 行目のカードがくっついている」と報告。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

Approver / Accounting のダッシュボードで、MyReportCountCards（1 行目カードグループ）と ActionCards（2 行目カードグループ）の間に余白がなく、2 つの Grid container がくっついて表示される。Admin では同箇所に余白が存在しており、ロール間で視覚的整合性が取れていない。

### 原因

`frontend/src/pages/dashboard/DashboardPage.tsx` で 2 つの Grid container を縦並びにする箇所のうち、**Admin だけ `<Box sx={{ mt: 2 }}>` でラップしている**が、Approver / Accounting ではラップしておらず、`Grid container spacing={2}` は同一 container 内のアイテム間にのみ効くため container 間（縦方向）には間隔が出ない。

| ロール | カードグループ構造 | 行間 |
|---|---|---|
| Member | `MyReportCountCards` 1 Grid のみ | 該当なし（1 行） |
| Approver | `MyReportCountCards` Grid + `ApproverActionCards` Grid（直後） | 不在 |
| Accounting | `MyReportCountCards` Grid + `AccountingActionCards` Grid（直後） | 不在 |
| Admin | `TenantStatusCards` Grid + `<Box sx={{ mt: 2 }}>` で member count Grid をラップ | あり |

### 影響範囲

- **発生**: Approver / Accounting ダッシュボード
- **発生しない**: Member（カードグループが 1 つのみ）、Admin（`mt: 2` ラップ済み）

## 影響

- Approver / Accounting のダッシュボードでカードグループ間の余白が欠如し、視覚的に詰まった印象になる
- Admin との整合性欠如（同一コンポーネント内でロール別に挙動が異なる）
- ポートフォリオデモでロール切替時に違和感を与える

## 提案

### 採用方針（案 A — 最小修正）

Approver / Accounting の ActionCards Grid を `<Box sx={{ mt: 2 }}>` でラップ（Admin と同じパターン）。

```tsx
{isApprover && (
  <Box sx={{ mt: 2 }}>
    <Grid container spacing={2} data-testid="approver-action-cards">
      ...
    </Grid>
  </Box>
)}

{isAccounting && (
  <Box sx={{ mt: 2 }}>
    <Grid container spacing={2} data-testid="accounting-action-cards">
      ...
    </Grid>
  </Box>
)}
```

### 比較案

- **案 B**: カード行コンテナを共通コンポーネント化（`<DashboardCardRow>`）→ post-MVP リファクタリングとして既存方針通り棚上げ。本 issue は MVP 内での即時修正（案 A）を採用

## 修正対象ファイル

| ファイル | 変更内容 |
|---|---|
| `frontend/src/pages/dashboard/DashboardPage.tsx` L120-145（Approver / Accounting 両ブロック） | ActionCards Grid を `<Box sx={{ mt: 2 }}>` でラップ |
| `frontend/src/pages/dashboard/__tests__/DashboardPage.test.tsx` | Approver / Accounting の DOM 構造アサーションがあれば追従 |

## 完了条件

- Approver / Accounting ダッシュボードで MyReportCountCards と ActionCards の間に Admin と同等の余白（`mt: 2` = 16px）が表示される
- DevTools 視覚確認: Approver / Accounting の余白が Admin と同等
- 既存テストへの影響なし

## SMK 再検証要項目

- ダッシュボード（Approver / Accounting）の DevTools 視覚確認: MyReportCountCards と ActionCards の間に Admin と同等の余白が表示

## 関連

- issue #168（CountCard border 高さズレ）— 同種の整合性 issue、本 issue の発見契機
- 既存方針: 「カード共通化は post-MVP」（session-log 2026-05-02 学び欄）
- 設計書: `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md`
- 関連 PR (TBD): `step11/issue-169-dashboard-cards-row-gap`

## MVP 区分

MVP（ロール別整合性、ポートフォリオデモ印象に直結）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

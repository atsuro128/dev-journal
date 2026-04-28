# ダッシュボードの月別合計テーブルと直近レポート一覧の境界が不明瞭（セクション見出し・列見出し欠落 + 隣接配置）

## 発見日
2026-04-28

## カテゴリ
ui-design / frontend

## 影響度
中（機能影響なし。情報構造が不明瞭で、ユーザーがどのテーブルが何の情報か判別しづらい）

## 発見経緯
user-report / Step 11-A SMK-011 検証時に Approver でダッシュボードを観察。Member ロールでは月別合計テーブルが表示されないため気付かなかったが、Approver ではカウントカードの直下に月別合計テーブル + その直下に直近レポート一覧が並び、両者の境界が視覚的に不明瞭で、列見出しの有無も不揃いという指摘

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 該当箇所

**1. `MonthlySummaryTable.tsx`**
- 列見出し（年月 / 合計金額）は **あり**（L57-60、`<TableHead>`）
- **コンポーネント自体のセクション見出しは なし**

**2. `RecentReportList.tsx`**
- TableHead **そのものなし**（L48 で `<TableBody>` のみ）。**列見出しもなし**
- **セクション見出しもなし**
- 末尾に「すべてのレポートを見る」リンクのみ

**3. `DashboardPage.tsx:166-174`**

```tsx
{/* Approver / Accounting / Admin: 月別サマリー */}
{isApproverOrAccountingOrAdmin && (
  <MonthlySummaryTable items={monthlySummaryItems} />
)}

{/* Member / Approver / Accounting: 最近のレポート一覧 */}
{isMemberLike && (
  <RecentReportList reports={recentReports} />
)}
```

→ 両テーブルが **間に余白もマージンも見出しも挟まず連続配置** されている。

### 観察

Approver / Accounting / Admin（月別サマリ表示ロール）でダッシュボードを開くと、画面構成は以下:

1. カウントカード（承認待ち / 支払待ち / テナント全体）
2. 直下に **列見出しありの月別合計テーブル**（年月・合計金額）
3. 直下に **列見出しなしの直近レポート一覧**（タイトル / 期間 / 金額 / ステータスがただの行リストで並ぶ）

両者が密接配置され、かつ後者は列見出しすらないため、ユーザーは「これが何のテーブルか」を即座に判別しづらい。

### 影響範囲

- **発生**: Approver / Accounting / Admin（月別サマリ表示ロール）
- **発生しない**: Member（月別サマリ非表示。カード直下が直近レポート一覧で、文脈上自明）

### 過去経緯

過去 issue **#145**（ReportWorkflowInfo セクション見出し追加、post-MVP）と類似の「セクション見出し欠落」課題。本件は #145 より影響が大きいため MVP 範囲。

## 影響

- ロール別動線（Approver / Accounting / Admin）でダッシュボードの主要情報がパッと読み取れない
- セクション見出しがないことで「このテーブルは何の情報か」をユーザーが推測する必要がある
- 列見出しの有無が不揃いで、ダッシュボード全体の情報構造の質が低く感じられる（ポートフォリオデモとしての印象も低下）

## 提案

### Step 1: 設計書での意図確認

`dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` および `50_detail_design/screens/dashboard.md` を確認:

- 月別合計テーブル / 直近レポート一覧のセクション見出し・列見出しの規定
- 規定がない場合は設計書側に追記が必要

### Step 2: 実装修正

**修正案A: 各コンポーネント内にセクション見出しを追加（推奨）**

```tsx
// MonthlySummaryTable.tsx
<Box>
  <Typography variant="h6" sx={{ mb: 1 }}>月別支出サマリー</Typography>
  <TableContainer ...>...</TableContainer>
</Box>

// RecentReportList.tsx
<Box>
  <Typography variant="h6" sx={{ mb: 1 }}>最近のレポート</Typography>
  <TableContainer ...>
    <Table>
      <TableHead>
        <TableRow>
          <TableCell>タイトル</TableCell>
          <TableCell>期間</TableCell>
          <TableCell align="right">金額</TableCell>
          <TableCell>ステータス</TableCell>
        </TableRow>
      </TableHead>
      <TableBody>...</TableBody>
    </Table>
  </TableContainer>
</Box>
```

メリット: コンポーネント自体が独立して読めるようになる
デメリット: コンポーネント仕様変更（既存テスト要更新）

**修正案B: DashboardPage 側でセクション見出しと余白を追加**

```tsx
{isApproverOrAccountingOrAdmin && (
  <Box sx={{ mt: 3 }}>
    <Typography variant="h6" sx={{ mb: 1 }}>月別支出サマリー</Typography>
    <MonthlySummaryTable items={monthlySummaryItems} />
  </Box>
)}

{isMemberLike && (
  <Box sx={{ mt: 3 }}>
    <Typography variant="h6" sx={{ mb: 1 }}>最近のレポート</Typography>
    <RecentReportList reports={recentReports} />
  </Box>
)}
```

メリット: コンポーネント側を変更せず、配置側で構造を整える
デメリット: 列見出し追加（RecentReportList の TableHead 追加）は別途必要

### 推奨

**修正案A（コンポーネント内に見出し追加）+ RecentReportList に列見出し追加** を推奨。コンポーネント単体で意味のある UI を成すため、再利用性も高まる。

設計書 `dashboard.md` で見出し有無の規定を確認 → 規定がなければ追補。

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/dashboard/MonthlySummaryTable.tsx` | セクション見出し追加 |
| `expense-saas/frontend/src/components/dashboard/RecentReportList.tsx` | セクション見出し + 列見出し追加 |
| `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx` | セクション間の余白調整（mt: 3 等） |
| `expense-saas/frontend/src/components/dashboard/__tests__/MonthlySummaryTable.test.tsx` | 見出しレンダリングテスト更新 |
| `expense-saas/frontend/src/components/dashboard/__tests__/RecentReportList.test.tsx` | 見出し + 列見出しレンダリングテスト追加 |
| `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` | 見出し仕様規定の追補（必要に応じて） |
| `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` | 同上 |

## 完了条件

- 月別合計テーブルにセクション見出しが表示される
- 直近レポート一覧にセクション見出しと列見出しが表示される
- 両テーブル間に十分な余白があり、視覚的に分離されている
- 設計書で見出し仕様が規定されている
- 既存テストが通る + 必要に応じて新規テスト追加

## ラベル

- type: ui / ux
- area: frontend / docs（設計確認）

## 関連

- 過去 issue: #145（ReportWorkflowInfo セクション見出し追加、post-MVP）— 同種の「セクション見出し欠落」課題
- 関連 SMK: SMK-011（二重押下防止）の副次発見

## MVP 区分

MVP（情報構造が不明瞭で、ポートフォリオデモとしての印象に直結）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

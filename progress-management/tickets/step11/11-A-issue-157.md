# issue #157: ダッシュボードのテーブルセクション境界明確化

- 担当: frontend-developer（FE 実装） + 指揮役（設計書改訂を dev-journal/master 直接編集）
- 依存: なし（issue #158 と並列実行可能。修正対象ファイルは独立）
- ブランチ: `step11/157-dashboard-section-headings`（expense-saas）
- 出力先:
  - 設計書改訂: `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md`, `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md`（dev-journal/master 直接編集）
  - FE 実装: `expense-saas/frontend/src/components/dashboard/MonthlySummaryTable.tsx`, `RecentReportList.tsx`, `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx`（worktree 隔離ブランチ）
  - テスト追加・更新: `expense-saas/frontend/src/components/dashboard/__tests__/MonthlySummaryTable.test.tsx`, `RecentReportList.test.tsx`
- 元資料: `dev-journal/issues/open/157-dashboard-table-section-boundary-unclear.md`（採用方針: 案A）
- テンプレート: `ai-dev-framework/templates/ticket-template.md`

## 入力

| 資料 | パス | 参照箇所 | 用途 |
|------|------|----------|------|
| issue 本文 | `dev-journal/issues/open/157-dashboard-table-section-boundary-unclear.md` | 全文（特に「修正案A」「完了条件」） | 採用方針・受け入れ条件の確定 |
| UI コンポーネント設計 | `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` | §3 MonthlySummaryTable / RecentReportList の責務定義、§9 設計識別子 | 見出し追加に伴うコンポーネント責務の追補対象を確定 |
| 画面詳細設計 | `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` | §4.6, §4.7（表示仕様）、§9 ロール別画面構成サマリ | 見出し仕様（タイトル文言・配置）を追補 |
| 既存実装（FE） | `expense-saas/frontend/src/components/dashboard/MonthlySummaryTable.tsx` | L44-72 | セクション見出し追加箇所（コンポーネントルートを `<Box>` でラップしてタイトル + テーブルを配置） |
| 既存実装（FE） | `expense-saas/frontend/src/components/dashboard/RecentReportList.tsx` | L40-71 | セクション見出し + `<TableHead>` 列見出しの追加箇所 |
| 既存実装（FE） | `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx` | L166-174 | 両セクション間の余白付与（`mt: 3` 相当）。Approver 用承認待ちカード・Accounting 用支払待ちカードと月別サマリーの間にも余白を確認 |
| 既存テスト | `expense-saas/frontend/src/components/dashboard/__tests__/MonthlySummaryTable.test.tsx` | L15-62（DSH-FE-022〜026） | 既存テストの破壊的変更回避（`screen.getAllByRole('row')` の件数前提を維持） |
| 既存テスト | `expense-saas/frontend/src/components/dashboard/__tests__/RecentReportList.test.tsx` | L23-49（DSH-FE-027〜029） | TableHead 追加で行数が変わる点に注意したアサーション再設計 |
| 共通実装ルール | `.claude/rules/implementation-workflow.md` | worktree 作業ルール / コメント言語 / FE エラーハンドリング | worktree 隔離での作業手順遵守 |

## 責務

### 含めること

- `MonthlySummaryTable.tsx` のルートを `<Box>` でラップし、`<Typography variant="h6">月別支出サマリー</Typography>`（`sx={{ mb: 1 }}`）をテーブル上部に追加する
- `RecentReportList.tsx` のルートを `<Box>` でラップし、`<Typography variant="h6">最近のレポート</Typography>`（`sx={{ mb: 1 }}`）をリスト上部に追加する
- `RecentReportList.tsx` の `<Table>` に `<TableHead>` を追加し、列見出し「タイトル / 期間 / 金額（right 寄せ）/ ステータス」を表示する
- `DashboardPage.tsx` で `MonthlySummaryTable` と `RecentReportList` の各レンダリング箇所を `<Box sx={{ mt: 3 }}>` でラップし、上部セクション（カウントカード・承認/支払待ちカード）からの視覚的余白を確保する
- 設計書 2 本（`55_ui_component/screens/dashboard.md`, `50_detail_design/screens/dashboard.md`）にセクション見出し・列見出しの規定を追補する
- 既存テスト（DSH-FE-022〜026 / DSH-FE-027〜029）を新仕様に合わせて更新し、見出し検証アサーションを追加する

### 含めないこと

- ダッシュボード UI の大幅再設計（カラースキーム変更・Card 化・グラフ追加など）
- `MonthlySummaryTable` 0 件時の見出し表示方針変更（既存「データがありません」のみ表示）
  - 0 件時は見出しを表示しないか / 表示するかは MVP 範囲内で迷いが少ない案 A（**0 件時も見出しを表示する**）を採用する。理由: 「何のセクションが空なのか」を伝えるほうが情報構造として自然
- `RecentReportList` 0 件時の `EmptyState` 表示方針の変更（既存仕様維持）
- 上記以外の画面（issue #158 含む他 issue 対応）への波及修正
- E2E テスト追加（DOM 構造変化の確認はユニットテストで完結する）

## 完了条件

issue 本文「完了条件」5 項目に以下のマッピングで対応する（DoD 6 項目に分解）。

- [ ] **DoD-1（issue 完了条件 1）**: `MonthlySummaryTable` をレンダリングしたときに「月別支出サマリー」セクション見出しが表示される（`MonthlySummaryTable.test.tsx` で `screen.getByText('月別支出サマリー')` を検証）
- [ ] **DoD-2（issue 完了条件 2 前半）**: `RecentReportList` をレンダリングしたときに「最近のレポート」セクション見出しが表示される（`RecentReportList.test.tsx` で `screen.getByText('最近のレポート')` を検証）
- [ ] **DoD-3（issue 完了条件 2 後半）**: `RecentReportList` のテーブルに列見出し「タイトル / 期間 / 金額 / ステータス」が表示される（`RecentReportList.test.tsx` で 4 列の列見出しを検証 + ヘッダー行込みの `getAllByRole('row')` 件数確認）
- [ ] **DoD-4（issue 完了条件 3）**: `DashboardPage` 上で `MonthlySummaryTable` と `RecentReportList` の前に `mt: 3` 相当の余白が付与され、両テーブル間に視覚的な分離がある。**検証手段**: DashboardPage の差分が `<Box sx={{ mt: 3 }}>` ラップ追加のみとなる小さな変更のため、PR 差分レビュー（指揮役 + reviewer）でラップ存在を確認する。新規ユニットテスト追加は不要（既存 DashboardPage 専用テストがないため。E2E スナップショットへの依存もしない）
- [ ] **DoD-5（issue 完了条件 4）**: 設計書 2 本に見出し仕様の追補が反映されている（後述「設計書改訂方針」のセクション）
- [ ] **DoD-6（issue 完了条件 5）**: 既存テスト（DSH-FE-022〜029）が全て PASS し、見出し追加分の新規アサーションも PASS する（`npm run test --workspace=frontend` で対象テストファイルが緑になる）
- [ ] PR 本文に DoD 6 項目のチェックボックスを記載し、レビュー時に確認可能な状態にする

---

## 影響範囲

### コード（expense-saas）

- `frontend/src/components/dashboard/MonthlySummaryTable.tsx` — セクション見出し追加
- `frontend/src/components/dashboard/RecentReportList.tsx` — セクション見出し + `<TableHead>` 列見出し追加
- `frontend/src/pages/dashboard/DashboardPage.tsx` — 両セクションのラップ `<Box sx={{ mt: 3 }}>` 追加（L167-174）
- `frontend/src/components/dashboard/__tests__/MonthlySummaryTable.test.tsx` — 見出し検証追加
- `frontend/src/components/dashboard/__tests__/RecentReportList.test.tsx` — 見出し + 列見出し検証追加（行数アサーション再計算）

### 設計書（dev-journal/master 直接編集）

- `deliverables/docs/55_ui_component/screens/dashboard.md`
  - §3 `MonthlySummaryTable` / `RecentReportList` の「責務」記述に **コンポーネント自体がセクション見出しを内包する** 旨を追記
  - §3 `RecentReportList` のコンポーネントツリー（§2）に列見出し（`TableHead`）を表現
  - §9 設計識別子テーブルの該当行（`§MonthlySummaryTable` / `§RecentReportList`）の「テスト対象の概要」にセクション見出し検証を追記
- `deliverables/docs/50_detail_design/screens/dashboard.md`
  - §4.6 月別支出サマリーの「表示仕様」に **セクション見出し「月別支出サマリー」を表示する** を追加
  - §4.7 直近レポート一覧の「表示仕様」に **セクション見出し「最近のレポート」を表示する**、および列見出し（タイトル / 対象期間 / 合計金額 / ステータス）を表示する旨を明記（既存の「列名」テーブル §4.7 冒頭は仕様としては記載済みだが、UI 上の列ヘッダー描画の明示は未記載）
  - §9.2/§9.3/§9.4 のロール別画面構成サマリ（テキスト図）でセクション見出し有無の追補は行わない（テキスト図の精緻化は post-MVP）

### 関連 issue / PR

- 過去 issue #145（ReportWorkflowInfo セクション見出し追加、post-MVP）と同種パターン。本 issue は MVP 区分のため #145 より優先する
- issue #158 と並列実行可能（修正対象ファイル重複なし）

---

## 設計書改訂方針

### `55_ui_component/screens/dashboard.md`

- **§3 MonthlySummaryTable**: 「責務」末尾に以下を追記
  > ルート要素は `<Box>` で、上部にセクション見出し `<Typography variant="h6">月別支出サマリー</Typography>` を配置し、その下に `<TableContainer>` を置く。0 件時もセクション見出しは表示する。
- **§3 RecentReportList**: 「責務」末尾に以下を追記
  > ルート要素は `<Box>` で、上部にセクション見出し `<Typography variant="h6">最近のレポート</Typography>` を配置する。`<Table>` には `<TableHead>` を含み、列見出し（タイトル / 期間 / 金額 / ステータス）を表示する。
- **§2 コンポーネントツリー**: `RecentReportList` 配下に `TableHead`（列見出し）を追記
- **§9 テスト追跡用 設計識別子**: `§MonthlySummaryTable` / `§RecentReportList` の「テスト対象の概要」に「セクション見出しの描画」「列見出しの描画」を追記

### `50_detail_design/screens/dashboard.md`

- **§4.6 月別支出サマリー — 表示仕様**: 既存リストの先頭に
  > - セクション見出し「月別支出サマリー」を表示する
- **§4.7 直近レポート一覧 — 表示仕様**: 既存リストの先頭に
  > - セクション見出し「最近のレポート」を表示する
  > - テーブルに列見出し（タイトル / 対象期間 / 合計金額 / ステータス）を表示する
- **§11 品質チェック**: 既存項目を保持。新規チェック項目は追加しない（既存 DASH-001〜005 の範囲内の UI 改善であり要件追加ではないため）

---

## 実装方針

### 全体方針

issue 採用方針「案A」に従い、各テーブルコンポーネント自体に見出しを内包させる。`DashboardPage` 側の責務はロール分岐とセクション間余白付与のみで、見出し管理はコンポーネント側に閉じる。これによりコンポーネント単独で意味のある UI として再利用可能性も高まる。

### MonthlySummaryTable.tsx 改修パターン

```tsx
// 0 件時もセクション見出しを表示する。
return (
  <Box>
    <Typography variant="h6" sx={{ mb: 1 }}>
      月別支出サマリー
    </Typography>
    {items.length === 0 ? (
      <Typography variant="body2" color="text.secondary" sx={{ py: 2 }}>
        データがありません
      </Typography>
    ) : (
      <TableContainer component={Paper} variant="outlined">
        <Table size="small" aria-label="月別支出サマリー">
          {/* 既存 TableHead / TableBody そのまま */}
        </Table>
      </TableContainer>
    )}
  </Box>
);
```

### RecentReportList.tsx 改修パターン

```tsx
return (
  <Box>
    <Typography variant="h6" sx={{ mb: 1 }}>
      最近のレポート
    </Typography>
    {reports.length === 0 ? (
      <EmptyState message="..." />
    ) : (
      <TableContainer component={Paper} variant="outlined">
        <Table size="small" aria-label="最近の経費レポート">
          <TableHead>
            <TableRow>
              <TableCell>タイトル</TableCell>
              <TableCell>期間</TableCell>
              <TableCell align="right">金額</TableCell>
              <TableCell>ステータス</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {/* 既存 RecentReportRow ループそのまま */}
          </TableBody>
        </Table>
      </TableContainer>
    )}
    <Box sx={{ mt: 1, textAlign: 'right' }}>
      {/* 既存「すべてのレポートを見る」リンクそのまま */}
    </Box>
  </Box>
);
```

> 注: `RecentReportRow` は `<TableRow>` を返す既存実装のはずだが、worktree 内で実装内容を確認し、列定義が `<TableHead>` の 4 列と整合することを確認すること。`align="right"` の有無も `RecentReportRow` 側の `<TableCell>` と合わせる。

### DashboardPage.tsx 改修パターン

L166-174 を以下に置き換える:

```tsx
{/* Approver / Accounting / Admin: 月別サマリー */}
{isApproverOrAccountingOrAdmin && (
  <Box sx={{ mt: 3 }}>
    <MonthlySummaryTable items={monthlySummaryItems} />
  </Box>
)}

{/* Member / Approver / Accounting: 最近のレポート一覧 */}
{isMemberLike && (
  <Box sx={{ mt: 3 }}>
    <RecentReportList reports={recentReports} />
  </Box>
)}
```

`Box` import が未追加であれば `import Box from '@mui/material/Box';` を追加する。

---

## 作業手順

### Phase 1: 設計書改訂（dev-journal/master 直接編集）

指揮役が dev-journal の master ブランチで直接編集する（worktree 不要）。

1. `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` を編集
   - §2 コンポーネントツリー: `RecentReportList` 配下に `TableHead` 追記
   - §3 `MonthlySummaryTable` / `RecentReportList` の責務追記
   - §9 設計識別子の「テスト対象の概要」追記
2. `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` を編集
   - §4.6 表示仕様: セクション見出し追加の記載
   - §4.7 表示仕様: セクション見出し + 列見出し描画の記載
3. dev-journal master に直接コミット
   ```
   docs(dashboard): セクション見出し・列見出しの仕様を追補 (#157)
   ```
4. push

### Phase 2: FE 実装（expense-saas の worktree 隔離ブランチ）

frontend-developer エージェントに以下を依頼する。

1. worktree 内に入り、ブランチをリネーム: `git branch -m step11/157-dashboard-section-headings`
2. `MonthlySummaryTable.tsx` を改修（上記実装パターン）
3. `RecentReportList.tsx` を改修（上記実装パターン）。`RecentReportRow.tsx` の TableCell 構成と `<TableHead>` の列順・align 整合を確認
4. `DashboardPage.tsx` の L166-174 を `<Box sx={{ mt: 3 }}>` でラップ
5. テスト更新
   - `MonthlySummaryTable.test.tsx`: `screen.getByText('月別支出サマリー')` のアサーション追加。0 件時も見出しが見えること（DSH-FE-026 改修）
   - `RecentReportList.test.tsx`: `screen.getByText('最近のレポート')` 追加。列見出し 4 つの存在検証 (`screen.getByText('タイトル')` 等) 追加。`getAllByRole('row')` の件数アサーションが必要なら header 1 + data 5 = 6 行で再計算。0 件時も見出しが見えること
6. ローカルで `npm run lint --workspace=frontend` と該当テストファイルのみ実行（フルスイートは CI に任せる）
7. コミット
   ```
   fix(frontend): ダッシュボードのセクション見出し・列見出しを追加 (#157)
   ```
8. push: `git push -u origin step11/157-dashboard-section-headings`
9. PR 作成: `gh pr create --title "fix(frontend): #157 ダッシュボードのセクション見出し・列見出しを追加" --body <DoD チェックリスト + 変更内容>`
10. PR URL を指揮役に返す

### Phase 3: PR レビュー → マージ → SMK 再検証

- 指揮役が PR をセルフレビュー
- マージ後、issue を `dev-journal/issues/open/` から `pending-review/` に移動（再検証待ち）
- Step 11-A の SMK-011 副次として動作確認 → resolved に移動

---

## テスト方針

### 追加・更新するテスト

| ID | 対象 | 内容 | 種別 |
|----|------|------|------|
| DSH-FE-022（更新） | `MonthlySummaryTable.test.tsx` | 既存「3 件で 4 行」アサーションは TableHead 件数変わらないため維持。新たに `screen.getByText('月別支出サマリー')` を追加 | 既存改修 |
| DSH-FE-026（更新） | `MonthlySummaryTable.test.tsx` | 0 件時も `screen.getByText('月別支出サマリー')` が見えることを確認。「データがありません」表示は維持 | 既存改修 |
| DSH-FE-NEW-1 | `MonthlySummaryTable.test.tsx` | セクション見出しが `<h6>` レベルでレンダリングされること（`getByRole('heading', { level: 6, name: '月別支出サマリー' })`） | 新規 |
| DSH-FE-027（更新） | `RecentReportList.test.tsx` | 既存「5 件タイトル表示」は維持。`getAllByRole('row')` を使う場合は header 1 + data 5 = 6 で再計算 | 既存改修 |
| DSH-FE-028（更新） | `RecentReportList.test.tsx` | 0 件時も `screen.getByText('最近のレポート')` が見えること。`EmptyState` 表示は維持 | 既存改修 |
| DSH-FE-NEW-2 | `RecentReportList.test.tsx` | `screen.getByText('タイトル')`, `期間`, `金額`, `ステータス` の 4 列見出しが描画されること | 新規 |
| DSH-FE-NEW-3 | `RecentReportList.test.tsx` | セクション見出しが `<h6>` レベルでレンダリングされること（`getByRole('heading', { level: 6, name: '最近のレポート' })`） | 新規 |

### 確認コマンド

```bash
# 該当テストファイルのみ実行（worktree 内）
npm run test --workspace=frontend -- src/components/dashboard/__tests__/MonthlySummaryTable.test.tsx
npm run test --workspace=frontend -- src/components/dashboard/__tests__/RecentReportList.test.tsx

# Lint（worktree 内）
npm run lint --workspace=frontend
```

> フルテストスイートは CI（GitHub Actions）で実行する。サブエージェントにローカル全件実行はさせない（feedback_no_local_test_run）。

### `DashboardPage.tsx` のテスト

- DashboardPage 専用のテストは現状不在のため新規追加は不要（issue 範囲外）
- `<Box sx={{ mt: 3 }}>` のラップ確認は既存 E2E (Playwright) の存在に委ねる。本チケットでは UNIT テスト範囲のみで十分

---

## 依存・並列性

- **依存なし**: Step 10 / 11-A SMK 既完了。本 issue は SMK-011 副次発見の改修
- **issue #158 と並列実行可能**: 修正対象ファイルが重複しない場合（要確認）
  - #157 修正対象: `MonthlySummaryTable.tsx`, `RecentReportList.tsx`, `DashboardPage.tsx`, 上記 2 つの `__tests__/`, `dashboard.md` 2 本
  - #158 のスコープが上記と重ならないことを着手時に確認する
- **後続作業**: マージ後、`11-A-local-verification.md` の発行 issue テーブルを更新し、SMK-011 結果テーブルの備考を「副次 issue #157 → resolved」に更新する

---

## リスク・注意点

1. **`RecentReportRow` の列構造との整合**
   - 既存 `RecentReportRow.tsx` が返す `<TableCell>` の **数・順・align** が `<TableHead>` の列定義と完全一致していないと、列ズレが発生する
   - 実装時に `RecentReportRow.tsx` を必ず読み、列順（タイトル / 期間 / 金額 / ステータス）と `align="right"` の対応を確認する
2. **既存テストの行数アサーション**
   - `MonthlySummaryTable.test.tsx` DSH-FE-022 は既に「ヘッダー行 + データ 3 行 = 4 行」を前提としているので変更不要
   - `RecentReportList.test.tsx` で `getAllByRole('row')` を使うアサーションがあれば、TableHead 追加で +1 行になる点に注意
3. **0 件時の見出し表示判断**
   - 案 A 採用（0 件時も見出しを表示）。設計書改訂内容と実装が同じ判断であることを確認する
4. **`Typography variant="h6"` の見出しレベル**
   - 「ダッシュボード」ページタイトル（h1）に対して h6 は階層が深すぎないか（h2/h3 が適切ではないか）の懸念があるが、issue 提案文言を尊重し h6 を採用する。本格的な見出し階層整理は post-MVP とする
5. **dev-journal master 直接コミット**
   - 設計書改訂は dev-journal の master 直接コミット（PR 経由ではない）。指揮役が責任を持つ
6. **MVP スコープ厳守**
   - 「セクション見出し追加 + 列見出し追加 + 余白調整」のみ。グラフ化・カード化・カラースキーム見直しは含めない

---

## 後続への引き継ぎ

- マージ後、本 issue を `dev-journal/issues/open/157-...md` → `pending-review/` に移動し、SMK-011 副次として再検証
- 再検証 PASS 後、`resolved/` に移動
- `11-A-local-verification.md` の発行 issue テーブルを更新（状態を resolved に）
- 本チケット（11-A-issue-157.md）を完了マーク → progress.md の課題セクションから #157 を「対応完了」に更新

# issue #164: Admin ダッシュボードを PC 画面サイズで開くとカードの幅が他ロールより短い

- 担当: frontend-developer（FE 実装） + 指揮役（設計書改訂を dev-journal/master 直接編集）
- 依存: なし（G1 #154+#160 / G2 #162 と並列実行可能。修正対象ファイル独立）
- 順序注意: **#157 PR #110 のマージ後同期が必要**（DashboardPage.tsx を共有）。詳細は「リスク・注意点」§1
- ブランチ: `step11/164-admin-dashboard-card-width`（expense-saas）
- 出力先:
  - 設計書改訂: `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md`, `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md`（dev-journal/master 直接編集）
  - FE 実装: `expense-saas/frontend/src/components/dashboard/TenantStatusCards.tsx`, `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx`（worktree 隔離ブランチ）
  - テスト追加・更新: `expense-saas/frontend/src/components/dashboard/__tests__/TenantStatusCards.test.tsx`（既存があれば改修、なければ新規）, `expense-saas/frontend/src/pages/dashboard/__tests__/DashboardPage.test.tsx`
- 元資料: `dev-journal/issues/open/164-admin-dashboard-card-width-narrow-pc.md`
- テンプレート: `ai-dev-framework/templates/ticket-template.md`

## 入力

| 資料 | パス | 参照箇所 | 用途 |
|------|------|----------|------|
| issue 本文 | `dev-journal/issues/open/164-admin-dashboard-card-width-narrow-pc.md` | 全文（特に「推定原因」「推奨方針」「完了条件」） | 採用方針・受け入れ条件の確定 |
| UI コンポーネント設計 | `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` | §3 `TenantStatusCards` 責務、§9 設計識別子 | Admin 表示時 Grid 配置の追補対象を確定 |
| 画面詳細設計 | `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` | §4.4（ステータス別件数カード）, §4.5（メンバー数カード）, §9.4（Admin 構成サマリ） | Admin 表示時の Grid 配置仕様を追補 |
| 既存実装（FE） | `expense-saas/frontend/src/components/dashboard/TenantStatusCards.tsx` | L33-75（Grid `size={{ xs: 12, sm: 4, md: 'auto' }}` 全 5 枚） | 真因箇所。`md: 'auto'` を Grid 等分値（`md: 2.4` または定数）に変更 |
| 既存実装（FE） | `expense-saas/frontend/src/components/dashboard/MyReportCountCards.tsx` | L26-37（`xs: 12, sm: 4`） | 比較対象。他ロールの均等幅基準 |
| 既存実装（FE） | `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx` | L146-162（Admin 分岐: `TenantStatusCards` + 直下「メンバー数」`CountCard`） | Admin 分岐レイアウト。メンバー数カードを Grid でラップする責務 |
| 既存実装（FE） | `expense-saas/frontend/src/components/dashboard/CountCard.tsx` | 全文 | カード共通基盤に幅指定を入れない方針の確認（変更しない） |
| 既存テスト | `expense-saas/frontend/src/pages/dashboard/__tests__/DashboardPage.test.tsx` | 全文（Admin 分岐ケース） | Admin 表示時の DOM 構造アサーション追加箇所 |
| 関連 PR 履歴 | `git show 6916ed0` | コミット差分 | #153 修正（Badge 削除）が幅問題の原因かを確認した結果、**無関係**（CountCard sx の borderTop のみ変更で width 影響なし）。本チケット根本原因は別 |
| 共通実装ルール | `.claude/rules/implementation-workflow.md` | worktree 作業ルール / コメント言語 / FE エラーハンドリング | worktree 隔離での作業手順遵守 |

## 責務

### 含めること

- `TenantStatusCards.tsx` の各 Grid `size` を `{ xs: 12, sm: 6, md: 2.4 }` に変更し、PC 幅（md 以上）で 5 枚を等分配置する（`12 / 5 = 2.4`）
- `DashboardPage.tsx` Admin 分岐の「メンバー数」`CountCard` を `<Grid container>` + `<Grid size={{ xs: 12, sm: 4 }}>` でラップし、他ロールカード（`MyReportCountCards`）と同じ視覚幅基準を適用する
- 上記レイアウト変更を `<Box sx={{ mt: 2 }}>` の余白付与とセットでまとめ、TenantStatusCards とメンバー数カード間の視覚的分離を維持する
- 設計書 2 本に Admin 表示時の Grid 配置仕様（ブレイクポイント `xs/sm/md`、5 等分の根拠 `12/5=2.4`、メンバー数カード単独行配置）を明文化する
- DashboardPage.test.tsx に Admin 表示時のカード幅検証ケースを追加（DOM 構造で MUI Grid item のクラスまたは `data-testid` 経由で確認）
- 他ロール（Member / Approver / Accounting）に副作用がないことを既存テストで確認する

### 含めないこと

- `CountCard.tsx` 共通基盤への幅指定追加（責務分離維持: 幅はコンテナ Grid で制御する）
- `MyReportCountCards.tsx` / Approver `CountCard` / Accounting `CountCard` の Grid 値変更（影響範囲を Admin 分岐に閉じる）
- ダッシュボード UI の大幅再設計（カラースキーム・カード化追加・グラフ追加）
- カード**枚数**自体の変更（5 枚 + メンバー数の構成は維持）
- 上記以外の画面・他 issue（#154/#160/#162 等）への波及修正
- E2E テスト追加（DOM 構造変化はユニットテストで完結する）
- レスポンシブブレイクポイント全体の見直し（`sm` の値が 4 か 6 かの最終決定は実装時に DevTools で目視確認 → 6 を採用予定だが視認結果次第で 4 も可）

## 完了条件

issue 本文「完了条件」4 項目を以下の DoD 6 項目に分解する。

- [ ] **DoD-1（issue 完了条件 1）**: Admin ダッシュボード PC 幅（md ≥ 900px）で TenantStatusCards 5 枚が他ロールカードと視覚的に同等の均等幅で表示される（DevTools の computed `width` で各カードの値が概ね均等、コンテナ幅から spacing を除いた値の 1/5 ± 数 px に収まる）
- [ ] **DoD-2（issue 完了条件 2）**: Admin の「メンバー数」CountCard が `<Grid size={{ xs: 12, sm: 4 }}>` でラップされ、他ロール `MyReportCountCards` の左端カードと同じ幅基準で配置される（PC 幅で 1/3 幅相当）
- [ ] **DoD-3**: ユニットテスト `DashboardPage.test.tsx` に Admin 表示時の Grid 配置検証ケース（新規 DSH-FE-NEW-A1: TenantStatusCards 5 枚が Grid item として描画されること、新規 DSH-FE-NEW-A2: メンバー数カードが Grid container 配下に配置されること）を追加し PASS する
- [ ] **DoD-4（issue 完了条件 3）**: 設計書 2 本に Admin 表示時の Grid 配置仕様（`xs:12, sm:6, md:2.4` for TenantStatusCards、`xs:12, sm:4` for メンバー数カード、5 等分の根拠 `12/5=2.4`）が明文化される
- [ ] **DoD-5（issue 完了条件 4）**: 既存テスト（DashboardPage / MyReportCountCards / TenantStatusCards / CountCard）が全て PASS し、他ロール表示に副作用がないことを確認する（`npm run test --workspace=frontend -- src/components/dashboard src/pages/dashboard` が緑）
- [ ] **DoD-6**: PR 本文に DoD 6 項目のチェックボックスを記載し、レビュー時に確認可能な状態にする

---

## 影響範囲

### コード（expense-saas）

- `frontend/src/components/dashboard/TenantStatusCards.tsx` — 各 Grid `size` を `{ xs: 12, sm: 6, md: 2.4 }` に変更（5 箇所）
- `frontend/src/pages/dashboard/DashboardPage.tsx` — Admin 分岐 L156-160（メンバー数 CountCard）を `<Grid container>` + `<Grid size>` でラップ。L146-162 全体の構造軽微更新
- `frontend/src/pages/dashboard/__tests__/DashboardPage.test.tsx` — Admin 表示時の DOM 構造アサーション追加（DSH-FE-NEW-A1, A2）
- `frontend/src/components/dashboard/__tests__/TenantStatusCards.test.tsx` — 既存があれば Grid size 検証を追加。なければ本チケットでは新規作成しない（DashboardPage.test.tsx で代替）

### 設計書（dev-journal/master 直接編集）

- `deliverables/docs/55_ui_component/screens/dashboard.md`
  - §3 `TenantStatusCards` の「責務」に **Grid 配置仕様（`xs:12, sm:6, md:2.4`、5 等分の根拠）** を追記
  - §3 `DashboardPage` または §3 末尾に **Admin 分岐のメンバー数カード Grid ラップ仕様（`xs:12, sm:4`）** を追記
  - §9 設計識別子テーブルの該当行（`§TenantStatusCards`, `§DashboardPage`）の「テスト対象の概要」に Admin 表示時 Grid 配置検証を追記
- `deliverables/docs/50_detail_design/screens/dashboard.md`
  - §4.4 「ステータス別件数カード（テナント全体）— Admin のみ」の「表示仕様」に **PC 幅で 5 等分配置（`md:2.4`）、タブレット幅で 2 列折返し（`sm:6`）、モバイルで縦積み（`xs:12`）** を明文化
  - §4.5 「メンバー数カード — Admin のみ」の「表示仕様」に **PC 幅で 1/3 幅（`sm:4`、他ロール MyReportCountCards と同基準）、`<Grid container>` 配下で配置** を明文化
  - §9.4 Admin 構成サマリのテキスト図注記に Grid 配置への参照（§4.4 / §4.5）を 1 行追加（テキスト図自体の精緻化は post-MVP）

### 関連 issue / PR

- 関連 issue: #153（Approver 向けカード幅統一済み、resolved）— 副作用ではないことを確認済み
- 関連 PR: #108（commit `6916ed0`、Badge 削除）— 本問題の原因ではない（差分は CountCard の sx と DashboardPage の `showBadge` props 削除のみ）
- 並列実行可能: G1 #154+#160（AppDataGrid）、G2 #162（ConfirmDialog）と修正対象ファイル独立
- **マージ順序注意**: #157 PR #110（DashboardPage 変更中）→ 本 #164（DashboardPage Admin 分岐変更）。詳細は「リスク・注意点」§1

---

## 設計書改訂方針

### `55_ui_component/screens/dashboard.md`

- **§3 TenantStatusCards**: 「責務」末尾に以下を追記
  > Grid 配置は `xs={12} sm={6} md={2.4}` を 5 枚に適用する。PC 幅（md ≥ 900px）では 5 等分（`12 / 5 = 2.4`）でコンテナいっぱいに横並び、タブレット幅（sm 600〜899px）では 2 列折返し、モバイル（xs < 600px）では縦積みとなる。`md: 'auto'` のような自然幅指定は使用しない（コンテンツ依存で幅が縮むため）。
- **§3 DashboardPage** または §3 末尾に追記
  > Admin 分岐の「メンバー数」`CountCard` は `<Grid container>` + `<Grid size={{ xs: 12, sm: 4 }}>` でラップする。これにより MyReportCountCards（他ロール）と同じ視覚的幅基準（PC 幅で 1/3 幅）を共有する。
- **§9 テスト追跡用 設計識別子**: `§TenantStatusCards` / `§DashboardPage` の「テスト対象の概要」に「Admin 表示時の Grid 配置検証（5 等分・メンバー数カードの Grid ラップ）」を追記

### `50_detail_design/screens/dashboard.md`

- **§4.4 ステータス別件数カード（テナント全体）— 表示仕様**: 既存リスト末尾に追記
  > - **Grid 配置**: PC 幅（md ≥ 900px）で 5 等分（`md:2.4`）、タブレット幅（sm 600〜899px）で 2 列折返し（`sm:6`）、モバイル（xs < 600px）で縦積み（`xs:12`）。
- **§4.5 メンバー数カード — 表示仕様**: 既存リスト末尾に追記
  > - **Grid 配置**: `<Grid container>` 配下で `xs:12, sm:4` を適用。PC 幅で他ロール MyReportCountCards の左端カードと同じ 1/3 幅基準とする。
- **§9.4 Admin**: テキスト図直前 or 直後に 1 行注記
  > Grid 配置の詳細は §4.4（5 等分）/ §4.5（1/3 幅）を参照。

---

## 実装方針

### 全体方針

issue 採用方針「推奨方針」に従い、**カード共通基盤（CountCard）には幅指定を一切入れず、コンテナ Grid 側で制御する責務分離を維持**する。本問題の真因は `TenantStatusCards` の `md: 'auto'` 指定で、これを 5 等分 `md: 2.4` に変更することで解決する。Admin の「メンバー数」CountCard は Grid ラップなしで block 要素として剥き出しになっていたため、Grid container でラップして他ロールと同じ幅基準に揃える。

### 真因の根拠

- `TenantStatusCards.tsx` L34-72 の Grid `size={{ xs: 12, sm: 4, md: 'auto' }}`: `md: 'auto'` は MUI Grid2 で「コンテンツ自然幅」を意味する。CountCard の中身（ラベル + 件数）はコンパクトなため、コンテナ幅から大きく余白が残り、視覚的に「カード幅が短い」印象になる。
- 比較対象 `MyReportCountCards.tsx` L27-35 は `size={{ xs: 12, sm: 4 }}`（md 指定なし）。MUI Grid2 は上位ブレイクポイント未指定の場合、下位ブレイクポイントの値を継承するため、PC 幅でも `sm:4` (= 4/12 = 33.3%) が適用され均等幅となる。
- 結果として **Admin だけ `md: 'auto'` で縮み、他ロールは均等幅** という見た目の差異が生じる。

### TenantStatusCards.tsx 改修パターン

```tsx
// 5 枚すべての Grid size を統一する。`12 / 5 = 2.4` で PC 幅 5 等分、タブレット 2 列折返し、モバイル縦積み。
return (
  <Grid container spacing={2} data-testid="tenant-status-cards">
    <Grid size={{ xs: 12, sm: 6, md: 2.4 }}>
      <CountCard label="下書き" count={draftCount} accentColor="default" href="/reports/all?status=draft" />
    </Grid>
    <Grid size={{ xs: 12, sm: 6, md: 2.4 }}>
      <CountCard label="提出済み" count={submittedCount} accentColor="info" href="/reports/all?status=submitted" />
    </Grid>
    <Grid size={{ xs: 12, sm: 6, md: 2.4 }}>
      <CountCard label="承認済み" count={approvedCount} accentColor="success" href="/reports/all?status=approved" />
    </Grid>
    <Grid size={{ xs: 12, sm: 6, md: 2.4 }}>
      <CountCard label="却下" count={rejectedCount} accentColor="error" href="/reports/all?status=rejected" />
    </Grid>
    <Grid size={{ xs: 12, sm: 6, md: 2.4 }}>
      <CountCard label="支払済み" count={paidCount} accentColor="secondary" href="/reports/all?status=paid" />
    </Grid>
  </Grid>
);
```

> 補足: `data-testid="tenant-status-cards"` を Grid container に追加し、テストで `within(...).getAllByRole('link')` 5 件を確認できるようにする。

### DashboardPage.tsx 改修パターン

L146-162 を以下に置き換える:

```tsx
{/* Admin: テナント全体ステータスカード + メンバー数 */}
{isAdmin && (
  <>
    <TenantStatusCards
      draftCount={dashboard.tenant_draft_count ?? 0}
      submittedCount={dashboard.tenant_submitted_count ?? 0}
      approvedCount={dashboard.tenant_approved_count ?? 0}
      rejectedCount={dashboard.tenant_rejected_count ?? 0}
      paidCount={dashboard.tenant_paid_count ?? 0}
    />
    <Box sx={{ mt: 2 }}>
      <Grid container spacing={2} data-testid="admin-member-count-cards">
        <Grid size={{ xs: 12, sm: 4 }}>
          <CountCard
            label="メンバー数"
            count={dashboard.tenant_member_count ?? 0}
            unit="人"
          />
        </Grid>
      </Grid>
    </Box>
  </>
)}
```

> `Box` import が未追加であれば `import Box from '@mui/material/Box';` を追加する（#157 のマージ後は追加済みになる想定）。

---

## 作業手順

### Phase 1: 設計書改訂（dev-journal/master 直接編集）

指揮役が dev-journal の master ブランチで直接編集する（worktree 不要）。

1. `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` を編集
   - §3 TenantStatusCards の責務追記
   - §3 DashboardPage の Admin メンバー数カード Grid ラップ仕様追記
   - §9 設計識別子の「テスト対象の概要」追記
2. `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` を編集
   - §4.4 表示仕様: Grid 配置追記
   - §4.5 表示仕様: Grid 配置追記
   - §9.4 注記 1 行追加
3. dev-journal master に直接コミット
   ```
   docs(dashboard): Admin 表示時の Grid 配置仕様を明文化 (#164)
   ```
4. push

### Phase 2: FE 実装（expense-saas の worktree 隔離ブランチ）

frontend-developer エージェントに以下を依頼する。

1. worktree 内に入り、ブランチをリネーム: `git branch -m step11/164-admin-dashboard-card-width`
2. **#157 PR #110 のマージ状況を確認**
   - マージ済みなら `git merge origin/master` または `git rebase origin/master` で同期
   - マージ未完了なら指揮役に判断を仰ぐ（先行マージ待ちが原則。詳細「リスク・注意点」§1）
3. `TenantStatusCards.tsx` を改修（上記実装パターン、5 箇所の Grid `size` 変更 + `data-testid` 追加）
4. `DashboardPage.tsx` Admin 分岐 L146-162 を改修（メンバー数 CountCard の Grid ラップ）
5. テスト追加・更新
   - `DashboardPage.test.tsx`: Admin ロールでレンダリング → `screen.getByTestId('tenant-status-cards')` 取得 → `within(...).getAllByRole('link')` 5 件検証（DSH-FE-NEW-A1）。`screen.getByTestId('admin-member-count-cards')` 取得 → 配下に「メンバー数」テキストが存在することを検証（DSH-FE-NEW-A2）
   - 他ロールテスト（Member/Approver/Accounting）が既存通り PASS することを確認
6. ローカルで lint と該当テスト実行（フルスイートは CI に任せる）
   ```
   npm run lint --workspace=frontend
   npm run test --workspace=frontend -- src/components/dashboard src/pages/dashboard
   ```
7. コミット
   ```
   fix(frontend): Admin ダッシュボードのカード幅を他ロールと統一 (#164)
   ```
8. push: `git push -u origin step11/164-admin-dashboard-card-width`
9. PR 作成: `gh pr create --title "fix(frontend): #164 Admin ダッシュボードのカード幅を他ロールと統一" --body <DoD チェックリスト + 変更内容 + before/after 説明>`
10. PR URL を指揮役に返す

### Phase 3: PR レビュー → マージ → SMK 再検証

- 指揮役が PR をセルフレビュー（DevTools で Admin / Member / Approver / Accounting の各ロールで PC 幅・タブレット幅・モバイル幅の computed `width` を比較）
- マージ後、issue を `dev-journal/issues/open/164-...md` → `pending-review/` に移動
- Step 11-A の SMK 再検証（Admin ダッシュボード PC 幅）→ resolved に移動

---

## テスト方針

### 追加・更新するテスト

| ID | 対象 | 内容 | 種別 |
|----|------|------|------|
| DSH-FE-NEW-A1 | `DashboardPage.test.tsx` | Admin ロールで TenantStatusCards 5 枚（5 つの link）が `data-testid="tenant-status-cards"` 配下に描画されること | 新規 |
| DSH-FE-NEW-A2 | `DashboardPage.test.tsx` | Admin ロールでメンバー数カードが `data-testid="admin-member-count-cards"` 配下に Grid container でラップされて描画されること（「メンバー数」テキスト + 「人」単位の存在検証） | 新規 |
| 既存 DSH-FE-* | `DashboardPage.test.tsx` | Member / Approver / Accounting 表示の既存ケースが影響なく PASS すること | 回帰確認 |
| 既存 DSH-FE-* | `MyReportCountCards.test.tsx`（存在する場合） | 既存テストが影響なく PASS すること | 回帰確認 |

> Grid item の computed `width` 値そのものを jsdom で検証することは不可（jsdom は CSS レイアウト計算をしない）。**DOM 構造の存在検証 + クラス名・props 検証**にとどめ、視覚的な幅検証は指揮役の DevTools 目視レビューに委ねる方針とする。

### 確認コマンド

```bash
# 該当テスト実行（worktree 内）
npm run test --workspace=frontend -- src/pages/dashboard/__tests__/DashboardPage.test.tsx
npm run test --workspace=frontend -- src/components/dashboard

# Lint（worktree 内）
npm run lint --workspace=frontend
```

> フルテストスイートは CI（GitHub Actions）で実行する（feedback_no_local_test_run）。

### 視覚検証（指揮役）

- マージ前のセルフレビューで Admin / Member / Approver / Accounting の 4 ロールを Storybook または preview 環境で開き、PC 幅（≥1200px）・タブレット幅（768px）・モバイル幅（375px）の 3 ブレイクポイントで computed `width` を DevTools で比較する
- Admin TenantStatusCards 5 枚の幅が概ね均等（コンテナ幅から spacing を除いた値の 1/5 ± 数 px）であることを確認
- Admin メンバー数カードが他ロール MyReportCountCards 左端カードと同じ幅であることを確認

---

## 依存・並列性

- **依存なし**: G1（#154+#160 AppDataGrid）と G2（#162 ConfirmDialog）と修正対象ファイル独立で並列実行可能
- **マージ順序**: #157 PR #110（DashboardPage 変更中）→ 本 #164。詳細「リスク・注意点」§1
- **後続作業**: マージ後、`11-A-local-verification.md` の発行 issue テーブルを更新し、SMK 結果テーブルの備考を「副次 issue #164 → resolved」に更新する

---

## リスク・注意点

1. **#157 PR #110 とのマージ順序（最重要）**
   - PR #110 は `DashboardPage.tsx` L166-174（MonthlySummaryTable / RecentReportList の `<Box sx={{ mt: 3 }}>` ラップ）を変更中
   - 本 #164 は同ファイル L146-162（Admin 分岐）を変更する。**行的な競合は最小**だが、`Box` import の追加など隣接変更は競合候補
   - **対処**: 本チケット着手時に PR #110 のマージ状況を `gh pr view 110 --json state,mergedAt` で確認
     - マージ済み: worktree で `git merge origin/master` を実行してから着手
     - 未マージ: 指揮役に判断を仰ぐ。原則として #157 マージ完了を待つ（同ファイル隣接変更を直列化することで競合解消コストを削減）
   - PR push 直前にも `git pull --rebase origin master` を実行し、最新差分を取り込む
2. **`md: 2.4` の小数指定の MUI Grid2 サポート**
   - MUI Grid2（`@mui/material/Grid2`）は小数値（例: 2.4）を受け付ける（v5+）。実装前に worktree 内の `package.json` で MUI バージョンを確認し、対応していなければ `md: 2`（4 枚 + 1 枚改行）または `md: 2`（端 1 枚は隣接行先頭、視覚的に違和感あり）にフォールバックする
   - フォールバック判断は実装エージェントに委ねるが、選択肢のメリット・デメリットを PR 本文に記載すること
3. **#153（PR #108）副作用ではないことの再確認**
   - `git show 6916ed0` の差分は `CountCard.tsx`（Badge wrapper 削除）と `DashboardPage.tsx` の `showBadge` props 削除のみで、**幅指定の追加・削除は一切ない**
   - 本問題の真因は `TenantStatusCards.tsx` の `md: 'auto'` 指定（PR #108 以前から存在）。issue 推定 D（前回 #153 修正の副作用）は **false** と判定し、issue 推定 A + 別箇所（メンバー数カードの Grid ラップ欠如）が真因
4. **CountCard 共通基盤に幅指定を入れない責務分離**
   - 「カードコンポーネント自体に幅指定を入れる」という誘惑があるが、コンテナ Grid 側で制御する設計を厳守する
   - 各画面（ダッシュボード以外）で CountCard が再利用された場合に幅指定が干渉するため
5. **`sm:6` か `sm:4` か（タブレット幅）**
   - `sm:6` は 2 列折返し、`sm:4` は 3 列で残り 2 枚が次行（5 枚配置）。`sm:6` のほうが視覚的に整う（2-2-1 配置）
   - 本チケットは `sm:6` を推奨。実装時に DevTools で確認し、視認性が悪ければ `sm:4` に変更可
6. **既存テストの破壊的変更回避**
   - `MyReportCountCards.test.tsx` / `TenantStatusCards.test.tsx`（存在する場合）/ `CountCard.test.tsx` の既存アサーションが本変更で破壊されないことを確認
   - 特に `getAllByRole` の件数アサーションは Grid 構造変更で影響を受ける可能性
7. **MVP スコープ厳守**
   - 「TenantStatusCards Grid size 変更 + メンバー数カード Grid ラップ + 設計書追記」のみ。レスポンシブブレイクポイント全体見直し・カードデザイン見直しは含めない

---

## 後続への引き継ぎ

- マージ後、本 issue を `dev-journal/issues/open/164-...md` → `pending-review/` に移動し、Admin ダッシュボード PC 幅で SMK 再検証
- 再検証 PASS 後、`resolved/` に移動
- `11-A-local-verification.md` の発行 issue テーブルを更新（状態を resolved に）
- 本チケット（11-A-issue-164.md）を完了マーク → progress.md の課題セクションから #164 を「対応完了」に更新

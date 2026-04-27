# per_page UI セレクタ実装 + 4 画面適用 + URL/UI 整合 + テスト追加 + 設計書改訂

## 発見日
2026-04-25

## カテゴリ
implementation / ui-design / testing

## 影響度
中（per_page を画面操作で変更可能にする UX 改善 + 既存実装バグ修正 + テストカバレッジ補強の複合スコープ）

## 発見経緯
Step 11-A SMK-081（ページ送り）検証準備中、URL 直打ち `?per_page=1` が無視されて 8 件全表示される実装バグを発見。仕様協議の結果、per_page を「URL 経由のみ・UI なし」「UI 経由のみ・URL なし」のいずれでもなく、「**URL ⇔ UI 双方向反映 + 動的セレクタで乖離解消**」として確定し、4 ページ統一で実装する方針に決定。

## 関連ステップ
Step 5（API 詳細設計 / 画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10-B（レポート UI）/ Step 10-C（ワークフロー UI）/ Step 10-F（Admin 画面）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（SMK-081/082 は本 issue 修正後に Phase 2 末尾再検証として実施）

## 関連 issue

- **#143**（open, 2026-04-24）: 添付トースト不在 + リスト UI 構造不一致。本 issue とは独立だが UI 一貫性の文脈で類似
- **WFL-005 / WFL-046**: 既存のワークフロー系 per_page 統合テスト。本 issue では Reports 系に拡張
- **RPT-FE-021 / TNT-FE-037 / WFL-FE-027 / WFL-FE-052**: 既存の FE Hook per_page 単体テスト。本 issue では結合経路を補強
- **`dev-journal/deliverables/docs/50_detail_design/screens/report-list.md` L108, L180, L238**: per_page の API 仕様
- **#146**（post-MVP）: 大規模クロステナント性能テスト。独立だが per_page 機能が動作する前提

## 問題

### 1. 現状の実装バグ

`expense-saas/frontend/src/pages/reports/ReportListPage.tsx:38-41` は URL から `page` / `status` / `from` / `to` のみを `searchParams.get` しており、`per_page` の取得処理がない。L54-59 の `useMyReports` 呼び出しでも `per_page` 引数を渡していないため、API クエリに `per_page` が乗らず BE デフォルト 20 件が常に適用される。

設計書 `50_detail_design/screens/report-list.md:108, 180, 238, 241` は per_page を URL クエリパラメータとして明示している（デフォルト 20、最大 100、バリデーション 1-100）が、FE 側の URL 反映が未配線。

AdminAllReportsPage（`/reports/all`）/ PendingApprovalsPage（`/approvals`）/ PayableReportsPage（`/payments`）も同パターンの穴がある可能性が高い（各 Hook には per_page 引数があるが、Page 側の URL 読み取り未配線）。

### 2. UI コントロールの不在

per_page を変更する UI セレクタがどの画面にも存在しない。設計書も明示していない。結果として:

- ユーザーが per_page を変更する手段がない（URL 直編集のみ）
- 「複数件数を選びたい」UX 要求に応えられない
- per_page という URL クエリパラメータの存在意義が不明確

### 3. テストカバレッジの穴

| レイヤ | 検証内容 | 既存テスト | URL→表示の結合検証 |
|--------|---------|----------|------------------|
| BE / WFL 系 (`/api/workflow/pending`, `/payable`) | API per_page 動作 | WFL-005 / WFL-046 | 部分的（API レベルのみ） |
| BE / Reports 系 (`/api/reports`, `/reports/all`) | API per_page 動作 | **なし** | なし |
| FE Hook（`useMyReports`, `useAllReports`, `usePendingReports`, `usePayableReports`） | per_page 引数 → API URL | RPT-FE-021 / TNT-FE-037 / WFL-FE-027 / WFL-FE-052 | Hook 内のみ |
| FE Page（4 画面） | URL→表示 | **なし** | **なし** |
| E2E | URL→表示 + UI 操作 | **なし**（Step 11-C で予定） | **なし** |

## 採用方針（ユーザー承認済み）

### A. UI セレクタ追加

- 4 画面（`/reports`, `/reports/all`, `/approvals`, `/payments`）で **AppPaginationFooter** を採用し、ページ番号 + per_page セレクタを横並び表示
- 選択肢: **[10, 20, 50, 100]**（デフォルト 20）
- パターン X（動的セレクタ）: URL の per_page 値が標準外（例: 1, 73）であってもセレクタは URL 値を正直に表示。標準外値は動的に options に追加（例: URL=1 なら options=[1, 10, 20, 50, 100]）

### B. URL 反映

- URL ⇔ UI を双方向反映
  - URL 直打ち / リロード / ブックマーク → セレクタが URL 値を反映
  - セレクタ操作 → URL の `per_page` クエリ更新
- 範囲外（1 未満 / 100 超え）は BE バリデーションに委ねる（FE 側で事前クランプはしない）

### C. レイアウト

- フッター 1 行: 中央に AppPagination（ページ番号）、右に PageSizeSelector
- スマホ幅（375px）: `flex-direction: column` で縦並びにフォールバック
- AppPagination 既存実装は維持（薄い MUI Pagination ラッパーのまま）

## 提案

### 新規コンポーネント

#### PageSizeSelector

ファイル: `expense-saas/frontend/src/components/ui/PageSizeSelector.tsx`

```typescript
interface PageSizeSelectorProps {
  /** 現在の表示件数 */
  perPage: number;
  /** 標準選択肢（例: [10, 20, 50, 100]） */
  standardOptions?: number[];
  /** 表示件数変更時のコールバック */
  onPerPageChange: (size: number) => void;
  /** ローディング中などで無効化 */
  disabled?: boolean;
}
```

挙動:
- standardOptions のデフォルトは `[10, 20, 50, 100]`
- `perPage` が standardOptions に含まれない場合、options を `[...standardOptions, perPage].sort((a,b)=>a-b)` として動的拡張
- MUI `<Select>` を使用、ラベル「表示件数:」付き

#### AppPaginationFooter

ファイル: `expense-saas/frontend/src/components/ui/AppPaginationFooter.tsx`

```typescript
interface AppPaginationFooterProps {
  /** 現在のページ番号 */
  currentPage: number;
  /** 総ページ数 */
  totalPages: number;
  /** ページ変更時のコールバック */
  onPageChange: (page: number) => void;
  /** 現在の表示件数 */
  perPage: number;
  /** 表示件数変更時のコールバック */
  onPerPageChange: (size: number) => void;
  /** ローディング中などで無効化 */
  disabled?: boolean;
}
```

挙動:
- `<Box display="flex" justifyContent="space-between" alignItems="center" flexDirection={{ xs: 'column', sm: 'row' }}>` でレスポンシブ
- 内部で `<AppPagination>` + `<PageSizeSelector>` を配置
- フッターは常時表示（`totalPages <= 1` でも非表示にしない）。AppPagination は `count={Math.max(totalPages, 1)}` で常時表示し、PageSizeSelector も常時操作可能にする。**注: 末尾「2026-04-26 追加判断 §Q3」で確定方針として明示済み。本文上記の「totalPages <= 1 で AppPagination のみ非表示」は旧案として失効**

### 4 画面の修正

各画面で URL `searchParams.get('per_page')` を読み取り、Hook へ転送、`AppPaginationFooter` を直置きの `AppPagination` から切り替える。

対象ファイル（パスは対応時に確認）:

1. `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`
2. `expense-saas/frontend/src/pages/admin/AdminAllReportsPage.tsx`（または相当ファイル、対応者が確認）
3. `expense-saas/frontend/src/pages/workflow/PendingApprovalsPage.tsx`（または相当）
4. `expense-saas/frontend/src/pages/workflow/PayableReportsPage.tsx`（または相当）

各ページで以下のパターンを適用:

```typescript
// 例: ReportListPage.tsx
const perPageParam = searchParams.get('per_page');
const per_page = perPageParam ? parseInt(perPageParam, 10) : 20;

const { data, ... } = useMyReports({ page, per_page, status, from, to, ... });

const handlePerPageChange = (size: number) => {
  const next = new URLSearchParams(searchParams.toString());
  next.set('per_page', String(size));
  next.set('page', '1'); // per_page 変更時は 1 ページ目にリセット（フィルタ変更と同パターン）
  setSearchParams(next);
};

// JSX
<AppPaginationFooter
  currentPage={pagination?.current_page ?? 1}
  totalPages={pagination?.total_pages ?? 1}
  onPageChange={handlePageChange}
  perPage={pagination?.per_page ?? 20}
  onPerPageChange={handlePerPageChange}
/>
```

`per_page` 変更時に `page=1` にリセットするのは、既存の status フィルタ変更時の挙動（`ReportListPage.tsx:88-97`）と同パターンを踏襲。

### 設計書改訂

#### 1. UI コンポーネント設計

- `dev-journal/deliverables/docs/55_ui_component/common-components.md`
  - `AppPagination` セクションを「ページ番号のみ提供」と明記
  - 新規 `PageSizeSelector` セクションを追加（Props / 動作 / 動的選択肢挙動）
  - 新規 `AppPaginationFooter` セクションを追加（Props / レイアウト / レスポンシブ動作）

#### 2. 各画面の詳細設計

- `dev-journal/deliverables/docs/50_detail_design/screens/report-list.md`
  - §UI 構成 / §動作仕様に PageSizeSelector の説明を追加
  - フッターレイアウト（中央: ページ番号、右: PageSizeSelector）を明記
  - per_page 変更時に page=1 にリセットする動作を明記
- `dev-journal/deliverables/docs/50_detail_design/screens/admin-all-reports.md`: 同上
- `dev-journal/deliverables/docs/50_detail_design/screens/workflow-pending.md`: 同上
- `dev-journal/deliverables/docs/50_detail_design/screens/workflow-payable.md`: 同上

#### 3. 状態管理

- `dev-journal/deliverables/docs/55_ui_component/state-management.md`
  - useMyReports / useAllReports / usePendingReports / usePayableReports の URL 連動仕様を追記（per_page も含める）
  - URL クエリ ⇔ Hook ⇔ API URL の流れを明記
  - 動的選択肢の挙動を補足

#### 4. 基本設計（必要に応じて）

- `dev-journal/deliverables/docs/40_basic_design/screens.md`
  - ページネーション機能の節に per_page セレクタを追記（既存記述があれば更新）

### テスト追加

#### BE 統合テスト

ファイル: `expense-saas/internal/handler/report_handler_test.go`（または相当）

- **`TestListMyReports_Pagination`**: `userMember` のトークンで該当テナントに 2 件以上のレポートを seed し、`GET /api/reports?per_page=1` で 200 OK + `data` 件数 1 + `pagination.total_pages >= 2` を検証（WFL-005 同パターン）
- **`TestListAllReports_Pagination`**: `userAdmin` のトークンで `GET /api/reports/all?per_page=1` で同等検証

#### FE コンポーネント単体テスト

- **PageSizeSelector**: 標準選択肢のレンダリング / 動的選択肢追加の挙動 / onChange 呼び出し
- **AppPaginationFooter**: AppPagination + PageSizeSelector の合成 / レスポンシブレイアウト / disabled 連動

#### FE Page 結合テスト

- **ReportListPage**: MSW で `useMyReports` が複数件 + `pagination.total_pages > 1` を返すよう設定。URL を `/reports?per_page=10` で開いて 10 件のみ描画 / セレクタが「10」を表示 / セレクタで 50 を選んで URL に `?per_page=50&page=1` が反映 / セレクタが「50」に更新を検証
- **AdminAllReportsPage / PendingApprovalsPage / PayableReportsPage**: 同等パターン

#### テストケース設計書

- `dev-journal/deliverables/docs/60_test/test_cases/reports.md`
  - BE 統合テスト用 ID（例: `RPT-XXX`）
  - FE Page 結合テスト用 ID（例: `RPT-FE-XXX`、本セッションで予約済みの RPT-FE-107 / 108 を考慮し以降を採番）
- `dev-journal/deliverables/docs/60_test/test_cases/tenant.md`（または admin.md）: AdminAllReportsPage 用 ID
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md`: 既存の WFL-005/046 を参考に PendingApprovalsPage / PayableReportsPage 用 ID（例: `WFL-FE-XXX`）

##### 共通コンポーネント単体テスト

- `dev-journal/deliverables/docs/60_test/test_cases/components.md`（または該当ファイル）に PageSizeSelector / AppPaginationFooter の ID 採番

## 修正対象ファイル一覧

### 実装

- `expense-saas/frontend/src/components/ui/PageSizeSelector.tsx`（新規）
- `expense-saas/frontend/src/components/ui/AppPaginationFooter.tsx`（新規）
- `expense-saas/frontend/src/pages/reports/ReportListPage.tsx`
- `expense-saas/frontend/src/pages/admin/AdminAllReportsPage.tsx`（パス確認）
- `expense-saas/frontend/src/pages/workflow/PendingApprovalsPage.tsx`（パス確認）
- `expense-saas/frontend/src/pages/workflow/PayableReportsPage.tsx`（パス確認）

### テスト

- `expense-saas/frontend/src/components/ui/__tests__/PageSizeSelector.test.tsx`（新規）
- `expense-saas/frontend/src/components/ui/__tests__/AppPaginationFooter.test.tsx`（新規）
- `expense-saas/frontend/src/pages/reports/__tests__/ReportListPage.test.tsx`
- `expense-saas/frontend/src/pages/admin/__tests__/AdminAllReportsPage.test.tsx`（パス確認）
- `expense-saas/frontend/src/pages/workflow/__tests__/PendingApprovalsPage.test.tsx`（パス確認）
- `expense-saas/frontend/src/pages/workflow/__tests__/PayableReportsPage.test.tsx`（パス確認）
- `expense-saas/internal/handler/report_handler_test.go`

### 設計書

- `dev-journal/deliverables/docs/55_ui_component/common-components.md`
- `dev-journal/deliverables/docs/55_ui_component/state-management.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/report-list.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/admin-all-reports.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/workflow-pending.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/workflow-payable.md`
- `dev-journal/deliverables/docs/40_basic_design/screens.md`（必要に応じて）

### テスト設計書

- `dev-journal/deliverables/docs/60_test/test_cases/reports.md`
- `dev-journal/deliverables/docs/60_test/test_cases/tenant.md`（または admin.md）
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md`
- `dev-journal/deliverables/docs/60_test/test_cases/components.md`（あれば）

## 完了条件

- 4 画面（`/reports`, `/reports/all`, `/approvals`, `/payments`）で AppPaginationFooter が表示される
- per_page セレクタで [10, 20, 50, 100] が選択可能、デフォルト 20
- URL 直打ちの `?per_page=N` 値（1-100）が UI セレクタに正しく反映される（標準外値は動的選択肢として追加表示）
- セレクタ操作で URL `per_page` クエリと `page=1` リセットが両方反映される
- スマホ幅（375px）でフッターが縦並びにフォールバック
- BE 統合テストで `/api/reports` / `/api/reports/all` の per_page 動作が検証される
- FE Page 結合テストで URL → 表示件数の経路 + セレクタ操作 → URL 更新の経路が検証される
- 設計書（common-components.md / state-management.md / 4 つの screens / 必要に応じて screens.md）が新仕様を反映
- 各 test_cases に新規 ID が採番される
- PR チェックリストに「テスト設計書の更新」項目を含める（`.claude/rules/project-rules.md` または `implementation-workflow.md` 準拠）

## 2026-04-26 追加判断（着手前確定）

### 確定事項

#### Q1: 共通コンポーネント単体テスト ID の採番方針
- 既存 AppDatePicker (`ADT-XXX`) / AppSelect (`ASL-XXX`) パターンを踏襲
- 新接頭辞: `PSS` (PageSizeSelector) / `APF` (AppPaginationFooter) を新設
- 採番先: `60_test/test_cases/reports.md` に集約（AppDatePicker / AppSelect と同位置）

#### Q2: AllReportsPage の URL 駆動化を本 issue に含める
- 現状 `useState` ベース → `useSearchParams` への移行を本 issue で対応
- per_page 配線の前提として `page` も URL 駆動化が必要

#### Q3: フッター非表示仕様の撤廃
- 既存の `pagination &&` 等による「1 ページ未満時フッター非表示」仕様を撤廃
- AppPaginationFooter は常時表示
- 内部 AppPagination も `count={Math.max(totalPages, 1)}` で常時表示
- 4 画面で挙動を統一

#### Q4: per_page NaN / 不正値の FE フォールバック
- `parseInt` で NaN / 負数の場合は 20 にフォールバック（FE 側で fail-soft）
- 範囲内の不正値（0 や 101 等）は引き続き BE バリデーションに委ねる

### 実ファイルパスの確定（architect 調査結果）
- AdminAllReportsPage → `frontend/src/pages/admin/AllReportsPage.tsx`
- PendingApprovalsPage → `frontend/src/pages/workflow/ApprovalListPage.tsx`
- PayableReportsPage → `frontend/src/pages/workflow/PaymentListPage.tsx`
- AllReportsPage 専用テストファイル `__tests__/AllReportsPage.test.tsx` は **実体既存**（`expense-saas/frontend/src/pages/admin/__tests__/AllReportsPage.test.tsx`、19,695 byte）。本 issue では既存ファイルにテストケースを追記する
- Hook 側（useReports / useAllReports）は per_page 引数を既にサポート → **Hook 改修不要**

### 重要リスク（実装フェーズで意識）
1. 動的選択肢の重複ガード（`Set` による重複除去、key 重複 warning 回避）
2. NaN / 負数のハンドリング（Q4 で確定 = FE 側 20 フォールバック）
3. フッター常時表示への移行（Q3 で確定 = 既存条件付きレンダリング撤去）
4. 375px 縦並びテストは jsdom で完結しないため `sx` prop 検証で代替
5. `setSearchParams` の race（per_page 変更時は 1 回のコールに集約）
6. BE 統合テストのシード件数（既存 fixture との衝突確認、対象テナントに 2 件以上必要）
7. common-components.md の使用マトリクス・チェックリスト更新漏れ

## 解決日
2026-04-27

## 解決方針

確定方針 Q1〜Q4（issue 末尾「2026-04-26 追加判断」）に従い、以下を実施:

- **β2 分離運用**（テスト/実装の独立性確保）でテスト先行 PR (#101) と実装 PR (#100) を別ブランチで並列作成し、PR1 を PR2 に統合してマージ
- 設計成果物（`dev-journal/`）3 コミットで先行確定 → expense-saas で実装 + テスト

## 修正内容

### dev-journal（設計成果物、3 コミット）
- `e4afdcc`: `common-components.md` (PageSizeSelector / AppPaginationFooter 新規) / `state-management.md` §3.1 / 4 画面詳細設計（50 + 55）/ `40_basic_design/screens.md` §4.9 / test_cases（PSS / APF / RPT-091 / TNT-012 / RPT-FE-111〜114 / TNT-FE-048〜051 / WFL-FE-083〜090 採番）/ `items.md` L364 訂正
- `46f74d5`: PageSizeSelector / AppPaginationFooter に `data-testid` 規約を追記（review warning W-2 対応）
- `bbfbf73`: PageSizeSelector の表示文字列フォーマット `{size} 件` を正本化（β2 で発覚した実装/テスト乖離対応）

### expense-saas（実装 + テスト、squash merge: #100 = `89875a5`）

**新規実装（2 件）**:
- `frontend/src/components/ui/PageSizeSelector.tsx`: MUI Select ベース、表示形式「{size} 件」、動的選択肢（Set 重複除去）、`disabled` 対応
- `frontend/src/components/ui/AppPaginationFooter.tsx`: AppPagination + PageSizeSelector の合成、`xs:column` / `sm:row` レスポンシブ、`sm` 以上で左スペーサー配置

**既存改修（5 件）**:
- `AppPagination.tsx`: `totalPages <= 1 → return null` 削除（Q3 フッター常時表示）
- `ReportListPage.tsx`: per_page URL 駆動化 + AppPaginationFooter 切り替え
- `AllReportsPage.tsx`: `useState` → `useSearchParams` 移行（page / filters / per_page 全て URL 駆動、Q2）+ AppPaginationFooter 切り替え
- `ApprovalListPage.tsx` / `PaymentListPage.tsx`: per_page 配線 + AppPaginationFooter 切り替え

**テスト追加（7 件）**:
- 共通: `PageSizeSelector.test.tsx` (PSS-001〜005) / `AppPaginationFooter.test.tsx` (APF-001〜007)
- Page 結合: ReportListPage / AllReportsPage / ApprovalListPage / PaymentListPage に各 4 件追加
- BE 統合: `report_handler_test.go` に RPT-091 / TNT-012 追加

### root-project
- `a1d5c1d`: `.claude/agents/test-implementer.md` の必須参照を `test_cases.md` (単一) → `test_cases/` (ディレクトリ) に訂正（運用上の記述ズレ修正）

## β2 分離運用の学び

- テスト先行 PR (#101) と実装 PR (#100) を別ブランチで並列作成し、最終的に PR2 に統合する運用を実施
- 「実装に合わせるリスク」を抑える効果は得られたが、設計書未明文化の細部（表示文字列「件」の有無）で乖離が発生 → `bbfbf73` で設計書追記して正本化
- 教訓: β2 分離する場合、設計書には **表示文字列レベルの仕様**まで明記する必要がある

## レビュー

- 内部 reviewer (PR #100): PASS（warning 1 / nit 3、対応または受容済み）
- 内部 reviewer (PR #101): PASS（warning 3 / nit 3、対応または受容済み）
- codex review (PR #100): 初回 FIX（blocker 1: PSS strict TS / warning 1: ReportListPage `?? page` 不一致）→ 修正 `b8323b8` → 再レビュー **PASS**
- codex review (設計成果物): 初回 FIX（blocker 2 / warning 1）→ 修正 → 再レビュー **PASS**（resolved/109〜111 として記録）
- ローカル CI: lint 0 errors / tsc 0 errors / vitest 702/702 PASS

## 残対応（マージ後）

- BE テスト実行: ユーザー作業（VS Code タスク `BE: full test`）でマージ後に確認

## 2026-04-27 再オープン

### 再オープン理由

Step 11-A SMK-081（ページ送り）検証で、AppPaginationFooter が「テーブルフッター位置への統合」になっておらず、テーブル要素の外側に独立した `<Box>` として配置されていることを発見。設計書 `common-components.md` L353「一覧テーブルの直下に配置する」が **テーブル内フッター位置への統合** か **テーブル外の隣接配置** かを明示していなかったため、実装が後者で確定してしまっていた。本来の意図は前者（テーブル内フッター位置への統合）。

ユーザー判断: 設計書 + 実装の両方を修正する（コンポーネント化しているため 4 画面一括対応で完結する）。

### 確定方針（2026-04-27 ユーザー協議で確定）

技術調査の結果、本プロジェクトの 4 対象画面はいずれも MUI X `DataGrid`（`AppDataGrid` 経由）を使用しており、HTML 上は `<table>` ではなく `<div role="grid">` 構造のため、ネイティブ MUI `<TableFooter>` への統合は不可。代わりに以下を確定方針とする。

- **採用案 D-1**: `AppDataGrid` の `slots.footer` プロパティに `AppPaginationFooter` を差し込み、DataGrid フッターコンテナ（`<div class="MuiDataGrid-footerContainer">`）位置に統合する
- **実装パターン ②a**: `AppDataGrid` 自体は拡張せず、DataGrid 標準の `slots` props を活用する
  - **直接利用パターン**（`ApprovalListPage` / `PaymentListPage`）: 画面側で `<AppDataGrid slots={{ footer: () => <AppPaginationFooter ... /> }} />` の形で直接 `slots.footer` に渡す
  - **中間ラッパー経由パターン**（既存中間ラッパー `pages/reports/ReportListTable.tsx` / `pages/admin/AllReportsTable.tsx` を持つ画面）: 中間ラッパーに `paginationFooter?: ReactNode` 等の prop を追加し、ラッパー内部で `slots.footer` に変換して `AppDataGrid` に渡す

`hideFooterPagination={true}` の併用を推奨（DataGrid 標準ページネーション UI との二重表示を避けるため。実装フェーズで最終動作確認）。

### 追加対応内容

#### 1. 設計書改訂

- `dev-journal/deliverables/docs/55_ui_component/common-components.md`
  - §AppDataGrid に「カスタムフッター」項を追加し、`slots.footer` 経由で `AppPaginationFooter` を統合するパターンを明示
  - §AppPaginationFooter §配置・利用前提を「`AppDataGrid` の `slots.footer` プロパティに渡し、DataGrid フッターコンテナに統合する」と全面改訂
  - 直接利用パターン / 中間ラッパー経由パターンの 2 系統併記
  - `hideFooterPagination={true}` 併用推奨を明記
  - DataGrid フッター内では外側マージン `mt={2}` 不要を明記
- 4 画面の詳細設計（`50_detail_design/screens/{report-list, admin-all-reports, workflow-pending, workflow-payable}.md`）で、フッター配置の文言を `slots.footer` 経由統合・パターン ②a に整合させる
- `state-management.md`（§3.1 URL ⇔ Hook ⇔ API URL 連動仕様）は配置パターンに依存しないため変更不要
- `40_basic_design/screens.md` §4.9 は基本設計粒度では既存記述で十分のため変更不要（詳細設計に委譲）

#### 2. 実装修正（4 画面一括）

- `frontend/src/pages/reports/ReportListPage.tsx` + `pages/reports/ReportListTable.tsx`（中間ラッパー、`paginationFooter` prop 追加）
- `frontend/src/pages/admin/AllReportsPage.tsx` + `pages/admin/AllReportsTable.tsx`（中間ラッパー、`paginationFooter` prop 追加）
- `frontend/src/pages/workflow/ApprovalListPage.tsx`（直接 `slots.footer` 渡し）
- `frontend/src/pages/workflow/PaymentListPage.tsx`（同）

各画面で `AppPaginationFooter` を `AppDataGrid` の `slots.footer` 経由で配置する。中間ラッパー画面 2 件は `paginationFooter` prop を追加し、ラッパー内部で `slots.footer` に変換する。直接利用画面 2 件は画面側で `slots.footer` に直接渡す。

#### 3. テスト修正

- `AppPaginationFooter` コンポーネント単体テスト（APF-001〜007）: `data-testid="app-pagination-footer"` は維持されるため testid 経路は影響限定。jsdom 上の動作確認のみで完結する範囲は変更不要
- 4 画面の Page 結合テスト: `AppPaginationFooter` の取得経路（`within(...)` 等）が DataGrid フッターコンテナ内部の DOM 階層変更で影響を受ける可能性 → 動作確認の上、必要なら追従修正
- 中間ラッパー単体テスト（`ReportListTable.test.tsx` / `AllReportsTable.test.tsx`）: `paginationFooter` prop の受け渡し検証を追加

### 完了条件（追加）

- 4 画面すべてで `AppPaginationFooter` が DataGrid フッターコンテナ（`<div class="MuiDataGrid-footerContainer">`）内に配置される（DevTools で確認）
- 設計書 `common-components.md` §AppDataGrid / §AppPaginationFooter が `slots.footer` 経由統合を明記
- 4 画面の詳細設計が画面ごとに直接利用 / 中間ラッパー経由のパターン区別を明記
- 375px 縦並びレスポンシブが DataGrid フッターコンテナ内でも維持される
- `AppDataGrid` の他利用箇所（4 画面以外）への副作用なし
- 既存テスト + 追加テストがローカル CI で PASS
- SMK-081 を再実施し PASS であること

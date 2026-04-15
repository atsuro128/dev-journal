# Workflow 2 ページ（Approval/Payment）に同期ロールチェックがなく、403 リダイレクトが遅延 + コンテンツがフラッシュ表示される

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / ux / consistency

## 影響度
低〜中（機能はするが、Member が直接 URL アクセスした際に「権限のないページの中身が一瞬見える」UX 劣化。情報漏洩リスクは画面ヘッダー程度で限定的）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-007 / SMK-025 を実施中（issue 105 で SMK のパス修正後）、Member ロールで 4 つの権限外ページに直接 URL アクセスして比較した:

| ページ | リダイレクト挙動 |
|---|---|
| `/settings/tenant`（TenantPage） | 即リダイレクト、フラッシュなし |
| `/reports/all`（AllReportsPage） | 即リダイレクト、フラッシュなし |
| `/approvals`（ApprovalListPage） | 「～待ち一覧」のヘッダー文言が一瞬見えてからリダイレクト |
| `/payments`（PaymentListPage） | 同上 |

Workflow 系 2 ページだけ挙動が劣化していることが判明。

## 関連ステップ
Step 10-F（ワークフロー機能実装）/ PR #53（issue 088 対応）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 事実

### 良い実装パターン: TenantPage

`expense-saas/frontend/src/pages/admin/TenantPage.tsx` は **同期的ロールチェック + 403 API エラーチェック**の二重防御:

```tsx
const currentUser = userData?.data;

// 1. 同期的ロールチェック（API 呼び出し前にチェック）
useEffect(() => {
  if (currentUser && currentUser.role !== 'admin') {
    navigate('/dashboard', {
      state: { toast: { severity: 'error', message: '権限がありません' } },
      replace: true,
    });
  }
}, [currentUser, navigate]);

// 2. API 403 エラーチェック（バックエンド側でのフェイルセーフ）
useEffect(() => {
  if (error instanceof ApiClientError && error.status === 403) {
    navigate('/dashboard', {
      state: { toast: { severity: 'error', message: '...' } },
      replace: true,
    });
  }
}, [error, navigate]);
```

`currentUser` は auth store からほぼ即座に取得でき、ロール不一致がわかった瞬間にリダイレクトされる。API 呼び出しの結果を待たないため画面に何も描画されない。

### 悪い実装パターン: ApprovalListPage / PaymentListPage

`expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx` L134-147（PaymentListPage も同型）:

```tsx
// 403 エラー時はダッシュボードにリダイレクトし、トーストで理由を通知する。
useEffect(() => {
  if (isError && error && (error as { status?: number }).status === 403) {
    void navigate('/dashboard', {
      state: { toast: { severity: 'error', message: '...' } },
      replace: true,
    });
  }
}, [isError, error, navigate]);
```

**API エラー（isError）でしかリダイレクトをトリガーしていない**。同期的なロールチェックがない。

### 結果として何が起きるか

1. ユーザーが `/approvals` に直接遷移
2. ApprovalListPage がマウント → `<Typography>承認待ち一覧</Typography>` 等のヘッダーが描画される
3. データ取得用の useQuery が API を呼ぶ
4. API が 403 を返す（数百ミリ秒）
5. `isError` が true になり、useEffect でリダイレクト
6. ユーザーには「承認待ち一覧というヘッダーが一瞬見えてから、ダッシュボードに飛ばされる」UX

TenantPage では (2) の前に (1) の同期チェックが走るため、ヘッダーすら描画されない。

### PR #53 の経緯

前セッション（2026-04-14）の session-log:
> 「Workflow 2 ページの 403 useEffect に toast state 追加」

PR #53 で Workflow 2 ページに toast 通知は追加されたが、同期チェックパターンに揃える対応は行われなかった。Group C のスコープ的に追加で取り込まれた経緯があり、TenantPage と同じパターンに揃える時間がなかった可能性がある。

## 修正方針

ApprovalListPage / PaymentListPage に TenantPage と同じ同期ロールチェックを追加する。

```tsx
const { data: userData } = useCurrentUser();
const currentUser = userData?.data;

// 同期的ロールチェック（Member は承認待ち一覧へのアクセス不可）
useEffect(() => {
  if (currentUser && currentUser.role !== 'approver' && currentUser.role !== 'admin') {
    navigate('/dashboard', {
      state: { toast: { severity: 'error', message: 'この画面にアクセスする権限がありません' } },
      replace: true,
    });
  }
}, [currentUser, navigate]);
```

**注意点**:
- ApprovalListPage は Approver / Admin のみ可（design 確認）
- PaymentListPage は Accounting / Admin のみ可（design 確認）
- ロール定数は `screens/*.md` および `authz.md` を正本にして拾うこと
- 既存の API 403 エラー useEffect はフェイルセーフとして残す（同期チェックを抜けても弾けるように）

### より根本的な対策（任意）

PrivateRoute レベルでロール別のアクセス制御を実装する。例:

```tsx
<Route element={<PrivateRoute roles={['approver', 'admin']} />}>
  <Route path="/approvals" element={<ApprovalListPage />} />
</Route>
```

これは全画面の RBAC 表示制御を一元化できるため、issue 104（UI RBAC カバレッジ監査）と合わせて検討する価値あり。本 issue では「最小修正で挙動を揃える」ことを優先する。

## 修正対象ファイル

- `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx`
- `expense-saas/frontend/src/pages/workflow/PaymentListPage.tsx`
- 既存テスト: `__tests__/ApprovalListPage.test.tsx` / `PaymentListPage.test.tsx`（同期チェック経路のテストを追加）

## 完了条件

- ApprovalListPage / PaymentListPage に同期ロールチェックが追加されている
- Member で `/approvals` / `/payments` を直接開いても、ヘッダー文言がフラッシュ表示されない
- TenantPage / AllReportsPage と同じ即時リダイレクト挙動になる
- 既存テストが通過し、新規アサーション（同期チェック経路）が追加されている

## 関連
- 088 / PR #53: 認可エラー UX 改善（Group C 修正） — 本 issue は PR #53 で取りこぼされた追加対応
- 104: UI RBAC カバレッジ監査 — 本 issue の根本対策（PrivateRoute レベルの一元化）の親 issue
- 105: smoke_check.md SMK-007 / SMK-025 のパス誤記 — SMK 検証の前段に必要

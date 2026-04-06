# テナント情報 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/admin-tenant.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/admin-tenant.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-ADM-002` |
| 画面名 | テナント情報 |
| URL パス | `/settings/tenant` |
| 画面詳細仕様 | `50_detail_design/screens/admin-tenant.md` |

---

## 2. コンポーネントツリー

```
TenantPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── PageTitle
        ├── TenantInfoCard
        │   ├── TenantInfoField
        │   └── PageSkeleton（← common-components.md）
        └── PhaseNotice
```

---

## 3. コンポーネント定義

### TenantPage

- 配置: `pages/admin/TenantPage.tsx`
- 責務: テナント情報画面のページコンポーネント。AppLayout でラップし、useTenant Hook を呼び出してテナント情報を取得する。403 エラー時はダッシュボードにリダイレクト、404 エラー時はメインコンテンツ領域にエラーメッセージを表示、500 系エラーはトーストで通知する
- 対応セクション: `50_detail_design/screens/admin-tenant.md` &sect;1, &sect;3, &sect;7, &sect;8

```typescript
// Props なし（ページコンポーネント）
```

### PageTitle

- 配置: `components/ui/PageTitle.tsx`
- 責務: ページタイトルを Typography で表示する
- 対応セクション: `50_detail_design/screens/admin-tenant.md` &sect;5（ページタイトル）

Props 型定義は `common-components.md §PageTitle` を参照。

### TenantInfoCard

- 配置: `pages/admin/TenantInfoCard.tsx`
- 責務: テナント情報を Card コンポーネントで表示する。ローディング中は PageSkeleton を表示し、データ取得後は TenantInfoField で会社名を読み取り専用で表示する。404 エラー時はエラーメッセージを表示する
- 対応セクション: `50_detail_design/screens/admin-tenant.md` &sect;5（テナント情報カード）, &sect;6, &sect;7

```typescript
interface TenantInfoCardProps {
  /** テナント情報 */
  tenant: TenantInfo | undefined;
  /** ローディング状態 */
  loading: boolean;
  /** エラー状態（404 等） */
  error: ApiClientError | null;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `tenant` | `TenantInfo \| undefined` | Yes | テナント情報（id, name, created_at） | useTenant Hook の data.data |
| `loading` | `boolean` | Yes | ローディング状態 | useTenant Hook の isLoading |
| `error` | `ApiClientError \| null` | Yes | エラー情報（404 時のメッセージ表示に使用） | useTenant Hook の error |

### TenantInfoField

- 配置: `pages/admin/TenantInfoField.tsx`
- 責務: ラベルと値のペアを読み取り専用で表示する。MVP ではテナント情報は会社名のみだが、Phase 3 での項目追加に対応できる汎用設計とする
- 対応セクション: `50_detail_design/screens/admin-tenant.md` &sect;5（テナント情報カード）

```typescript
interface TenantInfoFieldProps {
  /** フィールドのラベル */
  label: string;
  /** フィールドの値 */
  value: string;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `label` | `string` | Yes | 項目ラベル（「会社名」） | 固定値 |
| `value` | `string` | Yes | 項目の値 | useTenant Hook の data.data.name |

### PhaseNotice

- 配置: `pages/admin/PhaseNotice.tsx`
- 責務: Phase 3 予告メッセージをグレーの補足文として表示する。MVP では編集機能がないことをユーザーに伝える
- 対応セクション: `50_detail_design/screens/admin-tenant.md` &sect;5（Phase 3 予告メッセージ）

```typescript
interface PhaseNoticeProps {
  /** 予告メッセージテキスト */
  message: string;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `message` | `string` | Yes | 表示メッセージ（「テナント情報の編集機能は今後追加予定です。」） | 固定値 |

---

## 4. データフロー

```
GET /api/tenant
  → useTenant()（← state-management.md §3 データフェッチ系）
    → TenantPage
      → TenantInfoCard (props: tenant, loading, error)
        → PageSkeleton (props: variant="card")  [loading 時]
        → TenantInfoField (props: label="会社名", value=tenant.name)  [データ取得後]
      → PhaseNotice (props: message)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useTenant` | テナント情報のデータフェッチ | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | ヘッダーのユーザー情報表示・ロール確認 | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| テナント情報 | サーバー | useTenant Hook（TanStack Query useQuery） | TenantPage |
| ローディング状態 | サーバー | useTenant Hook の isLoading | TenantPage |
| エラー状態 | サーバー | useTenant Hook の error | TenantPage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | TenantPage のルートラッパー | children に PageTitle + TenantInfoCard + PhaseNotice を配置 |
| `PageSkeleton`（← common-components.md） | TenantInfoCard 内のローディング表示 | variant="card"。初回読み込み時に表示 |
| `AppToast`（← common-components.md） | TenantPage（SnackbarContext 経由） | API 通信エラー（500 系）のトースト表示 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `TenantPage` | Admin ロールのみアクセス可能。他ロール（Accounting / Approver / Member）はダッシュボードにリダイレクト | - | `authz.md §6.7`（`/api/tenant`: Admin のみ）/ `screens/admin-tenant.md §3` |
| `AppSidebar`（「テナント情報」メニュー） | Admin ロールのみ表示 | - | `screens/admin-tenant.md §8` |
| `TenantInfoCard` | 常時表示（読み取り専用） | 編集不可（MVP スコープ。編集は Phase 3 予定） | `screens/admin-tenant.md §5` |
| `PhaseNotice` | 常時表示 | - | `screens/admin-tenant.md §5` |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | TenantPage | URL パス、対応 API、対応ロール |
| &sect;3 アクセス制御 | TenantPage | Admin のみ。他ロールはリダイレクト |
| &sect;4 レイアウト | AppLayout, PageTitle, TenantInfoCard, PhaseNotice | ヘッダー + サイドナビ + メインコンテンツの標準レイアウト |
| &sect;5 表示項目（ページタイトル） | PageTitle | 「テナント情報」 |
| &sect;5 表示項目（テナント情報カード） | TenantInfoCard, TenantInfoField | ラベル「会社名」+ 値の読み取り専用表示 |
| &sect;5 表示項目（Phase 3 予告メッセージ） | PhaseNotice | 「テナント情報の編集機能は今後追加予定です。」 |
| &sect;6 ローディング | PageSkeleton | variant="card"。初回読み込み時 |
| &sect;7 エラー表示 | TenantInfoCard（404）, AppToast（500 系） | 404: メインコンテンツ領域に「テナント情報が見つかりません。」、500 系: トースト、403: リダイレクト、401: ログイン画面リダイレクト |
| &sect;8 画面遷移 | AppSidebar | サイドナビ「テナント情報」メニュー（Admin のみ表示） |
| &sect;9 API リクエスト/レスポンス | useTenant | GET /api/tenant |
| &sect;10 処理シーケンス | TenantPage（Hook 呼び出し） | JWT 検証 + RBAC + RLS は API 側で処理 |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/admin-tenant.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/admin-tenant.md §TenantPage` | コンポーネント単体テスト | Admin でのアクセス可否、他ロール（Accounting / Approver / Member）のリダイレクト、API エラー時の処理（500 系トースト、403 リダイレクト、404 エラー表示） |
| `55_ui_component/screens/admin-tenant.md §TenantInfoCard` | コンポーネント単体テスト | ローディング時の PageSkeleton 表示、データ取得後の会社名表示、404 エラー時のエラーメッセージ表示 |
| `55_ui_component/screens/admin-tenant.md §TenantInfoField` | コンポーネント単体テスト | ラベルと値の描画 |
| `55_ui_component/screens/admin-tenant.md §PhaseNotice` | コンポーネント単体テスト | 予告メッセージの描画 |
| `55_ui_component/screens/admin-tenant.md §PageTitle` | コンポーネント単体テスト | タイトルテキストの描画 |
| `55_ui_component/state-management.md §useTenant` | Hook 単体テスト | API 呼び出し・レスポンスのデータ変換・キャッシュ（staleTime 5 分） |

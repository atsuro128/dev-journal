# レポート編集 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/report-edit.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/report-edit.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-RPT-003` |
| 画面名 | レポート編集 |
| URL パス | `/reports/:id/edit` |
| 画面詳細仕様 | `50_detail_design/screens/report-edit.md` |

---

## 2. コンポーネントツリー

```
ReportEditPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── AppBreadcrumbs（← common-components.md）
        ├── PageSkeleton（← common-components.md）[データ読み込み中]
        └── ReportForm（← common-components.md）
            ├── FormAlert（← common-components.md）
            ├── AppTextField（← common-components.md）[タイトル]
            ├── ReportPeriodField（← common-components.md）
            │   ├── AppDatePicker（← common-components.md）[開始日]
            │   └── AppDatePicker（← common-components.md）[終了日]
            └── ReportFormActions（← common-components.md）
                ├── SubmitButton [保存する]
                └── CancelButton
```

---

## 3. コンポーネント定義

### ReportEditPage

- 配置: `pages/reports/ReportEditPage.tsx`
- 責務: レポート編集画面のページコンポーネント。URL パスパラメータ `:id` からレポート ID を取得し、useReport Hook で既存データを読み込む。アクセス制御を実施し、レポートが存在しない場合は一覧にリダイレクト、所有者でない場合は 403 トースト表示、draft でない場合は詳細画面にリダイレクトする。useUpdateReport Hook でレポート更新のミューテーションを実行し、成功時に `/reports/:id` に遷移する。楽観的ロック用に取得時の `updated_at` を保持する
- 対応セクション: `50_detail_design/screens/report-edit.md` &sect;1, &sect;6, &sect;7, &sect;10

```typescript
// Props なし（ページコンポーネント）
```

### ReportForm（共通）

- 配置: `components/report/ReportForm.tsx`
- 責務: レポートのタイトル・対象期間を入力するフォーム。作成画面と編集画面で共有される。Props の `submitLabel` と `defaultValues` で作成 / 編集を切り替える
- 対応セクション: `50_detail_design/screens/report-edit.md` &sect;3, &sect;4, &sect;5

Props 型定義は `common-components.md §ReportForm` を参照。編集画面では以下の Props 値で使用する。

| Props | 編集画面での値 |
|-------|-------------|
| `onSubmit` | ReportEditPage が useUpdateReport を呼び出す関数 |
| `onCancel` | `/reports/:id` に戻る navigate 呼び出し |
| `apiError` | useUpdateReport の error から変換したメッセージ（409 Conflict 含む） |
| `isPending` | useUpdateReport の isPending |
| `submitLabel` | `"保存する"` |
| `defaultValues` | useReport で取得した既存レポートの title, periodStart, periodEnd |

### FormAlert（共通）

- 配置: `components/ui/FormAlert.tsx`
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。編集画面では 409 Conflict エラー（「他のユーザーがこのレポートを更新しました。ページを再読み込みしてください。」）も表示対象となる
- 対応セクション: `50_detail_design/screens/report-edit.md` &sect;5, &sect;7

Props 型定義は `common-components.md §FormAlert` を参照。

### ReportPeriodField（共通）

- 配置: `components/report/ReportPeriodField.tsx`
- 責務: 対象期間の開始日・終了日を横並びに配置する
- 対応セクション: `50_detail_design/screens/report-edit.md` &sect;3

Props 型定義は `common-components.md §ReportPeriodField` を参照。

### ReportFormActions（共通）

- 配置: `components/report/ReportFormActions.tsx`
- 責務: フォーム下部のアクションボタン群。編集画面では submitLabel が「保存する」、キャンセルは詳細画面に戻る
- 対応セクション: `50_detail_design/screens/report-edit.md` &sect;7

Props 型定義は `common-components.md §ReportFormActions` を参照。

---

## 4. データフロー

```
GET /api/reports/:id（既存データ取得）
  → useReport(reportId)（← state-management.md §3 データフェッチ系）
    → ReportEditPage
      → [データ読み込み中] PageSkeleton (variant: 'form')
      → [データ読み込み完了]
        → AppBreadcrumbs (props: items = [マイレポート, レポートタイトル, 編集])
        → ReportForm (props: onSubmit, onCancel, apiError, isPending, submitLabel="保存する", defaultValues)

PUT /api/reports/:id（更新）
  → useUpdateReport()（← state-management.md §3 ミューテーション系）
    → ReportEditPage
      → ReportForm (props: onSubmit に updated_at を含めて送信)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useReport` | 既存レポートデータの取得（フォームプリフィル + アクセス制御判定） | `state-management.md §3 データフェッチ系` |
| `useUpdateReport` | レポート更新 API のミューテーション | `state-management.md §3 ミューテーション系` |
| `useCurrentUser` | 所有権チェック（report.submitter.id と currentUser.id の比較）、ヘッダーのユーザー情報表示 | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| 既存レポートデータ | サーバー | useReport Hook（TanStack Query useQuery） | ReportEditPage |
| フォーム入力値（title, periodStart, periodEnd） | フォーム | React Hook Form + Zod（reportUpdateSchema） | ReportForm 内 |
| フィールドバリデーションエラー | フォーム | React Hook Form の formState.errors | ReportForm 内 |
| API ミューテーション状態（isPending, error） | サーバー | useUpdateReport Hook（TanStack Query useMutation） | ReportEditPage |
| API エラーメッセージ | UI | useState（useUpdateReport の onError から変換） | ReportEditPage |
| 楽観的ロック用 updated_at | UI | useState（useReport で取得した updated_at を保持） | ReportEditPage |
| データ読み込み中フラグ | サーバー | useReport Hook の isLoading | ReportEditPage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | ReportEditPage のルートラッパー | children にメインコンテンツを配置 |
| `AppBreadcrumbs`（← common-components.md） | メインコンテンツ上部 | items: [{ label: 'マイレポート', href: '/reports' }, { label: レポートタイトル, href: '/reports/:id' }, { label: '編集' }] |
| `AppTextField`（← common-components.md） | ReportForm 内のタイトル入力 | name: title、label: タイトル、required: true |
| `AppDatePicker`（← common-components.md） | ReportPeriodField 内の日付入力（x 2） | label: 開始日 / 終了日、required: true |
| `PageSkeleton`（← common-components.md） | 既存データ読み込み中のスケルトン UI | variant: 'form' |
| `AppToast`（← common-components.md） | API エラー・409 Conflict 時のトースト通知 | severity: 'error' |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `ReportEditPage` | 認証済みユーザー（全ロール）。ただし所有者のみアクセス可能 | - | `authz.md` &sect;6.3（PUT /api/reports/:id: 所有者のみ） |
| `ReportEditPage` | レポートが存在しない場合、一覧にリダイレクト + トースト | - | `screens/report-edit.md` &sect;6 |
| `ReportEditPage` | 所有者でない場合、403 トースト表示 | - | `screens/report-edit.md` &sect;6 |
| `ReportEditPage` | status が draft でない場合、詳細画面にリダイレクト + トースト | - | `screens/report-edit.md` &sect;6 |
| `ReportForm` | 既存データ読み込み完了後に表示（読み込み中は PageSkeleton） | 送信中は全フィールド + ボタンが disabled | `screens/report-edit.md` &sect;7, &sect;8 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | ReportEditPage | URL パス、対応 API |
| &sect;2 レイアウト | AppLayout, AppBreadcrumbs, ReportForm | 作成画面と同じフォーム構造。ボタンラベルのみ異なる |
| &sect;3 入力項目 | AppTextField, ReportPeriodField, AppDatePicker | 既存データでプリフィル |
| &sect;4 バリデーションルール | ReportForm（reportUpdateSchema） | V1-V4, V5-S, V5-E: 作成画面と同一ルール（V5 は両フィールド blur で `trigger(['periodStart', 'periodEnd'])` を呼んで再評価。issue #141） |
| &sect;5 エラー表示 | FormAlert, AppTextField, AppDatePicker | フィールドレベル + サーバーエラー |
| &sect;6 アクセス制御 | ReportEditPage | 存在チェック、テナント境界、所有権、draft 状態チェック |
| &sect;7 操作と遷移 | ReportFormActions, SubmitButton | 「保存する」: PUT -> 詳細画面遷移、「キャンセル」: 詳細画面に戻る。楽観的ロック |
| &sect;8 ローディング | PageSkeleton, SubmitButton | 初回読み込み: スケルトン、保存中: ボタン disabled + スピナー |
| &sect;9 ロール別表示差異 | - | ロールによる差異なし（所有者 AND draft のみアクセス可能） |
| &sect;10 画面遷移 | ReportEditPage | 詳細画面からの遷移元、保存完了・キャンセルの遷移先 |
| &sect;11 API リクエスト/レスポンス | useUpdateReport Hook | PUT /api/reports/:id（updated_at 付き） |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/report-edit.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/report-edit.md §ReportEditPage` | コンポーネント単体テスト | 既存データのプリフィル、アクセス制御（存在チェック・所有権・draft 状態）、保存成功時の詳細画面遷移、キャンセル遷移、409 Conflict エラーハンドリング |
| `55_ui_component/screens/report-edit.md §ReportForm` | コンポーネント単体テスト | report-create.md と共通（defaultValues でのプリフィル、submitLabel の切り替え確認） |
| `55_ui_component/state-management.md §useReport` | Hook 単体テスト | API 呼び出し・レポートデータ取得・エラーハンドリング |
| `55_ui_component/state-management.md §useUpdateReport` | Hook 単体テスト | API 呼び出し・updated_at 付きリクエスト送信・キャッシュ無効化・409 Conflict ハンドリング |

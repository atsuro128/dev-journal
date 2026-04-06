# レポート作成 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/report-create.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/report-create.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-RPT-002` |
| 画面名 | レポート作成 |
| URL パス | `/reports/new` |
| 画面詳細仕様 | `50_detail_design/screens/report-create.md` |

---

## 2. コンポーネントツリー

```
ReportCreatePage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── AppBreadcrumbs（← common-components.md）
        └── ReportForm
            ├── FormAlert
            ├── AppTextField（← common-components.md）[タイトル]
            ├── ReportPeriodField
            │   ├── AppDatePicker（← common-components.md）[開始日]
            │   └── AppDatePicker（← common-components.md）[終了日]
            └── ReportFormActions
                ├── SubmitButton（← common-components.md）[作成する]
                └── CancelButton
```

---

## 3. コンポーネント定義

### ReportCreatePage

- 配置: `pages/reports/ReportCreatePage.tsx`
- 責務: レポート作成画面のページコンポーネント。URL クエリパラメータ `?ref=:id` の存在を確認し、再申請フローの場合は useReport Hook で元レポートのデータを取得してフォームにプリフィルする。useCreateReport Hook でレポート作成のミューテーションを実行し、成功時に `/reports/:id` に遷移する
- 対応セクション: `50_detail_design/screens/report-create.md` &sect;1, &sect;3, &sect;6, &sect;8

```typescript
// Props なし（ページコンポーネント）
```

### ReportForm

- 配置: `components/report/ReportForm.tsx`
- 責務: レポートのタイトル・対象期間を入力するフォーム。React Hook Form + Zod（reportCreateSchema / reportUpdateSchema）でクライアントサイドバリデーションを行い、送信時に onSubmit コールバックを呼び出す。API エラーはフォーム上部の FormAlert に表示する。レポート作成画面（SCR-RPT-002）とレポート編集画面（SCR-RPT-003）で共有される
- 対応セクション: `50_detail_design/screens/report-create.md` &sect;3, &sect;4, &sect;5

Props 型定義は `common-components.md §ReportForm` を参照。

### FormAlert

- 配置: `components/ui/FormAlert.tsx`
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。メッセージが null の場合は非表示
- 対応セクション: `50_detail_design/screens/report-create.md` &sect;5

Props 型定義は `common-components.md §FormAlert` を参照。

### ReportPeriodField

- 配置: `components/report/ReportPeriodField.tsx`
- 責務: 対象期間の開始日・終了日を横並びに配置し、「~」の区切りテキストを表示する。各日付ピッカーは React Hook Form の Controller 経由で制御される
- 対応セクション: `50_detail_design/screens/report-create.md` &sect;3

Props 型定義は `common-components.md §ReportPeriodField` を参照。

### ReportFormActions

- 配置: `components/report/ReportFormActions.tsx`
- 責務: フォーム下部のアクションボタン群（キャンセル・送信）を右寄せで配置する
- 対応セクション: `50_detail_design/screens/report-create.md` &sect;6

Props 型定義は `common-components.md §ReportFormActions` を参照。

---

## 4. データフロー

```
POST /api/reports
  → useCreateReport()（← state-management.md §3 ミューテーション系）
    → ReportCreatePage
      → AppBreadcrumbs (props: items = [マイレポート, レポート作成])
      → ReportForm (props: onSubmit, onCancel, apiError, isPending, submitLabel, defaultValues?)
        → FormAlert (props: message = apiError)
        → AppTextField (props: React Hook Form の Controller 経由) [タイトル]
        → ReportPeriodField (props: control, errors, disabled)
          → AppDatePicker x 2 (props: value, onChange, errorMessage) [開始日, 終了日]
        → ReportFormActions (props: submitLabel, loading, onCancel)
          → SubmitButton (props: label, loading)

--- 再申請時の追加データフロー ---
GET /api/reports/:id（元レポート取得）
  → useReport(refId)（← state-management.md §3 データフェッチ系）
    → ReportCreatePage
      → ReportForm (props: defaultValues = 元レポートの title, periodStart, periodEnd)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useCreateReport` | レポート作成 API のミューテーション | `state-management.md §3 ミューテーション系` |
| `useReport` | 再申請時の元レポートデータ取得（`?ref=:id` がある場合のみ） | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | ヘッダーのユーザー情報表示 | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| フォーム入力値（title, periodStart, periodEnd） | フォーム | React Hook Form + Zod（reportCreateSchema） | ReportForm 内 |
| フィールドバリデーションエラー | フォーム | React Hook Form の formState.errors | ReportForm 内 |
| API ミューテーション状態（isPending, error） | サーバー | useCreateReport Hook（TanStack Query useMutation） | ReportCreatePage |
| API エラーメッセージ | UI | useState（useCreateReport の onError から変換） | ReportCreatePage |
| 再申請元レポート ID | UI | URL クエリパラメータ `?ref=:id`（useSearchParams） | ReportCreatePage |
| 再申請元レポートデータ | サーバー | useReport Hook（ref が存在する場合のみ enabled） | ReportCreatePage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | ReportCreatePage のルートラッパー | children にメインコンテンツを配置 |
| `AppBreadcrumbs`（← common-components.md） | メインコンテンツ上部 | items: [{ label: 'マイレポート', href: '/reports' }, { label: 'レポート作成' }] |
| `AppTextField`（← common-components.md） | ReportForm 内のタイトル入力 | name: title、label: タイトル、required: true |
| `AppDatePicker`（← common-components.md） | ReportPeriodField 内の日付入力（x 2） | label: 開始日 / 終了日、required: true |
| `AppToast`（← common-components.md） | API 通信エラー時のトースト通知 | severity: 'error' |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `ReportCreatePage` | 認証済みユーザー（全ロール） | - | `authz.md` &sect;6.3（POST /api/reports: 全ロール許可） |
| `ReportForm` | 常時表示 | 送信中は全フィールド + ボタンが disabled | `screens/report-create.md` &sect;6 |
| `ReportForm`（再申請時） | URL パラメータ `?ref=:id` が存在する場合、元レポートのデータでプリフィル | 元レポートが自分のもの AND rejected 状態である場合のみ | `screens/report-create.md` &sect;3, `authz.md` &sect;7 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | ReportCreatePage | URL パス、対応 API |
| &sect;2 レイアウト | AppLayout, AppBreadcrumbs, ReportForm | フォーム構造 |
| &sect;3 入力項目 | AppTextField, ReportPeriodField, AppDatePicker | タイトル、対象期間（開始日・終了日）。再申請時のプリフィル |
| &sect;4 バリデーションルール | ReportForm（reportCreateSchema） | V1-V5: クライアントサイド Zod バリデーション |
| &sect;5 エラー表示 | FormAlert, AppTextField, AppDatePicker | フィールドレベル: helperText、サーバーエラー: FormAlert |
| &sect;6 操作と遷移 | ReportFormActions, SubmitButton | 「作成する」ボタン: POST -> 詳細画面遷移、「キャンセル」: 一覧に戻る |
| &sect;7 ロール別表示差異 | - | ロールによる差異なし |
| &sect;8 画面遷移 | ReportCreatePage | 一覧・詳細からの遷移元、作成完了・キャンセルの遷移先 |
| &sect;9 API リクエスト/レスポンス | useCreateReport Hook | POST /api/reports |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/report-create.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/report-create.md §ReportCreatePage` | コンポーネント単体テスト | 通常作成フロー（フォーム送信 -> 詳細画面遷移）、再申請フロー（?ref=:id でプリフィル）、キャンセル遷移 |
| `55_ui_component/screens/report-create.md §ReportForm` | コンポーネント単体テスト | フォーム入力・バリデーション（V1-V5）・送信・エラー表示・disabled 制御・defaultValues によるプリフィル |
| `55_ui_component/screens/report-create.md §FormAlert` | コンポーネント単体テスト | message が null の場合は非表示、値がある場合は Alert 表示 |
| `55_ui_component/screens/report-create.md §ReportPeriodField` | コンポーネント単体テスト | 開始日・終了日の表示、日付入力、開始日 <= 終了日 の相関バリデーション |
| `55_ui_component/screens/report-create.md §ReportFormActions` | コンポーネント単体テスト | 送信ボタンの loading 表示、キャンセルボタンの onCancel コールバック |
| `55_ui_component/state-management.md §useCreateReport` | Hook 単体テスト | API 呼び出し・reference_report_id 付きリクエスト・キャッシュ無効化・エラーハンドリング |

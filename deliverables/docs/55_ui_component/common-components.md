# 共通コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 複数画面で共有するコンポーネントの一覧・Props 定義・使用箇所を定義する |
| 正本情報 | 共通コンポーネント名・Props 型定義・配置パス・使用箇所マトリクス |
| 扱わない内容 | 画面固有のコンポーネント（screens/*.md）、実装コード（Step 10）、ピクセル単位のスタイル（ui-guidelines.md） |
| 主な参照元 | `50_detail_design/ui-guidelines.md`, `40_basic_design/screens.md`, `50_detail_design/screens/*.md` |
| 主な参照先 | `55_ui_component/screens/*.md`, `60_test/test_cases/*.md` |

---

## 1. レイアウトコンポーネント

### AppLayout

- 配置: `components/layout/AppLayout.tsx`
- 責務: 認証済み画面の共通レイアウト（AppHeader + AppSidebar + メインコンテンツ領域）を提供する。`screens.md` &sect;4.1 に定義されたレイアウト構成を実装する。メインコンテンツ領域は `Container maxWidth="lg"` で制約する（`ui-guidelines.md` &sect;3 準拠）
- 使用箇所: SCR-DASH-001, SCR-RPT-001, SCR-RPT-002, SCR-RPT-003, SCR-RPT-004, SCR-WFL-001, SCR-WFL-002, SCR-ADM-001, SCR-ADM-002

```typescript
interface AppLayoutProps {
  /** メインコンテンツ領域に描画する子要素 */
  children: React.ReactNode;
}
```

### AuthLayout

- 配置: `components/layout/AuthLayout.tsx`
- 責務: 未認証画面（ログイン・サインアップ・パスワードリセット）の共通レイアウトを提供する。画面中央にフォームカードを配置し、ヘッダー・サイドナビゲーションを表示しない（`screens.md` &sect;4.1 準拠）。認証済みユーザーがアクセスした場合はダッシュボードにリダイレクトする
- 使用箇所: SCR-AUTH-001, SCR-AUTH-002, SCR-AUTH-003, SCR-AUTH-004

```typescript
interface AuthLayoutProps {
  /** フォームカード内に描画する子要素 */
  children: React.ReactNode;
}
```

---

## 2. UI ガイドラインのカスタムコンポーネント

ui-guidelines.md &sect;4 で「カスタム必要」と定義されたコンポーネント。

### AppDataGrid

- 配置: `components/ui/AppDataGrid.tsx`
- ベース MUI コンポーネント: `DataGrid` (Community 版)
- 責務: 日本語ロケール設定、共通カラム定義（日付・金額・ステータス列のフォーマッタ）、ソート・フィルタのデフォルト設定を統一する

```typescript
import { DataGridProps, GridColDef } from '@mui/x-data-grid';

interface AppDataGridProps extends Omit<DataGridProps, 'localeText'> {
  /** カラム定義 */
  columns: GridColDef[];
  /** 行データ */
  rows: readonly Record<string, unknown>[];
  /** ローディング状態 */
  loading?: boolean;
  /** 空状態メッセージ（データが0件の場合に表示） */
  emptyMessage?: string;
}
```

### AppTextField

- 配置: `components/ui/AppTextField.tsx`
- ベース MUI コンポーネント: `TextField`
- 責務: `size="small"` と `fullWidth` をデフォルト化し、エラー表示ヘルパーを統一する（`ui-guidelines.md` &sect;7 準拠）

```typescript
import { TextFieldProps } from '@mui/material';

interface AppTextFieldProps extends Omit<TextFieldProps, 'size' | 'fullWidth'> {
  /** フィールド名（フォームライブラリ連携用） */
  name: string;
  /** 表示ラベル */
  label: string;
  /** エラーメッセージ（存在する場合、error=true + helperText にセット） */
  errorMessage?: string;
}
```

### AppSelect

- 配置: `components/ui/AppSelect.tsx`
- ベース MUI コンポーネント: `Select`
- 責務: `size="small"` と `fullWidth` をデフォルト化し、空選択肢のプレースホルダーを統一する

```typescript
interface SelectOption {
  /** 選択肢の値（API 送信値） */
  value: string;
  /** 選択肢の表示テキスト */
  label: string;
}

interface AppSelectProps {
  /** フィールド名（フォームライブラリ連携用） */
  name: string;
  /** 表示ラベル */
  label: string;
  /** 選択肢一覧 */
  options: SelectOption[];
  /** 現在の選択値 */
  value: string;
  /** 値変更時のコールバック */
  onChange: (value: string) => void;
  /** プレースホルダーテキスト（未選択時に表示） */
  placeholder?: string;
  /** エラーメッセージ */
  errorMessage?: string;
  /** 必須フィールドか */
  required?: boolean;
  /** 無効化 */
  disabled?: boolean;
}
```

### AppDatePicker

- 配置: `components/ui/AppDatePicker.tsx`
- ベース MUI コンポーネント: `DatePicker` (`@mui/x-date-pickers`)
- 責務: 日本語ロケール (`ja`)、`YYYY/MM/DD` フォーマット、`size="small"` をデフォルト化する（`ui-guidelines.md` &sect;7 準拠）

```typescript
interface AppDatePickerProps {
  /** フィールド名（フォームライブラリ連携用） */
  name: string;
  /** 表示ラベル */
  label: string;
  /** 現在の日付値（YYYY-MM-DD 形式の文字列、または null） */
  value: string | null;
  /** 値変更時のコールバック */
  onChange: (value: string | null) => void;
  /** エラーメッセージ */
  errorMessage?: string;
  /** 必須フィールドか */
  required?: boolean;
  /** 無効化 */
  disabled?: boolean;
}
```

### AppHeader

- 配置: `components/layout/AppHeader.tsx`
- ベース MUI コンポーネント: `AppBar`
- 責務: アプリロゴ・ユーザーメニュー（ユーザー名・ロール名表示、ログアウト）・Drawer 開閉トリガーを統合する（`screens.md` &sect;4.2 準拠）

```typescript
/** ユーザー情報（openapi.yaml UserProfile に準拠） */
interface HeaderUser {
  /** ユーザー名 */
  name: string;
  /** テナント内ロール */
  role: 'admin' | 'approver' | 'member' | 'accounting';
}

interface AppHeaderProps {
  /** 現在のログインユーザー情報 */
  user: HeaderUser;
  /** サイドバー開閉コールバック（md 未満のレスポンシブ対応） */
  onToggleSidebar: () => void;
  /** ログアウトコールバック */
  onLogout: () => void;
}
```

### AppSidebar

- 配置: `components/layout/AppSidebar.tsx`
- ベース MUI コンポーネント: `Drawer`
- 責務: ナビゲーションメニュー項目のロール別表示制御とアクティブ状態ハイライトを統合する（`screens.md` &sect;4.3 準拠）。md 以上で常時表示（permanent）、md 未満でハンバーガーメニュー開閉（temporary）

```typescript
interface AppSidebarProps {
  /** 現在のユーザーロール（メニュー項目の表示制御に使用） */
  role: 'admin' | 'approver' | 'member' | 'accounting';
  /** 現在の URL パス（アクティブ状態ハイライトに使用） */
  currentPath: string;
  /** サイドバーの開閉状態（md 未満で使用） */
  open: boolean;
  /** サイドバー閉じるコールバック（md 未満で使用） */
  onClose: () => void;
}
```

---

## 3. 業務共通コンポーネント

複数画面で共有する業務固有のコンポーネント。

### ConfirmDialog

- 配置: `components/ui/ConfirmDialog.tsx`
- 責務: 操作前の確認ダイアログを提供する（`screens.md` &sect;4.6 準拠）。提出・削除・承認・却下・支払完了・明細保存時の期間外警告（ITM-007）の各操作で使用する。却下時の却下理由入力、承認時の承認コメント入力にも対応する
- 使用箇所: SCR-RPT-004

```typescript
interface ConfirmDialogProps {
  /** ダイアログの開閉状態 */
  open: boolean;
  /** ダイアログのタイトル */
  title: string;
  /** ダイアログ本文メッセージ */
  message: string;
  /** 確認ボタンのテキスト（例: 「提出する」「削除する」「承認する」「却下する」「支払完了にする」） */
  confirmLabel: string;
  /** 確認ボタンの色（ui-guidelines.md ボタン使い分け準拠） */
  confirmColor?: 'primary' | 'error' | 'success' | 'secondary';
  /** キャンセルボタンのテキスト（デフォルト: 「キャンセル」） */
  cancelLabel?: string;
  /** テキスト入力フィールドの設定（却下理由・承認コメント用） */
  inputField?: {
    /** 入力フィールドのラベル */
    label: string;
    /** プレースホルダー */
    placeholder?: string;
    /** 必須か（却下理由: true、承認コメント: false） */
    required: boolean;
    /** 最大文字数（1000文字） */
    maxLength: number;
    /** 複数行入力か */
    multiline: boolean;
  };
  /** 処理中かどうか（true の場合、確認ボタンを disabled + スピナー表示） */
  loading?: boolean;
  /** 確認ボタン押下時のコールバック。inputField がある場合は入力値を引数で受け取る */
  onConfirm: (inputValue?: string) => void;
  /** キャンセル・ダイアログ外クリック時のコールバック */
  onCancel: () => void;
}
```

### AppToast

- 配置: `components/ui/AppToast.tsx`
- 責務: 操作結果の通知を Snackbar + Alert で統一表示する（`ui-guidelines.md` &sect;8 準拠）。表示位置は画面上部中央（`anchorOrigin={{ vertical: 'top', horizontal: 'center' }}`）、成功通知は5秒で自動非表示、エラー通知は手動で閉じる
- 使用箇所: SCR-DASH-001, SCR-RPT-001, SCR-RPT-002, SCR-RPT-003, SCR-RPT-004, SCR-WFL-001, SCR-WFL-002, SCR-ADM-001, SCR-ADM-002
- **AuthLayout 配下は対象外**: 認証画面（SCR-AUTH-001〜004）ではトースト通知を使用しない。認証失敗・レート制限・サーバーエラーはすべてフォーム上部の Alert コンポーネントで表示する（各認証画面仕様の §S1〜S3 準拠、`state-management.md` §6 のエラー分類と整合）。AuthLayout がヘッダーを持たず画面中央にフォームカードのみを配置するレイアウトであるため、フォーム直上のインラインアラートが適切である

```typescript
/** トースト通知の種別（ui-guidelines.md §8 準拠） */
type ToastSeverity = 'success' | 'error' | 'warning' | 'info';

interface AppToastProps {
  /** トーストの表示状態 */
  open: boolean;
  /** 通知の種別 */
  severity: ToastSeverity;
  /** 通知メッセージ */
  message: string;
  /** 閉じるコールバック */
  onClose: () => void;
  /** 自動非表示までの時間（ミリ秒）。null で自動非表示を無効化。severity が 'error' の場合は null がデフォルト */
  autoHideDuration?: number | null;
}
```

### AppPagination

- 配置: `components/ui/AppPagination.tsx`
- ベース MUI コンポーネント: `Pagination`
- 責務: ページネーションコントロールの統一表示（`screens.md` &sect;4.9 準拠）。ページ番号 + 前へ/次へボタンを表示し、現在のページ番号をハイライトする。総ページ数が多い場合は省略表示する（例: 1 2 3 ... 8 9 10）
- 使用箇所: SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002

```typescript
interface AppPaginationProps {
  /** 現在のページ番号 */
  currentPage: number;
  /** 総ページ数 */
  totalPages: number;
  /** ページ変更時のコールバック */
  onPageChange: (page: number) => void;
  /** 無効化（ローディング中など） */
  disabled?: boolean;
}
```

### StatusChip

- 配置: `components/ui/StatusChip.tsx`
- 責務: 経費レポートのステータスを色分けした Chip で表示する（`screens.md` &sect;4.8、`ui-guidelines.md` &sect;5 準拠）
- 使用箇所: SCR-DASH-001, SCR-RPT-001, SCR-RPT-004, SCR-ADM-001

```typescript
/** 経費レポートのステータス（openapi.yaml ReportStatus に準拠） */
type ReportStatus = 'draft' | 'submitted' | 'approved' | 'rejected' | 'paid';

interface StatusChipProps {
  /** レポートのステータス値 */
  status: ReportStatus;
}
```

内部マッピング（実装ガイド）:

| status | 表示テキスト | MUI Chip color |
|--------|-------------|---------------|
| `draft` | 下書き | `default` |
| `submitted` | 提出済み | `info` |
| `approved` | 承認済み | `success` |
| `rejected` | 却下 | `error` |
| `paid` | 支払済み | `secondary` |

### EmptyState

- 配置: `components/ui/EmptyState.tsx`
- 責務: データが存在しない場合の空状態メッセージを統一表示する（`screens.md` &sect;4.7 準拠）
- 使用箇所: SCR-DASH-001, SCR-RPT-001, SCR-RPT-004, SCR-WFL-001, SCR-WFL-002, SCR-ADM-001

```typescript
interface EmptyStateProps {
  /** 空状態のメッセージテキスト */
  message: string;
  /** アクションボタン（例: レポート作成ボタン）。任意 */
  action?: {
    /** ボタンのテキスト */
    label: string;
    /** ボタン押下時のコールバック */
    onClick: () => void;
  };
}
```

### PageSkeleton

- 配置: `components/ui/PageSkeleton.tsx`
- 責務: API 通信中のローディング表示を統一する（`screens.md` &sect;4.5 準拠）。一覧画面用のテーブルスケルトンと詳細画面用のカードスケルトンを variant で切り替える
- 使用箇所: SCR-DASH-001, SCR-RPT-001, SCR-RPT-003, SCR-RPT-004, SCR-WFL-001, SCR-WFL-002, SCR-ADM-001, SCR-ADM-002

```typescript
interface PageSkeletonProps {
  /** スケルトンの表示パターン */
  variant: 'table' | 'card' | 'form';
  /** テーブル行数（variant が 'table' の場合に使用。デフォルト: 5） */
  rows?: number;
}
```

### AppBreadcrumbs

- 配置: `components/ui/AppBreadcrumbs.tsx`
- 責務: ページのパンくずナビゲーションを統一表示する（`ui-guidelines.md` &sect;4 で MUI Breadcrumbs を使用 OK と定義）。レポート詳細画面等の階層が深い画面で使用する
- 使用箇所: SCR-RPT-002, SCR-RPT-003, SCR-RPT-004

```typescript
interface BreadcrumbItem {
  /** パンくずのテキスト */
  label: string;
  /** 遷移先パス（最後の要素は現在のページのため href なし） */
  href?: string;
}

interface AppBreadcrumbsProps {
  /** パンくずアイテムの配列（先頭がルート、末尾が現在のページ） */
  items: BreadcrumbItem[];
}
```

### PageTitle

- 配置: `components/ui/PageTitle.tsx`
- 責務: ページタイトルを Typography で表示する
- 使用箇所: SCR-WFL-001, SCR-WFL-002, SCR-ADM-001, SCR-ADM-002

```typescript
interface PageTitleProps {
  /** ページタイトルテキスト */
  title: string;
}
```

### FormAlert

- 配置: `components/ui/FormAlert.tsx`
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。メッセージが null の場合は非表示。認証画面ではトースト通知を使用しないため、API エラー表示はこのコンポーネントが担う
- 使用箇所: SCR-AUTH-001, SCR-AUTH-002, SCR-AUTH-003, SCR-AUTH-004, SCR-RPT-002, SCR-RPT-003, SCR-RPT-004

```typescript
interface FormAlertProps {
  /** 表示するエラーメッセージ。null の場合は非表示 */
  message: string | null;
  /** Alert の severity（デフォルト: 'error'） */
  severity?: 'error' | 'warning' | 'info' | 'success';
}
```

### SubmitButton

- 配置: `components/ui/SubmitButton.tsx`
- 責務: フォーム送信ボタン。送信中は disabled + スピナー表示。認証画面・業務画面で共通利用する
- 使用箇所: SCR-AUTH-001, SCR-AUTH-002, SCR-AUTH-003, SCR-AUTH-004, SCR-RPT-002, SCR-RPT-003

```typescript
interface SubmitButtonProps {
  /** ボタンのラベルテキスト */
  label: string;
  /** 送信中フラグ（true の場合 disabled + スピナー表示） */
  loading: boolean;
  /** ボタンの type 属性（デフォルト: 'submit'） */
  type?: 'submit' | 'button';
  /** ボタンの MUI color（デフォルト: 'primary'） */
  color?: 'primary' | 'error' | 'success' | 'secondary';
  /** ボタンの variant（デフォルト: 'contained'） */
  variant?: 'contained' | 'outlined' | 'text';
  /** フル幅表示（デフォルト: true） */
  fullWidth?: boolean;
}
```

### SelfLabel

- 配置: `components/ui/SelfLabel.tsx`
- 責務: 自分が作成したレポートであることを示す「自分」ラベルを Chip で表示する。申請者名の横に配置する。`is_own_report` フラグが true の場合のみ表示する
- 使用箇所: SCR-WFL-001, SCR-WFL-002

```typescript
interface SelfLabelProps {
  /** 自分のレポートかどうか（true の場合のみラベル表示） */
  isOwnReport: boolean;
}
```

### FilterResetButton

- 配置: `components/ui/FilterResetButton.tsx`
- 責務: フィルタ条件をリセットするボタン。フィルタが適用されている場合にのみ有効化される
- 使用箇所: SCR-WFL-001, SCR-WFL-002

```typescript
interface FilterResetButtonProps {
  /** リセットコールバック */
  onReset: () => void;
  /** フィルタが適用中かどうか（false の場合 disabled） */
  isFiltered: boolean;
}
```

### AuthNavLinks

- 配置: `pages/auth/AuthNavLinks.tsx`
- 責務: 認証画面下部のナビゲーションリンクを表示する。画面ごとに表示するリンクが異なるため、links 配列を受け取る
- 使用箇所: SCR-AUTH-001, SCR-AUTH-002, SCR-AUTH-003, SCR-AUTH-004
- 注記: 認証画面専用のため `pages/auth/` に配置

```typescript
interface AuthNavLink {
  /** リンクの前に表示する説明テキスト */
  prefix: string;
  /** リンクテキスト */
  label: string;
  /** 遷移先パス */
  to: string;
}

interface AuthNavLinksProps {
  /** 表示するリンクの配列 */
  links: AuthNavLink[];
}
```

### ReportForm

- 配置: `components/report/ReportForm.tsx`
- 責務: レポートのタイトル・対象期間を入力するフォーム。React Hook Form + Zod（reportCreateSchema / reportUpdateSchema）でクライアントサイドバリデーションを行い、送信時に onSubmit コールバックを呼び出す。API エラーはフォーム上部の FormAlert に表示する。レポート作成画面（SCR-RPT-002）とレポート編集画面（SCR-RPT-003）で共有される
- 使用箇所: SCR-RPT-002, SCR-RPT-003

```typescript
interface ReportFormValues {
  /** レポートタイトル */
  title: string;
  /** 対象期間開始日（YYYY-MM-DD 形式） */
  periodStart: string;
  /** 対象期間終了日（YYYY-MM-DD 形式） */
  periodEnd: string;
}

interface ReportFormProps {
  /** フォーム送信時のコールバック */
  onSubmit: (data: ReportFormValues) => void;
  /** キャンセルボタン押下時のコールバック */
  onCancel: () => void;
  /** API エラーメッセージ（フォーム上部に Alert 表示） */
  apiError: string | null;
  /** 送信中フラグ（ボタン・入力フィールドの disabled 制御） */
  isPending: boolean;
  /** 送信ボタンのラベルテキスト */
  submitLabel: string;
  /** フォームの初期値（編集時・再申請時にプリフィル） */
  defaultValues?: ReportFormValues;
}
```

### ReportPeriodField

- 配置: `components/report/ReportPeriodField.tsx`
- 責務: 対象期間の開始日・終了日を横並びに配置し、「~」の区切りテキストを表示する。各日付ピッカーは React Hook Form の Controller 経由で制御される
- 使用箇所: SCR-RPT-002, SCR-RPT-003

```typescript
interface ReportPeriodFieldProps {
  /** React Hook Form の control インスタンス */
  control: Control<ReportFormValues>;
  /** 開始日のエラーメッセージ */
  periodStartError?: string;
  /** 終了日のエラーメッセージ */
  periodEndError?: string;
  /** 無効化フラグ */
  disabled?: boolean;
}
```

### ReportFormActions

- 配置: `components/report/ReportFormActions.tsx`
- 責務: フォーム下部のアクションボタン群（キャンセル・送信）を右寄せで配置する
- 使用箇所: SCR-RPT-002, SCR-RPT-003

```typescript
interface ReportFormActionsProps {
  /** 送信ボタンのラベルテキスト */
  submitLabel: string;
  /** 送信中フラグ（送信ボタンを disabled + スピナー表示） */
  loading: boolean;
  /** キャンセルボタン押下時のコールバック */
  onCancel: () => void;
}
```

---

## 4. 使用箇所マトリクス

| コンポーネント | SCR-AUTH-001 | SCR-AUTH-002 | SCR-AUTH-003 | SCR-AUTH-004 | SCR-DASH-001 | SCR-RPT-001 | SCR-RPT-002 | SCR-RPT-003 | SCR-RPT-004 | SCR-WFL-001 | SCR-WFL-002 | SCR-ADM-001 | SCR-ADM-002 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| AppLayout | --- | --- | --- | --- | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| AuthLayout | ○ | ○ | ○ | ○ | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| AppHeader | --- | --- | --- | --- | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| AppSidebar | --- | --- | --- | --- | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| AppPagination | --- | --- | --- | --- | --- | ○ | --- | --- | --- | ○ | ○ | ○ | --- |
| AppDataGrid | --- | --- | --- | --- | --- | ○ | --- | --- | ○ | ○ | ○ | ○ | --- |
| AppTextField | ○ | ○ | ○ | ○ | --- | --- | ○ | ○ | ○ | ○ | ○ | --- | --- |
| AppSelect | --- | --- | --- | --- | --- | ○ | --- | --- | ○ | --- | --- | ○ | --- |
| AppDatePicker | --- | --- | --- | --- | --- | ○ | ○ | ○ | ○ | --- | --- | ○ | --- |
| AppToast | --- | --- | --- | --- | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| ConfirmDialog | --- | --- | --- | --- | --- | --- | --- | --- | ○ | --- | --- | --- | --- |
| StatusChip | --- | --- | --- | --- | ○ | ○ | --- | --- | ○ | --- | --- | ○ | --- |
| EmptyState | --- | --- | --- | --- | ○ | ○ | --- | --- | ○ | ○ | ○ | ○ | --- |
| PageSkeleton | --- | --- | --- | --- | ○ | ○ | --- | ○ | ○ | ○ | ○ | ○ | ○ |
| AppBreadcrumbs | --- | --- | --- | --- | --- | --- | ○ | ○ | ○ | --- | --- | --- | --- |
| PageTitle | --- | --- | --- | --- | --- | --- | --- | --- | --- | ○ | ○ | ○ | ○ |
| FormAlert | ○ | ○ | ○ | ○ | --- | --- | ○ | ○ | ○ | --- | --- | --- | --- |
| SubmitButton | ○ | ○ | ○ | ○ | --- | --- | ○ | ○ | --- | --- | --- | --- | --- |
| SelfLabel | --- | --- | --- | --- | --- | --- | --- | --- | --- | ○ | ○ | --- | --- |
| FilterResetButton | --- | --- | --- | --- | --- | --- | --- | --- | --- | ○ | ○ | --- | --- |
| AuthNavLinks | ○ | ○ | ○ | ○ | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ReportForm | --- | --- | --- | --- | --- | --- | ○ | ○ | --- | --- | --- | --- | --- |
| ReportPeriodField | --- | --- | --- | --- | --- | --- | ○ | ○ | --- | --- | --- | --- | --- |
| ReportFormActions | --- | --- | --- | --- | --- | --- | ○ | ○ | --- | --- | --- | --- | --- |

凡例: ○ = 使用する / --- = 使用しない

### マトリクス補足

- **AppLayout**: 全認証済み画面（9画面）で使用。AppHeader と AppSidebar を内部で統合する
- **AuthLayout**: 全未認証画面（4画面）で使用。AppHeader / AppSidebar は含まない
- **AppPagination**: 一覧画面（SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002）でオフセットベースページネーションコントロールを表示する
- **AppTextField**: 認証画面のフォーム入力、レポート作成・編集のタイトル入力、明細スライドパネルの摘要入力、承認待ち・支払待ち一覧の申請者名フィルタで使用
- **AppSelect**: レポート一覧・テナント全レポート一覧のステータスフィルタ、テナント全レポート一覧の申請者フィルタ、明細スライドパネルのカテゴリ選択で使用
- **AppDatePicker**: レポート一覧の期間フィルタ、レポート作成・編集の対象期間、明細スライドパネルの日付入力、テナント全レポート一覧の期間フィルタで使用
- **ConfirmDialog**: レポート詳細画面のみで使用するが、提出・削除・承認・却下・支払完了・明細削除・添付削除の7種類の確認操作を担うため共通コンポーネントとして定義
- **PageTitle**: 承認待ち一覧・支払待ち一覧・テナント全レポート一覧・テナント情報の各画面でページタイトルを統一表示する
- **FormAlert**: 認証画面（4画面）とレポート作成・編集・詳細画面でフォーム上部のエラー表示を統一する。認証画面ではトースト通知を使用しないため、API エラー表示はこのコンポーネントが担う
- **SubmitButton**: 認証画面（4画面）とレポート作成・編集画面でフォーム送信ボタンの挙動（送信中 disabled + スピナー）を統一する
- **SelfLabel**: 承認待ち一覧・支払待ち一覧で自己レポートを識別する「自分」ラベルを表示する
- **FilterResetButton**: 承認待ち一覧・支払待ち一覧でフィルタリセットボタンを表示する
- **AuthNavLinks**: 認証画面（4画面）で画面下部のナビゲーションリンクを表示する。認証画面専用のため `pages/auth/` に配置
- **ReportForm**: レポート作成画面とレポート編集画面で共通利用するフォームコンポーネント。ReportPeriodField と ReportFormActions を内部に含む

---

## 5. ディレクトリ配置

```
frontend/src/components/
├── layout/                # レイアウトコンポーネント
│   ├── AppLayout.tsx      # 認証済み画面の共通レイアウト
│   ├── AuthLayout.tsx     # 未認証画面の共通レイアウト
│   ├── AppHeader.tsx      # ヘッダー（AppBar ラッパー）
│   └── AppSidebar.tsx     # サイドナビゲーション（Drawer ラッパー）
├── ui/                    # 共通 UI コンポーネント
│   ├── AppPagination.tsx  # ページネーションコントロール（Pagination ラッパー）
│   ├── AppDataGrid.tsx    # DataGrid ラッパー
│   ├── AppTextField.tsx   # TextField ラッパー
│   ├── AppSelect.tsx      # Select ラッパー
│   ├── AppDatePicker.tsx  # DatePicker ラッパー
│   ├── AppToast.tsx       # Snackbar + Alert トースト通知
│   ├── ConfirmDialog.tsx  # 確認ダイアログ
│   ├── StatusChip.tsx     # ステータスバッジ
│   ├── EmptyState.tsx     # 空状態メッセージ
│   ├── PageSkeleton.tsx   # ローディングスケルトン
│   ├── AppBreadcrumbs.tsx # パンくずナビゲーション
│   ├── PageTitle.tsx      # ページタイトル
│   ├── FormAlert.tsx      # フォーム上部エラーアラート
│   ├── SubmitButton.tsx   # フォーム送信ボタン
│   ├── SelfLabel.tsx      # 自己レポート識別ラベル
│   └── FilterResetButton.tsx # フィルタリセットボタン
├── report/                # レポート関連共通コンポーネント
│   ├── ReportForm.tsx         # レポート作成・編集フォーム
│   ├── ReportPeriodField.tsx  # 対象期間入力フィールド
│   └── ReportFormActions.tsx  # フォームアクションボタン群
└── (pages/auth/)          # 認証画面専用コンポーネント
    └── AuthNavLinks.tsx   # 認証画面ナビゲーションリンク
```

既存スケルトン（`expense-saas/frontend/src/components/`）の `layout/`, `ui/`, `report/` ディレクトリ構成と整合している。

---

## 6. 品質チェック

- [x] ui-guidelines.md &sect;4 の「カスタム必要」6コンポーネントが全て含まれているか（AppDataGrid, AppTextField, AppSelect, AppDatePicker, AppHeader, AppSidebar）
- [x] 全共通コンポーネントに TypeScript interface の Props 型定義があるか
- [x] 使用箇所マトリクスが全13画面をカバーしているか
- [x] 配置先ディレクトリが明示されているか
- [x] openapi.yaml のレスポンス型と Props 型が整合しているか（ReportStatus, UserSummary, ToastSeverity 等）
- [x] authz.md のロール別表示制御が Props に反映されているか（AppSidebar.role, AppHeader.user.role）
- [x] screens.md &sect;4 の共通UIパターン（エラー表示、ローディング、空状態、確認ダイアログ、ステータスバッジ、ページネーション（AppPagination: オフセットベース））が全てカバーされているか
- [x] 用語が glossary.md に準拠しているか（提出/却下/再申請/支払完了）
- [x] 既存スケルトン（`expense-saas/frontend/src/components/`）のディレクトリ構成と整合しているか
- [x] 画面固有のコンポーネントを含めていないか（5.5-C-* で定義）
- [x] 複数画面で共有するコンポーネント（PageTitle, FormAlert, SubmitButton, SelfLabel, FilterResetButton, AuthNavLinks, ReportForm, ReportPeriodField, ReportFormActions）が全て収録されているか

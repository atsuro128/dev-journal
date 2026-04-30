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
- カスタムフッター: MUI X DataGrid 標準の `slots.footer` プロパティを上書きしてカスタムフッターを差し込めるようにする。本プロジェクトでは `AppPaginationFooter` を `slots.footer` 経由で DataGrid 内部のフッターコンテナ（`<div class="MuiDataGrid-footerContainer">`）に統合するパターンを採用する。`slots.footer` を上書きする際は標準のページネーション UI と二重表示にならないよう `hideFooterPagination={true}` の併用を推奨する
- **件数 0 時の最小高さ確保**: 件数 0（`rows.length === 0`）時に `NoRowsOverlay` が描画されてもコンテナ高さ不足で隠れないよう、共通基盤側で `minHeight: 200`（px）を確保する。各画面側で minHeight を個別設定する必要はない（全 4 一覧画面で統一挙動とする）。issue #154 対応
- **横スクロール対応**: ルート Box は `overflowX: 'auto'` を付与し、列幅合計（各列の `minWidth` 合計）が画面幅を超える場合に DataGrid 領域内で横スクロールできるようにする。ページ全体の横スクロールにはせず、テーブル領域内に閉じ込める。スマホ幅（375px）で列内容が ellipsis 省略される問題（issue #160）を回避するため、各画面側で各列に適切な `minWidth` を設定することを推奨する
- 使用箇所（追加）: SCR-WFL-003（issue #158 / 11-A-issue-158、処理済みレポート一覧の DataGrid 表示）

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
  /** フォーカスアウト時のコールバック（RHF 連携用） */
  onBlur?: () => void;
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
  /** 現在の日付値（YYYY-MM-DD 形式の文字列。未入力時は空文字 `""`） */
  value: string;
  /** 値変更時のコールバック（未入力時は空文字 `""` を渡す） */
  onChange: (value: string) => void;
  /** フォーカスアウト時のコールバック（RHF 連携用） */
  onBlur?: () => void;
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
- 責務: 操作前の確認ダイアログを提供する（`screens.md` &sect;4.6 準拠）。提出・削除・承認・却下・支払完了・明細保存時の期間外警告（ITM-007）・編集中の変更破棄の各操作で使用する。却下時の却下理由入力、承認時の承認コメント入力にも対応する
- 多重送信防止のため、API mutate を伴う使用箇所では `loading` prop に mutation の `isPending` を渡すこと（`loading=true` 時は確認ボタン disabled + キャンセル / 外側クリック / Escape も無効化される）
- API エラー表示は原則トーストで通知しダイアログは閉じる（既存のアプリ全体パターンと整合）。例外として **`inputField.required` を持つダイアログ（却下ダイアログ）** では `apiError` パターンを採用し、エラー時はダイアログを閉じず `apiError` 経由で `FormAlert` にエラー文言を表示する（詳細は内部仕様 &sect;API エラー表示）
- 使用箇所: SCR-RPT-004

#### 内部仕様（必須入力バリデーション / ちらつき防止）

- **必須入力バリデーション**（issue #159 対応）: `inputField.required === true` の場合、入力フィールドは以下の挙動を取る:
  - フォーカスアウト（`onBlur`）で「touched 状態」を立てる
  - touched 状態かつ `!value.trim()` のとき、`inputField.errorMessage` が指定されていれば `helperText` に赤字エラー文言として表示する。未指定時はエラー文言を表示せず、文字数カウンタのみを表示する（フォールバック文言は提供しない。呼び出し側で必須項目には必ず `errorMessage` を指定すること。文言の所在を画面仕様書に紐付けて一元管理するため）
  - 文字数超過（`value.length > maxLength`）時も同様に赤字エラー文言を `helperText` に表示する
  - 確認ボタン（`onConfirm`）の `disabled` 制御も併用してよい（ボタン disabled = 誤送信防止の二段構え。詳細は `50_detail_design/screens/report-detail.md` &sect;4 D4 行参照）
- **メッセージちらつき防止**（issue #156 対応 / 大幅改修）: `open=false` の間、ダイアログ内に表示される全表示 props（`title` / `message` / `confirmLabel` / `confirmColor` / `inputField` / `apiError`）は前回の値を保持する（usePrevious 内部実装）。これにより閉じる際のフェードアウトアニメーション中に文字列が即座に空になって「ちらつき」が発生する事象を防ぐ。**`title` / `message` だけでなく、ボタン文言（`confirmLabel`）・色（`confirmColor`）・入力フィールド設定（`inputField`）・API エラー文言（`apiError`）も全て保持する** ことで、`workflowDialogAction = null` 等で親側 state が一斉に変わってもフェードアウト中のレイアウト・文言切替を完全防止する。`loading` のみ usePrevious 適用外（閉じる時 false に戻る挙動を維持）。次に `open=true` で新しい props が渡された時点で表示が更新される
- **API エラー表示**（issue #159 D4 規定対応）: `apiError` prop が設定されると `<FormAlert>` でダイアログ本文上部に表示する。エラー時はダイアログを閉じず `open=true` を維持して `apiError` を set することで、設計書 `50_detail_design/screens/report-detail.md` §11 D4 MissingRejectionReason 規定（必須入力 422 時にダイアログ内エラー表示、開いたまま）に準拠する。**適用範囲は `inputField.required` を持つダイアログ（却下ダイアログ D4）のみ**。`inputField` を持たない単純確認ダイアログ（提出 / 削除 / 承認 / 支払完了 / 明細削除 / 添付削除）では原則トースト + ダイアログ即時閉じる旧パターンを維持する（アプリ全体のエラー通知パターンと整合）。API 呼び出しを伴わない確認ダイアログ（明細保存時の期間外警告 ITM-007、編集中の変更破棄）でも `apiError` は不要

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
    /** バリデーションエラーメッセージ（required=true のとき、onBlur + 未入力時に helperText に表示。省略時は文字数カウンタのみ表示・エラー文言なし） */
    errorMessage?: string;
  };
  /** 処理中かどうか（true の場合、確認ボタンを disabled + スピナー表示） */
  loading?: boolean;
  /**
   * API エラー文言（422 MissingRejectionReason 等）。
   * null/undefined のとき非表示。FormAlert でダイアログ本文上部に表示する。
   * エラー時はダイアログを閉じず open を維持したまま apiError を set することで
   * 設計書 report-detail.md §11 D4 MissingRejectionReason 規定に準拠する。
   */
  apiError?: string | null;
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
- 責務: ページ番号のみを提供するページネーションコントロール（`screens.md` &sect;4.9 準拠）。ページ番号 + 前へ/次へボタンを表示し、現在のページ番号をハイライトする。総ページ数が多い場合は省略表示する（例: 1 2 3 ... 8 9 10）
- **per_page セレクタは扱わない**: 表示件数（per_page）の変更 UI は本コンポーネントに含めない。`PageSizeSelector` および両者を合成する `AppPaginationFooter` 側で扱う
- **常時表示前提**: 4 画面（SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002）では `AppPaginationFooter` 経由で利用される。利用者は `count={Math.max(totalPages, 1)}` を渡すことを前提とし、本コンポーネントは「totalPages == 0」のような条件付き非表示を行わない（フッター非表示仕様の撤廃）
- 使用箇所: SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002（いずれも `AppPaginationFooter` 経由）
- 使用箇所（追加）: SCR-WFL-003（issue #158 / 11-A-issue-158、`AppPaginationFooter` 経由）

```typescript
interface AppPaginationProps {
  /** 現在のページ番号 */
  currentPage: number;
  /** 総ページ数（呼び出し側で `Math.max(totalPages, 1)` を渡すこと） */
  totalPages: number;
  /** ページ変更時のコールバック */
  onPageChange: (page: number) => void;
  /** 無効化（ローディング中など） */
  disabled?: boolean;
}
```

### PageSizeSelector

- 配置: `components/ui/PageSizeSelector.tsx`
- ベース MUI コンポーネント: `Select`
- 責務: 1 ページあたりの表示件数（per_page）を選択する UI。標準選択肢 `[10, 20, 50, 100]` をデフォルトで提示し、URL 由来の標準外値（例: 1, 73）が現在値として渡された場合は動的に選択肢に追加して URL を正直に反映する
- 使用箇所: SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002（いずれも `AppPaginationFooter` 経由）
- 使用箇所（追加）: SCR-WFL-003（issue #158 / 11-A-issue-158、`AppPaginationFooter` 経由）

#### 配置・利用前提

- 単独でも利用可能だが、本 MVP では常に `AppPaginationFooter` 内で `AppPagination` と並置する
- 視覚的なラベル（`<InputLabel>` 等）は付与しない。MenuItem の表示テキストが「20 件」「50 件」と単位込みで表示されるため、ラベル無しでも UI 上の意味は文脈から読み取れる。スクリーンリーダー対応として Select に `aria-label="表示件数"` を付与する
- **フッター高さを支配しない**: 本コンポーネントは `AppPaginationFooter` 内でフッター 1 行に並置されるため、Select 枠線がフッター全体の高さを引き上げないようにする。`size="small"` を維持しつつ、必要に応じて `variant="standard"`（下線のみ）の採用や FormControl 余白（`margin="none"` / `sx={{ my: 0 }}` 等）の調整で薄い見え方にする。`AppPaginationFooter` 側の `minHeight: 52`（MUI 標準フッター高さ相当）を本コンポーネントが超えないことを実装フェーズで動作確認する

#### Props 型定義

```typescript
interface PageSizeSelectorProps {
  /** 現在の表示件数（URL クエリ `per_page` に対応する整数） */
  perPage: number;
  /** 標準選択肢。デフォルト `[10, 20, 50, 100]` */
  standardOptions?: number[];
  /** 表示件数変更時のコールバック（呼び出し側で URL 更新と page=1 リセットを行う） */
  onPerPageChange: (size: number) => void;
  /** ローディング中などで無効化 */
  disabled?: boolean;
}
```

#### 動的選択肢の挙動（重要）

- `perPage` が `standardOptions` に含まれる場合: 選択肢は `standardOptions` のまま
- `perPage` が `standardOptions` に含まれない場合: 選択肢を `[...standardOptions, perPage]` を昇順ソートし、`Set` 等で重複除去した配列に動的拡張する（動的選択肢の重複ガード。MUI の MenuItem `key` 重複 warning を回避）
- 例: `standardOptions=[10,20,50,100]`, `perPage=1` の場合 → 選択肢 `[1, 10, 20, 50, 100]`
- 例: `standardOptions=[10,20,50,100]`, `perPage=20` の場合 → 選択肢 `[10, 20, 50, 100]`（重複追加しない）

#### 表示文字列フォーマット

- 各 `<MenuItem>` の表示テキストは **`{size} 件`** とする（例: 「10 件」「20 件」「50 件」「100 件」）。日本語 UX 慣習に合わせて単位「件」を付与する
- MUI Select の現在値表示（combobox の `textContent`）も同形式（「20 件」等）で表示される
- テスト側で数値抽出を行う場合は `parseInt(option.textContent ?? '', 10)` を使う（`Number()` だと `NaN` になる）
- アクセシブル名（`getByRole('option', { name: ... })`）は **`'10 件'` のように単位込み**で指定する

#### サイズ・variant 方針

- `size="small"` を既定とする（MUI Select 標準の小サイズ。フッター高さを最小化）
- `variant="standard"` を確定採用とする。MUI X DataGrid 標準フッター（`@mui/material/TablePagination`）の Select は `variant="standard"` をハードコードしており（`node_modules/@mui/material/TablePagination/TablePagination.js` L260 で確認）、本プロジェクトの「MUI 標準寄せ」趣旨と整合させる。下線のみの薄い見た目になり、フッター内で枠線が目立たない
- FormControl 余白を `margin="none"` + `sx={{ my: 0 }}` で完全に排除し、`AppPaginationFooter` の最小高さ（`minHeight: 52`）を Select が支配しないようにする
- ラベル無し方針: 視覚的なラベル（`<InputLabel>`）を付与せず、Select 単体で配置する。MenuItem 表示テキストが「{size} 件」形式で単位込みのため文脈から意味が読み取れる。MUI 標準 TablePagination の左横インライン形式とは異なるが、SMK-081 検証で floating label の見栄え違和感が報告されたため撤去

#### アクセシビリティ

- 視覚ラベルが無いため、Select に `inputProps={{ 'aria-label': '表示件数' }}` を付与してスクリーンリーダー対応する
- `disabled` 時は `aria-disabled="true"` が MUI から自動付与される

#### テスト用識別子（`data-testid`）

- ルート（外側 `<FormControl>` または `<Box>`）: `data-testid="page-size-selector"`
- テストからは `screen.getByTestId('page-size-selector')` で参照する（PSS-001〜005、および 4 画面の Page 結合テスト）

### AppPaginationFooter

- 配置: `components/ui/AppPaginationFooter.tsx`
- ベース MUI コンポーネント: なし（`AppPagination` + `PageSizeSelector` の合成コンポーネント）
- 責務: 一覧画面のフッター 1 行に「中央: ページ番号（`AppPagination`）／右: 表示件数セレクタ（`PageSizeSelector`）」を並置する。レスポンシブ対応として 375px 等のスマホ幅では縦並びにフォールバックする
- 使用箇所: SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002
- 使用箇所（追加）: SCR-WFL-003（issue #158 / 11-A-issue-158、直接利用パターンで `AppDataGrid` の `slots.footer` に渡す）

#### 配置・利用前提

- **`AppDataGrid` の `slots.footer` プロパティに渡し、DataGrid フッターコンテナに統合する**。テーブル要素の外側に独立した `<Box>` として配置しない
- 利用パターンは以下の 2 系統を許容する。いずれの場合も `AppDataGrid` 自体は拡張せず、DataGrid 標準の `slots` props を活用する
  1. **直接利用パターン**（`ApprovalListPage` / `PaymentListPage` 等、`AppDataGrid` を直接呼び出す画面）: 画面側で `<AppDataGrid slots={{ footer: () => <AppPaginationFooter ... /> }} />` のように直接 `slots.footer` に `AppPaginationFooter` を渡す
  2. **中間ラッパー経由パターン**（既存中間ラッパー `pages/reports/ReportListTable.tsx` / `pages/admin/AllReportsTable.tsx` 等、`AppDataGrid` を内包する中間コンポーネントを持つ画面）: 中間ラッパーに `paginationFooter?: ReactNode` 等の prop を追加し、ラッパー内部で `slots.footer` に変換して `AppDataGrid` に渡す。画面側からは `paginationFooter={<AppPaginationFooter ... />}` の形で受け渡す
- `slots.footer` を上書きすると DataGrid 標準のページネーション UI と二重表示になるリスクがあるため、`hideFooterPagination={true}` の併用を推奨する（実装フェーズで動作確認）
- **マージン**: DataGrid フッターコンテナ内に統合する場合、外側マージン（`mt={2}` 等）は不要。テーブル外配置が前提だった旧仕様の `mt={2}` は撤去する
- **常時表示**: 内部 `AppPagination` には `count={Math.max(totalPages, 1)}` を渡し、`totalPages <= 1` でも非表示にせずページ番号「1」を表示する。`PageSizeSelector` も同様に常時表示する（件数変更により行数が増減した結果を確認できるようにするため）

#### Props 型定義

```typescript
interface AppPaginationFooterProps {
  /** 現在のページ番号 */
  currentPage: number;
  /** 総ページ数（0 や 1 でも内部で `Math.max(totalPages, 1)` を適用して常時表示） */
  totalPages: number;
  /** ページ変更時のコールバック */
  onPageChange: (page: number) => void;
  /** 現在の表示件数 */
  perPage: number;
  /** 表示件数変更時のコールバック（呼び出し側で URL 更新と page=1 リセットを行う） */
  onPerPageChange: (size: number) => void;
  /** PageSizeSelector に渡す標準選択肢（省略時は PageSizeSelector のデフォルト [10,20,50,100]） */
  standardOptions?: number[];
  /** ローディング中などで無効化（AppPagination / PageSizeSelector 双方に伝播） */
  disabled?: boolean;
  /**
   * 件数表示用の総件数。
   * - 指定された場合、フッター左側に「{start} - {end} / 全 {total} 件」を表示する
   * - 未指定（undefined）の場合は件数表示を非表示にする（後方互換、既存利用箇所への影響なし）
   * - `currentPage` / `perPage` / `totalCount` から内部で start/end を算出する（呼び出し側に算出ロジックを持たせない）
   * - API レスポンスの `pagination.total_count` をそのまま渡す想定
   */
  totalCount?: number;
}
```

##### 件数表示フォーマット

- 表示形式: **「{start} - {end} / 全 {total} 件」**（例: 「11 - 20 / 全 37 件」）
- 算出ロジック（コンポーネント内部で実施、呼び出し側には算出責務を持たせない）:
  - `start = (currentPage - 1) * perPage + 1`
  - `end = Math.min(currentPage * perPage, totalCount)`（端数ページの上限は `totalCount` と同値）
  - 端数ページ例: `totalCount=37, perPage=10, currentPage=4` → 「31 - 37 / 全 37 件」
- 0 件時: `totalCount === 0` の場合は **「0 - 0 / 全 0 件」** と表示する（非表示にはしない。フッター常時表示方針と整合させる。空状態でも件数 0 を明示することで「データが 0 件である」状態を MUI 標準フッターと同様に明示する）
- `totalCount` 未指定時のみ件数表示自体を出さない（後方互換）

#### レイアウト仕様

- ルート要素: `<Box display="flex" alignItems="center" flexDirection={{ xs: 'column', sm: 'row' }} gap={{ xs: 1, sm: 0 }} sx={{ borderTop: '1px solid', borderColor: 'divider', minHeight: 52, px: 2, py: 0.5 }}>`
- **境界線**: ルート要素に `borderTop: '1px solid'` + `borderColor: 'divider'` を付与する。MUI X DataGrid 標準フッター（`MuiDataGrid-footerContainer`）の border-top と揃え、リスト本体とフッターの境界を視覚的に明示する
- **最小高さ**: `minHeight: 52` 程度（MUI 標準 `TablePagination` の高さ相当）を設定する。`px: 2` / `py: 0.5` 等の余白で `PageSizeSelector` の Select 枠線がフッター全体の高さを支配しないように調整する
- **件数表示の配置**（`totalCount` 指定時のみ表示）:
  - `xs`（< 600px、375px 端末を含む）: `flex-direction: column` で縦並び。**件数表示が一番上**、続いて `AppPagination`、最後に `PageSizeSelector` の順
  - `sm` 以上（>= 600px）: `flex-direction: row` で横並び。**左: 件数表示**、中央: `AppPagination`、右: `PageSizeSelector`
- `sm` 以上での中央寄せ保証:
  - `totalCount` 指定時: 件数表示と PageSizeSelector それぞれを `<Box flex={1}>` でラップし、中央 `AppPagination` を `flex: 'none'` 相当（要素自身の幅）で配置する。両端要素の `flex={1}` 比で `1 : auto : 1` の配分が成立し、`gap` には依存しない（`gap` は要素間の隙間制御専用、中央寄せには使わない）
  - `totalCount` 未指定時: 件数表示要素を出さず、旧仕様の左スペーサー `<Box flex={1} sx={{ display: { xs: 'none', sm: 'block' } }} />` を入れて中央寄せを保証する（`xs` 時は縦並びレイアウトを優先するためスペーサーを非表示）
  - 共通制約: `gap={{ xs: 1, sm: 0 }}` のうち `sm` で `gap: 0` としているのは、両端要素の `flex={1}` で十分な間隔が確保されるため余分な `gap` を加えないという設計判断（`gap` を入れると右寄せ要素がさらに右にずれてレイアウトが崩れる）
- スペーサー / レイアウトは **DataGrid フッターコンテナの幅に追従する**前提。フッターコンテナは DataGrid のテーブル幅と一致するため、テーブル幅が変わっても中央寄せレイアウトは保たれる
- **中央寄せ手法は `flex` ベースで確定**: 上記「totalCount 指定時の両端 `flex={1}` ラッパー」「未指定時の左スペーサー `<Box flex={1}>`」を最終仕様とする。CSS Grid 等の代替案は採用しない（`flex` で十分かつ MUI 標準フッターも flex ベースのため標準寄せ趣旨と一致）
- 375px 縦並びテストは jsdom 上で完結しないため、`sx` prop / `flexDirection` の値検証で代替する

#### アクセシビリティ

- 内部の `AppPagination` および `PageSizeSelector` のアクセシビリティ仕様を継承する
- フッター全体に追加の `role`/`aria-*` は付与しない（MUI の Pagination / Select が個別に適切な ARIA 属性を提供する）

#### テスト用識別子（`data-testid`）

- ルート要素（最外殻の `<Box>`）: `data-testid="app-pagination-footer"`
- テストからは `screen.getByTestId('app-pagination-footer')` で参照する（APF-001〜007、および 4 画面の Page 結合テスト）
- 件数表示要素（`totalCount` 指定時のみ描画）: `data-testid="app-pagination-footer-count"`（APF-009 / APF-010 で参照）
- 内部の `AppPagination` / `PageSizeSelector` の testid はそれぞれの規約に準拠

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
- 使用箇所（追加）: SCR-WFL-003（issue #158 / 11-A-issue-158、空状態「処理済みのレポートはありません。」表示）

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
- 使用箇所（追加）: SCR-WFL-003（issue #158 / 11-A-issue-158、初回読み込み中のテーブルスケルトン）

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
  /** リンクの前に表示する説明テキスト（任意。省略時は label のみ表示） */
  prefix?: string;
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
| PageSizeSelector | --- | --- | --- | --- | --- | ○ | --- | --- | --- | ○ | ○ | ○ | --- |
| AppPaginationFooter | --- | --- | --- | --- | --- | ○ | --- | --- | --- | ○ | ○ | ○ | --- |
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
- **AppPagination**: 一覧画面（SCR-RPT-001, SCR-ADM-001, SCR-WFL-001, SCR-WFL-002）でオフセットベースのページ番号コントロールを表示する。本 MVP では常に `AppPaginationFooter` 経由で利用される
- **PageSizeSelector**: 同上 4 画面で表示件数セレクタ（per_page）を提供する。標準選択肢 `[10,20,50,100]`、URL の標準外 per_page 値は動的に追加表示する
- **AppPaginationFooter**: 同上 4 画面でフッター 1 行（中央: ページ番号、右: 表示件数セレクタ）を提供する。`xs` で縦並びにフォールバック。`totalPages <= 1` でも常時表示
- **AppTextField**: 認証画面のフォーム入力、レポート作成・編集のタイトル入力、明細スライドパネルの摘要入力、承認待ち・支払待ち一覧の申請者名フィルタで使用
- **AppSelect**: レポート一覧・テナント全レポート一覧のステータスフィルタ、テナント全レポート一覧の申請者フィルタ、明細スライドパネルのカテゴリ選択で使用
- **AppDatePicker**: レポート一覧の期間フィルタ、レポート作成・編集の対象期間、明細スライドパネルの日付入力、テナント全レポート一覧の期間フィルタで使用
- **ConfirmDialog**: レポート詳細画面のみで使用するが、提出・削除・承認・却下・支払完了・明細削除・添付削除・明細保存時の期間外警告（ITM-007）などの確認操作を担うため共通コンポーネントとして定義（却下時の却下理由入力・承認時の承認コメント入力にも対応）
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
│   ├── AppPagination.tsx  # ページ番号コントロール（Pagination ラッパー、per_page セレクタは含まない）
│   ├── PageSizeSelector.tsx       # 表示件数セレクタ（per_page、Select ラッパー、動的選択肢対応）
│   ├── AppPaginationFooter.tsx    # ページ番号 + 表示件数セレクタを 1 行配置（xs で縦並び）
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

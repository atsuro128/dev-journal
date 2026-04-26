# UI デザインガイドライン

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | UI の共通ルールと利用指針を定義する |
| 正本情報 | テーマ方針、共通コンポーネント利用方針、アクセシビリティ基本方針 |
| 扱わない内容 | 個別画面の詳細仕様（screens/*.md）、画面遷移（ui_flow.md） |
| 主な参照元 | `40_basic_design/screens.md`, `30_arch/adr/0001-tech-stack.md` |
| 主な参照先 | `50_detail_design/screens/*.md` |

## 1. 概要

本書は、経費精算SaaS（MVP）のフロントエンド実装における UI デザイン方針を定義する。MUI (Material UI) をコンポーネントライブラリとして使用し、実装者が統一的に UI を構築するための最低限のガイドラインを示す。

### 方針

- MVP フェーズでは MUI デフォルトテーマをベースに最小限のカスタマイズで進める
- カラー・フォント等のデザイントークンは `theme.ts` の1ファイル変更で全画面に反映可能な構成とする
- 過度なカスタマイズは避け、MUI のデフォルト挙動を最大限活用する

### 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `30_arch/architecture.md` §4 | フロントエンド構成・ディレクトリ |
| `30_arch/adr/0001-tech-stack.md` | MUI 選定理由 |
| `40_basic_design/screens.md` | 画面一覧・共通UIパターン |

---

## 2. テーマ設定方針

### カラーパレット

MUI デフォルトテーマをそのまま使用する。カスタマイズが必要になった場合は `theme.ts` を変更するだけで全画面に反映される。

| 役割 | カラーコード | MUI パレット名 |
|------|------------|---------------|
| プライマリ | #1976d2 (blue) | `primary` |
| セカンダリ | #9c27b0 (purple) | `secondary` |
| エラー | #d32f2f (red) | `error` |
| 警告 | #ed6c02 (orange) | `warning` |
| 成功 | #2e7d32 (green) | `success` |
| 情報 | #0288d1 (light blue) | `info` |

### フォント

- ベース: MUI デフォルト (Roboto)
- 日本語フォールバック: Noto Sans JP
- `theme.ts` の `typography.fontFamily` で設定する

```typescript
// frontend/src/theme.ts
const theme = createTheme({
  typography: {
    fontFamily: '"Roboto", "Noto Sans JP", "Helvetica", "Arial", sans-serif',
  },
});
```

### スペーシング

`theme.spacing` の基準値は MUI デフォルトの **8px** をそのまま使用する。全ての余白・間隔は基準値の倍数で指定する。

#### 余白スケール

| トークン名 | 値 | `theme.spacing()` | 主な用途 |
|-----------|-----|-------------------|---------|
| xs | 4px | `0.5` | アイコンとテキストの間隔、Chip 内余白 |
| sm | 8px | `1` | フォーム要素内のパディング、隣接ボタン間 |
| md | 16px | `2` | カード内パディング、フォーム要素間の垂直間隔 |
| lg | 24px | `3` | セクション間の間隔、カード間のギャップ |
| xl | 32px | `4` | ページ上下の余白、メインコンテンツの外側パディング |

#### 主要レイアウトでの使い方

| レイアウト要素 | 適用箇所 | 値 |
|--------------|---------|-----|
| カード内パディング | `Card` > `CardContent` の `sx.p` | `md` (16px) |
| フォーム要素間 | `Stack` の `spacing` | `md` (16px) |
| セクション間 | 見出し＋コンテンツブロック間の `mb` | `lg` (24px) |
| 一覧行間 | `DataGrid` の `rowSpacingType` は MUI デフォルト | デフォルト |
| ダイアログ内パディング | `DialogContent` の `sx.p` | `lg` (24px) |
| ページ外側余白 | `Container` の `sx.py` | `xl` (32px) |

```typescript
// frontend/src/theme.ts（spacing 部分）
const theme = createTheme({
  spacing: 8, // デフォルト値を明示（1 unit = 8px）
});

// 使用例: theme.spacing(2) → '16px', theme.spacing(0.5) → '4px'
```

### ダークモード

MVP では未対応とする。将来対応する場合は `theme.ts` の `palette.mode` を切り替える構成を想定する。

### テーマファイル配置

```
expense-saas/frontend/src/theme.ts
```

---

## 3. レイアウト方針

### 基本構造

認証済み画面は AppBar（上部固定） + Drawer（サイドバー） + メインコンテンツの構成とする（screens.md §4.1 準拠）。未認証画面（ログイン・サインアップ・パスワードリセット）ではAppBar・Drawer を表示しない。

```
+---------------------------------------------+
| AppBar (position="fixed")                   |
+----------+----------------------------------+
|          |                                  |
| Drawer   |  Container (maxWidth="lg")       |
|          |    メインコンテンツ                |
|          |                                  |
+----------+----------------------------------+
```

### ブレークポイント

MUI デフォルトのブレークポイントをそのまま使用する。

| ブレークポイント | 幅 |
|----------------|-----|
| xs | 0px 以上 |
| sm | 600px 以上 |
| md | 900px 以上 |
| lg | 1200px 以上 |
| xl | 1536px 以上 |

### レスポンシブ対応

- md 以上: Drawer を常時表示（permanent）
- md 未満: Drawer をハンバーガーメニューで開閉（temporary）
- レイアウトは MUI の `Grid` / `Stack` で構成し、ブレークポイントに応じてカラム数を変更する

### コンテンツ幅

メインコンテンツ領域は `Container maxWidth="lg"`（最大幅 1200px）で制約する。

---

## 4. コンポーネント使用方針

MUI コンポーネントを以下の3区分に分類し、実装者の判断基準を明確にする。

### 使用 OK — そのまま使用してよいコンポーネント

MUI のデフォルト Props・スタイルのまま利用する。プロジェクト固有のラッパーは不要。

| コンポーネント | 用途 | 備考 |
|--------------|------|------|
| `Button` | 全画面の操作ボタン | variant / color の使い分けは §4 末尾を参照 |
| `Typography` | テキスト表示全般 | variant でサイズ・ウェイトを統一 |
| `Card`, `CardContent`, `CardActions` | 詳細画面・サマリ表示 | パディングはスペーシング規約 (md: 16px) に従う |
| `Dialog`, `DialogTitle`, `DialogContent`, `DialogActions` | 確認ダイアログ | screens.md §4.6 の操作で使用 |
| `Snackbar` + `Alert` | トースト通知 | 成功/エラーを severity で区別（§8 参照） |
| `Chip` | ステータス表示 | color マッピングは §5 参照 |
| `Stack` | 垂直・水平レイアウト | spacing にスペーシングトークンを使用 |
| `Grid` | グリッドレイアウト | レスポンシブ対応に使用 |
| `Container` | ページ幅制約 | `maxWidth="lg"` 固定 |
| `Skeleton` | 初回読み込み表示 | 画面描画前のプレースホルダー |
| `CircularProgress` | 操作中ローディング | ボタン内や画面中央に表示 |
| `Tooltip` | 補足情報 | IconButton と組み合わせて使用 |
| `Breadcrumbs` | パンくずナビ | screens.md §4.3 準拠 |
| `IconButton` | ツールバー等の省スペースボタン | 必ず `Tooltip` と併用 |

### 非推奨 — 使用しないコンポーネント

以下のコンポーネントは MVP では使用しない。理由とともに代替手段を示す。

| コンポーネント | 非推奨の理由 | 代替 |
|--------------|------------|------|
| `Accordion` | MVP の画面設計に折りたたみ UI がない | `Card` でセクションを分割 |
| `Tabs` | 画面内タブ切替は screens.md に定義されていない | 画面遷移で対応 |
| `Stepper` | ウィザード形式のフローは MVP スコープ外 | 単一フォームで完結 |
| `SpeedDial` | モバイルファーストの FAB パターンは採用しない | 通常の `Button` |
| `BottomNavigation` | モバイル専用ナビは MVP スコープ外 | `Drawer` で統一 |
| `Table` (素の HTML テーブル) | 一覧画面ではソート・フィルタが必要 | `DataGrid` を使用 |

### カスタム必要 — ラップしてプロジェクト固有の Props・スタイルを適用するコンポーネント

以下のコンポーネントはプロジェクト共通のラッパーコンポーネントを作成し、統一した Props・スタイルで使用する。ラッパーは `frontend/src/components/` に配置する。

| MUI コンポーネント | ラッパー名 | カスタム内容 |
|-------------------|----------|-------------|
| `DataGrid` (Community 版) | `AppDataGrid` | 日本語ロケール設定、共通カラム定義（日付・金額・ステータス列のフォーマッタ）、ソート・フィルタのデフォルト設定 |
| `TextField` | `AppTextField` | `size="small"` デフォルト化、`fullWidth` デフォルト化、エラー表示の統一ヘルパー |
| `Select` | `AppSelect` | `size="small"` デフォルト化、`fullWidth` デフォルト化、空選択肢のプレースホルダー統一 |
| `DatePicker` | `AppDatePicker` | 日本語ロケール (`ja`)・`YYYY/MM/DD` フォーマット・`size="small"` のデフォルト化 |
| `AppBar` | `AppHeader` | アプリロゴ・ユーザーメニュー・Drawer 開閉トリガーの統合（screens.md §4.2 準拠） |
| `Drawer` | `AppSidebar` | ナビゲーションメニュー項目・アクティブ状態ハイライトの統合（screens.md §4.2 準拠） |

### ボタンの使い分け

| 用途 | variant | color | 例 |
|------|---------|-------|----|
| 主要操作（作成・提出・承認） | `contained` | `primary` | 「レポート作成」「提出する」 |
| 副次操作（キャンセル・戻る） | `outlined` | `primary` | 「キャンセル」 |
| 危険操作（削除・却下） | `contained` | `error` | 「削除する」「却下する」 |
| テキストリンク的操作 | `text` | `primary` | 「詳細を見る」 |

---

## 5. ステータス表示のカラーマッピング

経費レポートのステータスに対応する MUI `Chip` の `color` prop を以下の通りとする（screens.md §4.8 準拠）。

| ステータス | Chip color | 表示テキスト | 視覚的な色 |
|-----------|-----------|-------------|-----------|
| draft | `default` | 下書き | グレー |
| submitted | `info` | 提出済み | 青 |
| approved | `success` | 承認済み | 緑 |
| rejected | `error` | 却下 | 赤 |
| paid | `secondary` | 支払済み | 紫 |

使用例:

```tsx
<Chip label="提出済み" color="info" size="small" />
```

---

## 6. アイコン方針

- `@mui/icons-material` を使用する
- 操作ボタンにはアイコン + テキストを併用する（アイコンのみのボタンは避ける）
- ツールバーやアクション列など、スペースが限られる場合は `IconButton` + `Tooltip` で代替可

使用例:

```tsx
<Button startIcon={<AddIcon />} variant="contained">
  明細追加
</Button>
```

---

## 7. フォーム方針

### バリデーションエラー表示

`AppTextField` の `error` prop と `helperText` prop で表示する（screens.md §4.4 準拠）。MUI 素の `TextField` ではなく共通ラッパーの `AppTextField` を使うことで、必須マーカー方針（下記「必須フィールド」）を統一する。

```tsx
<AppTextField
  label="タイトル"
  error={!!errors.title}
  helperText={errors.title?.message}
  required
/>
```

### 必須フィールド

すべてのフォームの必須フィールドには `required` prop を**統一して付与する**。ただし、ラベルへのアスタリスク (`*`) は表示しない（HTML5 `required` 属性は input 要素にのみ付与される）。

**理由**:

- `getByLabelText('タイトル')` のようなラベル完全一致テストとの整合（`*` 付きラベルは検索キーが揺れて壊れやすい）
- 全画面で必須項目が大半を占めるため、`*` の情報量が乏しく視覚的ノイズになる
- a11y は HTML5 `required` 属性 + MUI による `aria-required` 自動付与で担保する（スクリーンリーダーが「必須」と読み上げる）
- 視覚的なマーカーよりも、blur / submit 時のバリデーションエラー文言で気付かせる方針（screens.md §4.4 準拠）

**共通コンポーネント側の責務**:

`*` 抑止は**共通コンポーネント側で実施する**（フォーム実装側で個別対応しない）。`AppTextField` / `AppSelect` / `AppDatePicker` のいずれも、`required` prop 受領時に **ラベルへの `*` を付与せず、input 要素にのみ `required` 属性を付与**する。具体的には `slotProps.htmlInput.required`（`AppTextField` / `AppDatePicker`）または `Select.inputProps.required`（`AppSelect`）経由で input にだけ HTML5 `required` を伝搬し、`FormControl` / `TextField` 自体の `required` prop はラベルに渡さない。

詳細は `dev-journal/deliverables/docs/55_ui_component/common-components.md` を参照。

### 日付入力

`@mui/x-date-pickers` の `DatePicker` を使用する。日本語ロケール（`ja`）を設定し、`YYYY/MM/DD` 形式で表示する。

### 金額入力

`TextField` に `type="number"` または `InputProps` でフォーマットを制御する。通貨は日本円（JPY）固定。表示時は3桁区切りカンマを付与する。

---

## 8. 通知 (トースト) 方針

`Snackbar` + `Alert` を組み合わせたトースト通知を使用する。

| 種別 | Alert severity | 使用場面 |
|------|---------------|---------|
| 成功 | `success` | 作成完了、提出完了、承認完了、支払完了 |
| エラー | `error` | API 通信エラー、権限エラー |
| 警告 | `warning` | 一時的な問題（リトライ促し等） |
| 情報 | `info` | 状態変更の通知 |

- 表示位置: 画面上部中央（`anchorOrigin={{ vertical: 'top', horizontal: 'center' }}`）
- 自動非表示: 5秒後（`autoHideDuration={5000}`）
- エラー通知は自動非表示しない（ユーザーが手動で閉じる）

---

## 9. 品質チェック

- [ ] 全画面で共通レイアウト（AppBar + Drawer）が適用されているか（未認証画面を除く）
- [ ] ステータス表示が本書のカラーマッピングに従っているか
- [ ] ボタンの variant / color が本書の使い分けに従っているか
- [ ] フォームの必須フィールドに `required` prop が設定されているか
- [ ] バリデーションエラーが `error` + `helperText` で表示されているか
- [ ] 操作ボタンにアイコン + テキストが併用されているか
- [ ] 確認ダイアログが screens.md §4.6 の操作に対して実装されているか
- [ ] ローディング表示（Skeleton / CircularProgress）が実装されているか
- [ ] 空状態メッセージが screens.md §4.7 に従って表示されているか
- [ ] レスポンシブ対応が md ブレークポイントで切り替わるか

---

## 10. 将来の拡張（MVP スコープ外）

以下は MVP では対応せず、将来検討する。

- **ダークモード対応**: `theme.ts` の `palette.mode` 切替で対応可能な構成は維持する
- **カスタムテーマ**: テナントごとのブランドカラー設定
- **アニメーション**: ページ遷移アニメーション、マイクロインタラクション
- **アクセシビリティ強化**: WCAG 2.1 AA 準拠の詳細な監査と対応
- **多言語対応 (i18n)**: 現時点では日本語のみ

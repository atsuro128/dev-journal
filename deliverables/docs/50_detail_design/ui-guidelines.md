# UI デザインガイドライン

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

画面種別に応じて以下の MUI コンポーネントを使用する。

| 画面種別 | 使用コンポーネント | 備考 |
|----------|-------------------|------|
| 一覧画面（レポート一覧、承認待ち等） | `DataGrid` (Community 版) | ソート・フィルタ機能を活用 |
| フォーム（作成・編集） | `TextField`, `Select`, `DatePicker` | バリデーション表示は `error` + `helperText` |
| 詳細画面 | `Card`, `Typography`, `Chip` | ステータスは Chip で表示 |
| 確認ダイアログ | `Dialog` | screens.md §4.6 の操作で使用 |
| 通知 | `Snackbar` + `Alert` | トースト通知。成功/エラーを色で区別 |
| ナビゲーション | `AppBar`, `Drawer`, `Breadcrumbs` | screens.md §4.2, §4.3 準拠 |
| ボタン | `Button` | 主操作: `variant="contained"`, 副操作: `variant="outlined"` |
| ローディング | `Skeleton`, `CircularProgress` | 初回読み込み: Skeleton, 操作中: CircularProgress |

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

`TextField` の `error` prop と `helperText` prop で表示する（screens.md §4.4 準拠）。

```tsx
<TextField
  label="タイトル"
  error={!!errors.title}
  helperText={errors.title?.message}
  required
/>
```

### 必須フィールド

`required` prop を使用する。MUI がラベルにアスタリスク (*)を自動付与する。

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

- 表示位置: 画面下部中央（`anchorOrigin={{ vertical: 'bottom', horizontal: 'center' }}`）
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

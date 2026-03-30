# レビュー結果: 8-3 フロントエンド初期化

- レビュー日: 2026-03-30
- レビュー対象: expense-saas/frontend/ 配下の全ファイル（新規作成）、docker-compose.yml の frontend サービスコメント解除
- 入力資料: architecture.md §4, チケット 8-3-frontend-init.md, step8-foundation.md

## 判定: PASS

下流工程（8-5 FE-BE 連携）が曖昧さなく作業できる状態である。

## 完了条件の検証

| 完了条件 | 結果 | 根拠 |
|---------|------|------|
| npm run build が通る | OK | `tsc -b && vite build` が正常終了（dist/assets/index-ghviMtR9.js 生成確認） |
| ディレクトリ構成が architecture.md §4.1 と一致する | OK | 全ファイル・ディレクトリが設計通りに配置されている（下記「構成比較」参照） |
| docker compose up で frontend サービスが起動する | OK（構成確認） | Dockerfile.dev と docker-compose.yml の frontend サービス定義が妥当。ポート 5173 公開、volumes マウント、api サービスへの depends_on が設定済み |

## レビュー観点別の検証

### 1. フロントエンドのビルドが通るか

`npm run build` 実行結果: 成功。TypeScript コンパイル（`tsc -b`）および Vite ビルド（`vite build`）ともにエラーなし。`npx tsc --noEmit` も警告・エラーゼロ。

### 2. ディレクトリ構成が architecture.md §4.1 と一致するか

architecture.md §4.1 で定義された全ファイル・ディレクトリが存在する。

| architecture.md §4.1 | 実体 | 状態 |
|----------------------|------|------|
| src/main.tsx | 存在 | ThemeProvider + QueryClientProvider + App のラッパー |
| src/App.tsx | 存在 | BrowserRouter + 8 ルート定義 |
| src/pages/ (8ページ) | 存在 | LoginPage, DashboardPage, ReportListPage, ReportDetailPage, ReportCreatePage, ReportEditPage, ApprovalListPage, PaymentListPage |
| src/components/ui/ | 存在 | .gitkeep のみ（適切） |
| src/components/layout/ | 存在 | .gitkeep のみ（適切） |
| src/components/report/ | 存在 | .gitkeep のみ（適切） |
| src/hooks/useAuth.ts | 存在 | 空スタブ（`export {};`） |
| src/hooks/useReports.ts | 存在 | 空スタブ |
| src/api/client.ts | 存在 | 空スタブ |
| src/api/auth.ts | 存在 | 空スタブ |
| src/api/reports.ts | 存在 | 空スタブ |
| src/api/types.ts | 存在 | 空スタブ |
| src/stores/auth.ts | 存在 | 空スタブ |
| src/lib/constants.ts | 存在 | 空スタブ |
| src/lib/format.ts | 存在 | 空スタブ |
| index.html | 存在 | lang="ja"、title="経費精算SaaS" |
| vite.config.ts | 存在 | react プラグイン、dev server 設定 |
| tsconfig.json | 存在 | strict: true、react-jsx |
| theme.ts | 存在 | MUI createTheme |
| package.json | 存在 | 依存関係定義 |

設計に存在しないが成果物として追加されたファイル: `Dockerfile.dev`（チケットの責務に明記されており適切）。

### 3. docker-compose.yml の変更が frontend セクションのコメント解除のみか

`git diff HEAD -- docker-compose.yml` で確認。変更箇所は frontend サービスブロックのコメント解除（行 38-49）のみ。api セクション・db セクション・volumes セクションに変更なし（api セクションの JWT パス修正・volumes 変更は 8-2 コミット `8ac141f` で実施済み）。

### 4. MUI テーマ、react-router、TanStack Query の基本設定が適切か

**MUI テーマ** (`theme.ts`):
- `createTheme` でカスタムパレット（primary/secondary）と日本語対応のフォントファミリーを設定
- `main.tsx` で `ThemeProvider` + `CssBaseline` でラップ済み

**react-router** (`App.tsx`):
- `BrowserRouter` + `Routes` + `Route` で 8 ルートを定義
- ルートパスが architecture.md §4.3 の全ページに対応
- `/reports/:id` と `/reports/:id/edit` で動的パラメータを使用

**TanStack Query** (`main.tsx`):
- `QueryClient` をデフォルトオプション付きで生成（`retry: 1`, `refetchOnWindowFocus: false`）
- `QueryClientProvider` で App 全体をラップ

### 5. Dockerfile.dev が開発用として適切に構成されているか

- ベースイメージ: `node:22-alpine`（LTS、軽量）
- `package.json` と `package-lock.json*` を先にコピーし `npm install` でキャッシュ活用
- EXPOSE 5173（Vite dev server デフォルトポート）
- CMD: `npm run dev`（Vite dev server 起動）
- docker-compose.yml のボリュームマウント（`./frontend:/app` + `/app/node_modules`）によりホットリロード対応

### 6. 8-3 のスコープ内に収まっているか

- API クライアント実装（8-5 の責務）: 含まれていない。`src/api/*.ts` は全て `export {};` の空スタブ
- ページコンポーネント実装（Step 10 の責務）: 含まれていない。全ページは `return <div>PageName</div>;` の最小スタブ
- フック・ストア・ユーティリティ: 全て空スタブ

スコープ逸脱なし。

## 指摘事項

| # | チェック種別 | 対象ファイル | 問題内容 | 重大度 | 推奨対応 |
|---|------------|------------|---------|--------|---------|
| 1 | 型安全性 | `frontend/src/vite-env.d.ts`（不在） | Vite の型定義ファイル `vite-env.d.ts`（`/// <reference types="vite/client" />`）が存在しない。現時点ではビルドに影響しないが、8-5 で `import.meta.env` を使用する際に型エラーまたは型補完が効かない問題が発生する | warning | `src/vite-env.d.ts` を作成し `/// <reference types="vite/client" />` を記述。8-5 着手前に対応推奨 |
| 2 | CI 連携 | `frontend/package.json` | `lint` スクリプトが未定義。architecture.md §8.1 では `eslint, tsc --noEmit` を CI で実行する設計だが、現時点で eslint の依存もスクリプトも存在しない | info | 8-8（CI/CD パイプライン）または 8-9（開発者ツール）で対応予定のため、8-3 のスコープ外として許容。ただし 8-8 チケットの前提として認識しておくこと |
| 3 | ビルド成果物 | `frontend/tsconfig.tsbuildinfo` | `tsc -b` が生成するインクリメンタルビルド情報ファイルがリポジトリに残存する可能性がある。root `.gitignore` に `tsconfig.tsbuildinfo` が含まれていない | info | 8-10（整理）の `.gitignore` 整備で `*.tsbuildinfo` を追加推奨 |

## 下流工程への影響評価

### 8-5（FE-BE 連携）への受け渡し

8-5 が必要とする以下の前提が全て整っている:

- `vite.config.ts` にプロキシ設定を追加可能な構造
- `src/api/client.ts` が空スタブとして存在し、fetch ラッパーの実装先が明確
- `src/stores/auth.ts` がトークン管理の実装先として存在
- `src/hooks/useAuth.ts` が認証フックの実装先として存在
- TanStack Query の基盤設定が完了

warning 指摘 #1（`vite-env.d.ts`）は 8-5 着手前に解消が望ましいが、ブロッカーではない（ビルドは通る）。

### Step 10（機能実装）への受け渡し

- 全 8 ページのスタブコンポーネントが存在
- ルーティング定義が完了
- MUI テーマが適用済み

問題なし。

# frontend バンドルの chunk size 500 kB 超過警告（code splitting 未適用、post-MVP）

## 発見日
2026-05-13

## カテゴリ
performance / frontend-build / post-mvp

## 影響度
低（ビルド・デプロイ・機能動作には影響なし。初回ロードのモバイル UX に潜在的影響）

## 発見経緯
proactive — Step 11-E Phase 0（frontend クリーンビルド）の `npm run build` 出力で vite からの警告を検出。

## 関連ステップ
- Step 11-E Phase 0 着手時に検出
- 対応は MVP リリース後（フロントエンド最適化スプリント）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外として扱う。MVP リリース前には対応しない。管理方式は **ops-080** で検討中。

## 問題

`expense-saas/frontend` の本番ビルドで vite が以下の警告を出力した:

```
dist/assets/index-B1Q7IuJ_.js  1,291.29 kB │ gzip: 391.47 kB │ map: 6,663.74 kB

(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking
- Adjust chunk size limit for this warning via build.chunkSizeWarningLimit
```

**現状の bundle 構成**:
- 単一 chunk に全コードが含まれる（code splitting 未適用）
- 主な重量物（推定）:
  - MUI Core / MUI X DataGrid（数百 kB）
  - React + React Router
  - 各画面コンポーネント（reports / approvals / payments / all-reports / admin）

**サイズ評価**:
- raw: 1,291 kB
- gzip 後: 391 kB
- vite の警告閾値: 500 kB（raw）

## 影響

### MVP リリース判定

なし。Step 11-E（デプロイ・スモークテスト）は通る。動作要件を満たす。

### 初回ロード UX

| 環境 | 影響 |
|------|------|
| 高速回線（社内デモ・有線）| 体感差なし（500ms 程度の差） |
| 4G モバイル（10 Mbps）| 初回ロードに +1〜2 秒（gzip 391 kB / 10 Mbps ≈ 0.3 秒だが TLS handshake / DNS 含めて延長） |
| 3G / 低速回線 | 初回ロードに +3〜5 秒、UX 悪化 |

ただし本プロジェクトは:
- ポートフォリオ用途で UAT は社内デモ限定（11-E Q2 で HTTP のみ・社内限定運用に確定済み）
- 一度ロードされればキャッシュで体感問題なし
- 業務用 SaaS の特性上、社外モバイル利用は想定しない

→ MVP では実害なし。

### ポートフォリオ品質（採用面接観点）

「フロントエンド最適化どう考えてる？」と問われた際の説明責任あり:
- 現状: code splitting 未適用、単一 chunk、警告残存
- 説明可能な状態: issue #177 起票済み、対応プラン明示、対応タイミング決定

### CI / ビルドログ

vite の警告がビルドログに残るため、CI 結果を見た時に警告が常にキレイにならない。リリース判定の視認性をわずかに損なう。

## 提案

### 対応タイミング

**MVP リリース後**（Step 11-F UAT 完了後）。フロントエンド最適化スプリントの一部として扱う。

### 対応プラン

#### 1. ルートベース code splitting（最優先）

`frontend/src/App.tsx` のルート定義で `lazy()` + `<Suspense>` を適用:

```tsx
// Before
import { ReportsPage } from './pages/reports/ReportsPage';

// After
const ReportsPage = lazy(() => import('./pages/reports/ReportsPage'));
```

対象（推定）:
- reports / approvals / payments / all-reports / admin / users / categories / dashboard

期待効果: 初回ロード時にダッシュボード + 認証画面のみ取得、他はナビゲーション時に取得。初期 bundle を 300〜500 kB（raw）程度に縮小可能。

#### 2. manualChunks による vendor 分離

`vite.config.ts` で重量ライブラリを separate chunk に:

```ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'vendor-react': ['react', 'react-dom', 'react-router-dom'],
        'vendor-mui': ['@mui/material', '@mui/icons-material', '@mui/system'],
        'vendor-mui-x': ['@mui/x-data-grid', '@mui/x-date-pickers'],
        'vendor-utils': ['date-fns', 'zod'],
      },
    },
  },
}
```

期待効果: vendor を別 chunk にして長期キャッシュ可能化。アプリ更新時の再ダウンロード対象を最小化。

#### 3. chunkSizeWarningLimit の調整（補助）

code splitting 適用後も MUI X DataGrid 等で 500 kB を超える chunk が残る場合に限り、`vite.config.ts` で `build.chunkSizeWarningLimit: 800` 程度に上げる。警告抑止のみ目的の上げ方は避ける。

#### 4. ソースマップ生成のデフォルト見直し

現状: `dist/assets/index-B1Q7IuJ_.js.map` が 6.6 MB 生成される。本番デプロイで `.map` を配信すると:
- メリット: 本番エラーのスタックトレース解析
- デメリット: 配信サイズ +6 MB、ソース構造の露出

検討案:
- `build.sourcemap: 'hidden'` で `.map` 生成するが index.js からの参照を削除（Sentry 等のエラートラッキング側にのみアップロード）
- または `build.sourcemap: false` で完全無効化

本 issue とは別案件として切り出す候補。

### 検証手順

1. `vite.config.ts` 修正
2. `npm run build` で chunk 構成変化を確認
3. 主要 chunk が 500 kB 未満かを確認
4. `npm run preview` でローカルプレビューし、ルート遷移時の chunk 取得を Network タブで確認
5. `e2e/` で動作デグレなしを確認

### 受容方針（MVP 期間中）

警告残置のまま MVP リリースする。説明可能な状態を維持:
- issue #177 で対応プラン記録済み
- 「ポートフォリオ用途で UAT は社内限定、フロントエンド最適化は MVP リリース後にスプリントで対応」と説明

## 関連 issue / PR

- 関連 issue: ops-080（Post-MVP スコープ管理方法）— 本 issue の管理方式
- 関連: Step 11-E 11-E チケット（Phase 0 着手時に検出）

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

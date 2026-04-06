# 一覧画面のページネーション方式をカーソルベースからオフセットベースに変更

## 発見日
2026-04-06

## カテゴリ
ui-design

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
Step 4（基本設計）、Step 5（詳細設計）、Step 5.5（UI コンポーネント設計）、Step 8（基盤構築）

## ブロッカー
なし

## 問題

一覧画面（レポート一覧、テナント全レポート一覧、承認待ち一覧、支払待ち一覧）のページネーション方式が「カーソルベース + さらに読み込むボタン」で設計されている。この方式では:

1. 「さらに読み込む」を繰り返すとブラウザに全件が蓄積され、画面が重くなる
2. 特定のページに直接飛べない
3. 経費精算のように過去データを検索する業務アプリにはページ番号式の方が適している

## 影響

以下の成果物に変更が必要:

### 上流（正本）
- `40_basic_design/screens.md` — 共通 UI パターンのページネーション定義
- `50_detail_design/screens/report-list.md` — レポート一覧のページネーション仕様
- `50_detail_design/screens/admin-all-reports.md` — テナント全レポート一覧のページネーション仕様
- `50_detail_design/screens/workflow-pending.md` — 承認待ち一覧のページネーション仕様
- `50_detail_design/screens/workflow-payable.md` — 支払待ち一覧のページネーション仕様
- `50_detail_design/openapi.yaml` — クエリパラメータを cursor から page + per_page に変更

### 下流（追従）
- `55_ui_component/state-management.md` — 一覧 Hook は useQuery のまま（useInfiniteQuery 不要）、パラメータ型に page を追加
- `55_ui_component/common-components.md` — ページネーション用の共通コンポーネント追加を検討
- `expense-saas/backend/` — リポジトリ層のカーソル処理をオフセット処理に変更

## 提案

1. screens.md の共通 UI パターンを「オフセットベース + ページ番号」に変更
2. 各画面詳細仕様のページネーションセクションを更新
3. openapi.yaml のクエリパラメータを `cursor` から `page` + `per_page` に変更
4. state-management.md のパラメータ型を更新（page: number を追加、cursor を削除）
5. バックエンドのリポジトリ層を修正

---

## 解決内容

## 解決日

# Step 8 の WB に共通 UI コンポーネント実装タスクが欠落している

## 発見日
2026-04-06

## カテゴリ
project-management

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 8（基盤構築）、Step 5.5（UI コンポーネント設計）、Step 10（機能実装）

## ブロッカー
なし

## 問題

テンプレートの正しいフローでは Step 5.5（UI コンポーネント設計）が Step 8（基盤構築）の前に完了しているため、Step 8 の時点で common-components.md に定義された共通 UI コンポーネント（FormAlert, SubmitButton, AppPagination, CountCard 等）を実装できるはず。

しかし現在の Step 8 の work-breakdown にはスケルトン作成（ディレクトリ構造・空ファイル）しか含まれておらず、共通 UI コンポーネントの実装タスクが定義されていない。

結果として、共通 UI コンポーネントの実装が Step 10（機能実装）の各機能タスク内で暗黙的に行われることになり、以下の問題が生じる:

1. 最初の機能タスク（10-A 認証）に共通コンポーネント実装の負荷が偏る
2. 並列で進める他の機能タスク（10-B〜10-G）が共通コンポーネントの完成を待つ必要がある
3. 共通コンポーネントの品質レビューが機能レビューに混在する

## 影響

- `ai-dev-framework/guide/work-breakdown/step8-foundation.md` — 共通 UI コンポーネント実装タスクの追加が必要
- テンプレートとしての正しいフロー（Step 5.5 → Step 8 → Step 10）を反映すべき

## 提案

1. `ai-dev-framework/guide/work-breakdown/step8-foundation.md` に「共通 UI コンポーネント実装」タスクを追加
   - 入力: `55_ui_component/common-components.md`
   - 出力: common-components.md に定義された全コンポーネントの実装コード
   - 担当: frontend-developer
   - 依存: フロントエンド初期化（8-3）
2. 今回のプロジェクトでは Step 8 完了済みのため、実装は Step 10 着手前に別途対応する

---

## 解決内容

## 解決日

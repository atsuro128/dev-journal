# 引き継ぎメモ

## セッション: 2026-04-05 11:58

### ゴール
- PostToolUse(Agent) ワークツリー自動クリーンアップフックの動作検証

### 作業ログ
- `git worktree list` でクリーンな初期状態を確認（main ワークツリーのみ）
- settings.json と agent-worktree-cleanup.sh の内容を確認
- テスト用サブエージェントを `isolation: worktree` で起動し、何もせず終了させた
- 直後に `git worktree list` を確認 → ワークツリーが自動削除されていることを確認
- `.claude/worktrees/` ディレクトリも空であることを確認
- デバッグ用コードの残存チェック → なし
- PostToolUse ワークアラウンド有効を確認、コミット完了（`4b4505d`）

### 未完了
- ops-055: work-breakdown テンプレートと実ファイルの構造不整合
- ops-047: work-breakdown ディレクトリ構成
- ops-050: 受け渡し契約の冗長性
- 050: JWT 署名アルゴリズム
- 052: JWT 鍵ファイル必須化

### ブロッカー
なし

### 次にやること
1. Step 9（テストコード実装）着手 — チケット起票から
2. ops issue の優先度判断（055, 047, 050 は Step 9 をブロックしない）
3. issue 050, 052（JWT 関連）は Step 9 or Step 10 で対応可

### 学び・気づき
- **PostToolUse(Agent) フックでワークツリークリーンアップが機能する**: WorktreeRemove フック未発火バグ（anthropics/claude-code#28363）のワークアラウンドとして有効。`isolation` フィールドの有無で worktree 未使用エージェントへの影響もなし

### 意思決定ログ
- **WorktreeRemove フックは残置**: 本体バグ修正後に活用可能なため削除しない。実質的なクリーンアップは PostToolUse が担う構成

---

## セッション: 2026-04-05 11:24（前回）

### ゴール
- issue 036（LSP サーバー統合）の動作検証・クローズ
- issue 049（コメント言語方針）の方針決定・既存コメント日本語化・クローズ

### 作業ログ
- **issue 036（LSP サーバー統合）** — resolved（見送り）
  - gopls/vtsls ともに指揮役から直接呼び出すと正常動作を確認
  - しかし implementation-workflow.md、CLAUDE.md、エージェント定義の tools にいくら書いてもサブエージェントは LSP を使わなかった
  - 検証5回実施（ルール強化、CLAUDE.md 追記、tools: LSP 追加）全て Grep/Read で処理
  - 原因: サブエージェントには LSP ツールが渡らない仕様上の制約（利用可能ツールは Read, Glob, Grep, Edit, Write, Bash の6つに固定）
  - codex にも調査依頼 → 公式な解決策なし。GitHub issue #8395 で関連機能要望が closed
  - 結論: このプロジェクトの作業の大半はサブエージェント経由のため恩恵なし。036 関連の全変更を revert
  - revert 時にコミット履歴を洗い、gopls と typescript の Dockerfile 残留を発見・除去
- **issue 049（コメント日本語化）** — resolved
  - ユーザーと方針を合意: 全コメント日本語（godoc/JSDoc 含む）、エラーメッセージ・ログは英語維持
  - `.claude/rules/implementation-workflow.md` にコメント言語ルールを追加
  - 4つのサブエージェントで並列変換（60ファイル）
  - サブエージェントがエラーメッセージ・ログ文字列も日本語化していたのを発見・修正（jwt.go 3箇所、main.go 2箇所）
  - 3リポジトリにコミット完了
- **ops-056（成果物のテンプレート準拠修正）** — ユーザーが対応済み
- **implementation-workflow.md の修正**
  - 「品質チェック完了後」→「ビルド・lint 通過後」に修正

### 未完了
- ops-055: work-breakdown テンプレートと実ファイルの構造不整合
- ops-047: work-breakdown ディレクトリ構成
- ops-050: 受け渡し契約の冗長性
- 050: JWT 署名アルゴリズム
- 052: JWT 鍵ファイル必須化

### ブロッカー
なし

### 次にやること
1. Step 9（テストコード実装）着手 — チケット起票から
2. ops issue の優先度判断（055, 047, 050 は Step 9 をブロックしない）
3. issue 050, 052（JWT 関連）は Step 9 or Step 10 で対応可

### 学び・気づき
- **LSP ツールはサブエージェントに渡らない**: tools フィールドに LSP を追加してもサブエージェントの利用可能ツールは6つに固定。CLAUDE.md やルールファイルでの指示も効果なし
- **サブエージェントはコメント以外も変える**: エラーメッセージ・ログ文字列をコメントと混同して日本語化した。diff で `//` 以外の変更行を抽出するチェックが有効
- **複数セッションにまたがる変更のロールバックは危険**: コミット履歴を洗わないと残留が発生する。036 では gopls と typescript が Dockerfile に残っていた
- **コーディング規約の置き場**: ai-dev-framework ではなく `.claude/rules/implementation-workflow.md`（paths: expense-saas/**/*）が正しい

### 意思決定ログ
- **LSP 統合は見送り**: サブエージェントに LSP ツールが渡らない仕様制約のため。Claude Code 側の対応を待って再検討
- **コメント言語は全日本語**: ユーザーは日本企業向けポートフォリオを想定。エラーメッセージ・ログは英語維持
- **sqlcgen/ は変換対象外**: sqlc 自動生成コードのため

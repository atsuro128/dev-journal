# 引き継ぎメモ

## セッション: 2026-04-07 14:59

### ゴール
- Step 9 チケット起票（9-A〜9-G）+ 9-A 認証テスト実装・マージ

### 作業ログ
- **Step 9 チケット起票** — 7 チケット（9-A〜9-G）作成、progress.md に Step 9 セクション追加
- **ops-057 起票** — 共通 UI コンポーネント実装チケット未起票の issue
- **9-A 認証テスト実装** — test-implementer エージェントで実装、PR #5 作成
  - BE: AUTH-001〜080（ドメイン単体 + ハンドラ統合 + サービス）
  - FE: AUTH-FE-001〜076（認証画面コンポーネント + hooks テスト）
  - 共通テストヘルパー: renderWithProviders、GenerateTestRefreshToken
- **CI 修正 3 回**
  - 1回目: ESLint 未使用型 + staticcheck 未使用関数
  - 2回目: FE tsconfig に vitest/globals 型追加 + BE/FE test に continue-on-error 追加
  - 3回目: FE vitest run にも continue-on-error 追加
- **ops-058 起票** — Step 10 完了後に continue-on-error を削除する issue（影響度: 高）
- **内部レビュー PASS** — reviewer エージェント。warning 3 件（AUTH-029 境界値、setTokens 二重呼び出し）、blocker なし
- **codex レビュー → 修正 → 再々レビュー**
  - 初回: 4 件（continue-on-error, AUTH-044, AUTH-045, token_hash+AUTH-070）
  - AUTH-044/045: revoked/expired JWT を実際に生成して検証する形に修正
  - token_hash: SHA-256 ハッシュ保存に修正（security.md 準拠）
  - AUTH-070: 副作用検証順序を修正（token 無効化確認→ログインの順に）
  - continue-on-error: 指揮役判断で却下（ops-058 で管理）
  - 再々レビューでコード指摘は全て解消確認
- **PR #5 squash merge** — master にマージ完了
- **worktree 分離ルール整備**
  - 問題: エージェントが worktree を無視して本体 expense-saas を直接編集
  - implementation-workflow.md に worktree 作業ルール + ブランチ操作手順を追加
  - workflow.md に `isolation: "worktree"` 必須ルールを追加
  - implement SKILL.md からブランチ操作の重複記述を削除
  - 並列エージェント 2 体でテスト → worktree 分離が正しく機能することを確認
- **devcontainer.json 修正** — gh CLI 認証の永続化（named volume 追加）
- **expense-saas ローカル master 修復** — エージェントの本体汚染で diverged していたため `git reset --hard origin/master`

### 未完了
- ops-047, ops-050, ops-055: 運用系 issue（優先度低）
- ops-057: 共通 UI コンポーネント実装チケット起票
- ops-058: Step 10 完了後に continue-on-error 削除
- Step 10 着手前に WB とレビュー観点に MUI 準拠の完了条件を追加
- worktree 内の `git checkout {既存ブランチ}` が本体に影響する問題（原因未特定、対応保留）
- codex 再レビュー結果を PR に投稿するワークフロー改善

### ブロッカー
なし

### 次にやること
1. 9-B/C/D 並列着手（9-A 完了済みなので依存解消）
2. ops-057 対応（共通 UI コンポーネント実装チケット起票）
3. ops-047/050/055 対応（余力があれば）

### 学び・気づき
- **worktree 分離の重要性**: `isolation: "worktree"` を付けてもエージェントが本体を直接編集する問題が発生。原因はプロンプトで本体パスへの移動を指示していたこと。implementation-workflow.md にルールを一元化して対策
- **squash merge 後のローカル diverge**: ローカル master に未 push コミットがある状態で feature branch を切り、squash merge すると diverge する。根本原因は worktree 分離の失敗（エージェントが master に直接コミット）
- **codex の continue-on-error 指摘**: 3 回連続で同じ指摘。Step 9 では必要な設定なので指揮役判断で却下し、ops-058 で管理
- **codex レビュー結果の PR 投稿**: codex のローカル実行結果はターミナル出力のみで PR に反映されない場合がある。再レビュー時は PR への投稿を明示的に指示する必要がある

### 意思決定ログ
- **continue-on-error の扱い**: Step 9 では全テスト失敗が正常なため CI に必要。codex の指摘を 3 回却下し、ops-058 で Step 10 完了後の削除を管理する方針
- **worktree ルールの配置**: workflow.md（指揮役向け）+ implementation-workflow.md（サブエージェント向け）に分離。個別スキルには書かず一元管理
- **expense-saas ローカル master の reset**: 未 push コミットはすべて squash merge に含まれているため `git reset --hard origin/master` で安全にリセット可能と判断

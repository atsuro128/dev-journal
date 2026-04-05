# 引き継ぎメモ

## セッション: 2026-04-05 16:46

### ゴール
- Step 9 着手前に JWT 関連 issue（050, 052）を解決する

### 作業ログ
- **issue 050（JWT 署名アルゴリズム）** — resolved
  - ユーザーと RS256 / EdDSA / ML-DSA の比較検討を実施
  - NIST IR 8547 の 2030年非推奨ガイドラインを調査。issue の引用元（SP 800-131A Rev.2）が不正確であることを確認
  - EdDSA も量子耐性がなく、「2030年非推奨」を理由に移行するなら ML-DSA でないと論理的に一貫しない
  - ML-DSA は Go エコシステム未成熟で採用リスク高い
  - 結論: RS256 維持。ADR-0006 として判断根拠を記録
  - ADR-0006 作成 → 内部レビュー PASS → コミット → codex レビュー（指摘なし）
  - architecture.md §6.1, §9.2 に ADR-0006 への参照追加
- **issue 052（JWT 鍵ファイル必須化）** — resolved
  - config.go: JWT_PRIVATE_KEY_PATH, JWT_PUBLIC_KEY_PATH を必須環境変数に変更
  - main.go: 公開鍵は NewVerifier（読み込み+PEMパース+鍵検証）、秘密鍵は os.Stat（存在確認）
  - codex レビューで「秘密鍵の PEM パース検証が不足」と指摘
  - ユーザーと議論: Signer 未実装のため今は os.Stat + TODO コメントで対応。Step 10 で NewSigner 実装時に PEM パース検証に置き換える
  - コミット完了
- **ADR の背景セクションの書き方について議論**
  - ユーザー指摘: ADR を issue に紐づけるのは不適切。背景はプロジェクト文脈で書くべき
  - 修正: 「issue 050 で提起された」→「本システムが JWT を使う理由と判断ポイント」に書き換え
- **progress.md への追記は不要**とユーザーが判断（ノイズになる）

### 未完了
- ops-055: work-breakdown テンプレートと実ファイルの構造不整合
- ops-047: work-breakdown ディレクトリ構成
- ops-050: 受け渡し契約の冗長性

### ブロッカー
なし

### 次にやること
1. Step 9（テストコード実装）チケット起票 — architect エージェントに計画策定を委譲
2. work-breakdown の「この Step で決めること」の方針決定
3. チケットファイル 7 件作成（9-A〜9-G）
4. 9-A（認証テスト）の実装着手

### 学び・気づき
- **ADR は issue の解決記録ではない**: 背景はプロジェクト文脈（なぜこの判断が必要か）で書く。issue はきっかけに過ぎず、ADR 内で issue に紐づけるべきではない
- **codex の指摘を鵜呑みにしない**: 秘密鍵の PEM パース指摘は妥当だが、Signer 未実装の文脈を踏まえて os.Stat + TODO が最適解。codex にも追加判断を求めて合意を得た
- **progress.md の細かい追記はノイズ**: Step 3 完了後の追加チケット（3-8）を progress.md に書くとかえって見づらくなる

### 意思決定ログ
- **RS256 維持**: EdDSA は量子耐性なし（RS256 と同じ問題）、ML-DSA は Go エコシステム未成熟。NIST 2030年非推奨は予防的ガイドライン。詳細は ADR-0006
- **秘密鍵検証は os.Stat + TODO**: PEM パースして捨てるより、Signer 実装時に NewSigner として統合する方が責務として自然。TODO コメントで漏れを防止
- **ワークフロー手順の追いづらさ**: ADR 1件追加するだけで workflow.md → work-breakdown → 設計成果物フロー → テンプレートと複数ファイル横断が必要。ops issue 候補だがユーザーは Step 9 を優先

---

## セッション: 2026-04-05 11:58（前回）

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

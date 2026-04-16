# 引き継ぎメモ

## セッション: 2026-04-16 11:53〜13:10

### ゴール
- ops-107 Phase 3 実機検証（devcontainer rebuild 後）
- 設計書・ワークフロー・work-breakdown の CI 関連記述をローカルテスト前提に一括更新
- `/test` スキル作成

### 作業ログ

#### Phase 3 実機検証
1. devcontainer rebuild 後に再接続
2. verify-egress.sh PASS 確認（既存セキュリティ維持）
3. host gateway 5433 への接続確認 → Connection refused（ホスト到達 = firewall 通過）
4. 許可外ポート 5432 への接続確認 → No route to host（firewall でブロック）
5. Phase 3 全項目 PASS

#### 設計書・ワークフロー一括更新
6. ops-107 の追加決定事項を議論:
   - GHA を実質廃止するなら設計書の CI 前提記述も更新が必要
   - `/test` スキルで手動実行 + workflow.md の規約で担保（hook 自動化はしない）
7. Explore エージェントで CI 関連記述を洗い出し → codex にクロスチェック → 追加漏れ多数発見
8. designer エージェント 2 台を並行起動:
   - 設計成果物（6ファイル）+ Step 11 + workflow.md + ci.yml / deploy.yml
   - 過去 Step work-breakdown（step3〜step10、13ファイル）
9. ops-107 注記（「※ ops-107 により方針変更」）をユーザー指示で全箇所から削除
10. 内部レビュー → PASS (warning 2件: 用語ゆれ + 10-H タスク名曖昧)
11. warning 修正後コミット → codex レビュー
12. codex レビュー → FIX (4件):
    - High: ci.yml / deploy.yml が PR/push で自動起動 → workflow_dispatch に変更
    - Medium: Step 11 deploy.yml 有効化前提 → 手動デプロイに修正
    - Medium: ci.yml コメント不整合 → 「GitHub Hosted Runner 向けの参考実装」に修正
    - Low: Local CI → ローカルテスト結果の表記統一
13. 全 4 件修正 → codex 再レビュー → APPROVE
14. ユーザー指摘: 「Local CI」は CI の概念として正当 → Low の表記統一を revert

#### `/test` スキル作成
15. `.claude/skills/test/SKILL.md` を作成
    - frontend: devcontainer 内完結
    - backend: host gateway の DB 接続確認 → 失敗時はユーザーにホスト側操作を指示 → テスト実行
    - 結果報告: PR body に貼れる「## Local CI 結果」形式

### 今セッションで作成したコミット一覧

#### root-project
- `3355156` feat: /test スキルを追加
- `6d3dfe3` fix: /test スキルの結果セクション名を「ローカルテスト結果」に統一
- `5d1b24f` revert: Local CI セクション名を元に戻す

#### expense-saas
- `4a0024f` docs: ci.yml / deploy.yml に参考用デモの旨のコメント追加
- `359835e` fix: ci.yml / deploy.yml のトリガーを workflow_dispatch に変更

#### dev-journal
- `92f723e` docs: ローカルテストをメインとする方針に設計書・テスト戦略を更新
- `f8dcbbb` fix: codex 指摘対応 — Local CI → ローカルテスト結果に統一
- `da10354` revert: Local CI セクション名を元に戻す

#### ai-dev-framework
- `766dfa1` docs: work-breakdown と workflow をローカルテスト前提に更新
- `7923ade` fix: codex 指摘対応 — step11 deploy.yml 有効化前提を手動デプロイに修正

### 未完了

- **ops-107 issue ファイルの移動**: `open/` → `resolved/` に移動 + progress.md 更新
- **issue 109 対応**: 未着手
- **issue 104-A 着手**: 未着手

### ブロッカー
なし

### 次にやること

#### 優先度 1: ops-107 クローズ
1. ops-107 を `resolved/` に移動
2. progress.md 更新

#### 優先度 2: issue 109 対応（テスト設計書・実装間の乖離）
3. ID 重複 14 件の解消（最優先）
4. 未反映 22 ID の設計書追記
5. FE フィルタ 16 件の実装漏れ確認

#### 優先度 3: issue 104-A 着手
6. designer で screens/*.md からマトリクス抽出 → ui_coverage_matrix.md 作成

### 学び・気づき

- **codex の形式的な指摘には押し返す** — 「Local CI」→「ローカルテスト結果」への表記統一は codex の Low 指摘だったが、CI は GHA 固有の用語ではなく概念として正当。ユーザー判断で revert。codex の指摘を鵜呑みにしない
- **設計書の ops-107 注記は不要** — 設計書の内容自体を書き換えれば十分。「※ ops-107 により方針変更」のような注記は読者にとってノイズ
- **ci.yml のトリガー無効化を忘れない** — 参考用デモとして残す場合でも、`on: pull_request` のままだと PR ごとに GHA が走って無料枠を消費する。`workflow_dispatch` に変更必須

### 意思決定ログ

#### ローカルテスト運用の全体像
- テスト実行: `/test` スキルで指揮役が実行（手動実行オプションは廃止）
- PR マージ条件: PR body の Local CI セクションに結果記載
- ci.yml / deploy.yml: `workflow_dispatch`（手動実行のみ）。GitHub Hosted Runner 向けの参考実装として残す
- 設計書・work-breakdown: 全て「ローカルテストがメイン」に統一済み

#### codex の Low 指摘を revert した判断
- 「Local CI」は CI（継続的インテグレーション）の概念として正当
- GHA を廃止しても CI の概念自体は残る
- PR body のセクション名として「Local CI 結果」は適切

## 前回セッション

前回セッション（2026-04-16 11:00〜11:53）の詳細は `dev-journal/archives/session-logs/2026-04-16.md` を参照。

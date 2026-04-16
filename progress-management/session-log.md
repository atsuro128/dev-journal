# 引き継ぎメモ

## セッション: 2026-04-16 09:50〜10:56

### ゴール
- Group F-100（issue 100）実装 → PR レビュー → マージ
- issue 096（catch-all 404 ルート）実装 → PR レビュー → マージ
- issue 104 sub-issue 起票（104-A / 104-B）
- codex にテスト設計書・実装間の乖離調査を依頼

### 作業ログ

#### Phase 1: issue 100 実装（PR #58）
1. `/session-start` で状態確認 → Step 11-A 進行中、優先度 1 は Group F-100
2. `/issue 対応 100` で issue 確認、前セッション合意済みの方針（outlined Button + CircularProgress + DnD 視覚化 + 文言「ファイルを追加」）を再確認
3. frontend-developer を worktree で起動 → PR #58 作成
4. 変更: AttachmentUploader.tsx（VisuallyHiddenInput + Button + CircularProgress + DnD ゾーン）、テスト 16 件 PASS

#### Phase 2: issue 104 sub-issue 起票 + issue 096 並行
5. issue 100 エージェント待ちの間に issue 104 の下調べ → 104-A（リサーチ）/ 104-B（実装）のドラフト作成
6. ユーザーから「DnD 視覚化とは何？」の質問 → ドラッグ&ドロップのドロップ可能領域を視覚的に示すことと説明
7. 104-A / 104-B issue ファイルを `dev-journal/issues/open/` に作成

#### Phase 3: PR #58 内部レビュー + issue 096 並行起動
8. PR #58 内部レビュー（reviewer）→ PASS（W1: テスト ID 衝突、W2: dragleave バブリング、W3: 設計書表記乖離）
9. ユーザー指摘「テスト設計書にも反映が必要」→ designer に attachments.md 更新を委任
10. designer + frontend-developer（issue 096）を並列起動
11. designer 完了: ATT-FE-051〜053 の ID 確定
12. 指揮役が PR #58 に W1（ID 振り直し）+ W2（dragleave 修正）を push（`941fa35`）

#### Phase 4: codex 乖離調査 + PR レビュー
13. ユーザーから「最近の issue 対応でもテスト設計書に未反映があるのでは」→ codex に横断的な乖離調査を依頼
14. PR #59（issue 096）完了: NotFoundPage 新設 + App.tsx catch-all ルート追加、テスト 3 件 PASS
15. PR #58 codex レビュー → **FIX**（blocker: isPending 中でも DnD で再アップロード可能）
16. 指揮役が修正: handleDragOver / handleDrop に isPending ガード追加 + ATT-FE-028 テスト補強（`462f531`）
17. PR #59 内部レビュー PASS + codex APPROVE 相当 → スカッシュマージ
18. PR #58 codex 再レビュー → APPROVE 相当 → スカッシュマージ

#### Phase 5: ハウスキープ + 乖離調査結果
19. codex 乖離調査結果: 実装にあるが設計書にない 22 ID / 設計書にあるが実装にない 99 件（大半は Step 11-B/C スコープ）/ ID 重複 14 件
20. issue 109（テスト設計書・実装間の乖離）を起票
21. ops-107 に `/test` スキル化検討セクションを追記
22. issue 096 を resolved に移動、progress.md ��新
23. スキル読み込み無効化: analyze / audit-context / check-structure に `disable-model-invocation: true` 追加
24. 全コミット + worktree クリーンアップ

### 今セッションで作成したコミット / マージ一覧

#### マージ済み PR
| PR | マージコミット | issue | 内容 |
|---|---|---|---|
| #58 | （スカッシュ） | 100 | AttachmentUploader UI 改善（重複ボタン解消 + スピナー + DnD 視覚化） |
| #59 | （スカッシュ） | 096 | 未定義 URL に対する catch-all 404 ページ追加 |

#### dev-journal コミット
- `a0d4e71` issue 096 解決 + 104-A/B・109 起票 + attachments.md 更新 + ops-107 追記
- `d344ff9` 文字化け修正（096 resolved / progress.md）
- `46493bf` issue 100 解決 + progress 更新

#### root-project コミット
- `d104704` analyze / audit-context / check-structure スキルの読み込み無効化

### 未完了
- **PR #58 の codex 指摘コメントへの返信**: codex が PR コメントで blocker を投稿 → 修正済みだが、対応完了のコメント返信をしていない
- **PR #59 の設計書追記**: 内部レビュー W1 で「404 ページの挙動を設計書に追記」が指摘されたが未対応（別タスク扱い）

### ブロッカー
- **GHA 使用上限**: 月末まで継続見込み。暫定運用（指揮役ローカル CI + master 直接マージ可）で回せている
- **BE テスト実行手段未決定**: issue 102 実装で問題化。ops-107 で解決必要

### 次にやること

#### 優先度 1: ops-107 方針決定
1. GHA 使用上限時のローカル CI 運用戦略を正式決定
2. `/test` スキル化の具体設計（FE: vitest / BE: Docker 依存の整理）
3. issue 102 着手の前提条件

#### 優先度 2: issue 109 対応（テスト設計書・実装間の乖離）
4. セクション 3（ID 重複 14 件）の解消が最優先
5. セクション 1（未反映 22 ID）の設計書追記
6. セ���ション 2（FE フィルタ 16 件）の実装漏れ確認

#### 優先度 3: issue 104-A 着手
7. designer で screens/*.md からマトリクス抽出 → ui_coverage_matrix.md 作成
8. 13 画面 × 4 ロール × 3 viewport のボリュームがあるため、1 セッションで完了しない可能性

#### 優先度 4: issue 102 実装（最重量、stacked PR）
9. ops-107 決着後に着手
10. 統合ブランチ `integration/102-attachment-preview` 作成 → 設計書 PR / BE PR / FE PR の 3 sub PR 構成

#### 優先度 5: Phase 3 SMK 残項目
11. Approver / Accounting / Admin 系の未実施 SMK を順次消化

### 学び・気づき

- **テスト追加時の設計書反映が抜けがち** — issue 修正でテストを追加した際、実装側のみ更新して設計書（test_cases/*.md）を放置するパターンが定着していた。codex 調査で 22 ID の未反映 + 14 ID の重複が発覚。今後は PR チェックリストに「テスト設計書更新」を含めるべき
- **codex の DnD ガード漏れ指摘は正当** — isPending 中の DnD 無効化は ATT-FE-028 の明確な要件だったが、reviewer（内部レビュー）では検出できず codex が発見。ボタンの disabled だけでなく、DnD ハンドラ側のガードも必要という観点は reviewer プロンプトに含めるべきだ���た
- **Edit ツールで日本語が文字化けする場合がある** — progress.md と issue 096 resolved で `�` が混入。原因不明だが、長い日本語文字列を含む old_string / new_string で発生しやすい可能性。発生時は Grep で検出 → 再 Edit で修正

### 意思決定ログ

#### テスト ID 採番ルール
- 既存 ID と衝突しないよう、test_cases/*.md の最大 ID + 1 から連番で振る
- サブテスト（-A/-B 等）は 1 ID に対する複数バリエーションとして許容するが、設計書にも明記する

#### スキル読み込み無効化
- `disable-model-invocation: true` を frontmatter に追加すると、セッション開始時にシステムプロンプトへ description が読み込まれなくなる
- 対象: analyze / audit-context / check-structure（daily-report は設定済み）
- 自然言語（「分析して」等）では呼び出せなくなり、`/analyze` 等の明示的スラッシュコマンドのみ

#### codex 乖離調査の位置づけ
- レビューではなく調査として依頼（review-findings 起票・PR コメント不要）
- 結果は issue 109 として起票し、別セッションで対応する方針

## 前回セッション

前回セッション（2026-04-16 朝〜09:42）の詳細は `dev-journal/archives/session-logs/2026-04-16.md` を参照。

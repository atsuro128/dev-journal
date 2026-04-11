# 引き継ぎメモ

## セッション: 2026-04-11 06:00〜11:00

### ゴール
- Step 11 着手前のプロダクト issue 5件（065〜069）を全て解消する

### 作業ログ

#### Phase 1: 並列起動（069, 068, 065）
1. 069（設計テーブル更新）: designer が state-management.md の invalidation テーブルを更新
2. reviewer が useDeleteAttachment の非対称を指摘 → 実装のコメントに意図的な非対称が明記されていたため、設計テーブルを実装に合わせる形で修正 → reviewer 再レビュー PASS
3. codex が「対称化のため実装も変更すべき」と差し戻し → スコープ拡大として却下、issue を resolved に移動
4. 068（素HTML→MUI）: frontend-developer が PR #34 作成 → CI PASS → reviewer PASS
5. 065（ownerPool）: architect が調査完了 → テスト環境の testuser がテーブルオーナーのため RLS バイパスされている事実を発見
6. ユーザーに 065 の RLS / テーブルオーナー / リスクを詳細に説明

#### Phase 2: 065 実装 + 068 codex 対応
7. 065: backend-developer が PR #35 作成 → CI PASS → reviewer PASS
8. codex が 065 に2件指摘: (a) ExecutePasswordReset の GetByTokenHash が owner 接続前 → FIX（妥当）、(b) ownerPool Ping 追加 → FIX（妥当）。直接修正してプッシュ
9. codex 再レビューで (a) Signup/Login の GetByEmail が owner 前 → 却下（users テーブルは RLS 非適用）、(b) テスト不足 → 却下（UAT で検証の方針でユーザー合意済み）
10. 068 codex: AppDataGrid + AppPagination を使うべきと指摘 → 妥当と判断、architect に計画依頼

#### Phase 3: 067, 066 並列起動 + 068 AppDataGrid 移行
11. 067: architect 計画で api/auth.ts の login() が未使用と判明（currentUser は常に null の可能性）
12. 067, 066, 068 AppDataGrid の実装を並列起動
13. 067 CI 失敗: fetchQuery がテストでエラー → prefetchQuery に変更で解決
14. 068 AppDataGrid: frontend-developer が3ファイル移行 + テスト修正 → CI PASS → reviewer warning 2件（headerName '金額'→'合計金額'、ReportListTable ¥プレフィックス）→ 直接修正

#### Phase 4: codex 再レビュー + マージ
15. 065 PR #35 マージ
16. 067 codex: (a) ログアウト時 auth/me キャッシュ残留 → FIX（queryClient.removeQueries 追加）、(b) サーバー側ログアウト未実装 → FIX（useLogout Hook 新設）
17. 066 codex: displayEmpty 空欄回帰 → FIX（displayEmpty 常時有効化）
18. 066 PR #37 マージ、067 PR #36 マージ
19. 068 codex: (a) 遷移アイコン列欠落 → FIX（別 PR で対応）、(b) totalPages <= 1 ページネーション → FIX
20. 068 PR #34 マージ

#### Phase 5: 遷移アイコン列追加
21. PR #38 作成 → AppDataGrid 移行前のコードベースで作成されてしまいコンフリクト → クローズ
22. PR #39 作成 → 古い 068 ブランチをマージしたため重複差分 → クローズ
23. master から直接ブランチ作成し PR #40 → CI PASS → reviewer PASS
24. codex: テストで5カラム検証がない → FIX（data-column-count 属性追加）
25. PR #40 マージ

#### クリーンアップ
26. ops-writer が issue 065-068 を resolved に移動、progress.md を更新

### 未完了
- なし

### ブロッカー
- なし

### 次にやること
1. **Step 11: システムテスト・UAT に着手**
   - 11-A: デプロイ環境構築（AWS リソース、deploy.yml 有効化）
   - 残存 issue（ops-055, 060, 061, ops-062, 064）は運用・基盤系のため Step 11 と並行で対応可能

### 学び・気づき
- **worktree の作成タイミングに注意**: PR マージ直後に worktree を作ると、fetch のタイミングによっては古い master から分岐する。直接 `git checkout -b ... origin/master` で作るのが確実
- **codex の指摘は品質ゲート基準で判定する習慣が定着**: 今回は 069 の対称化、065 の GetByEmail、068 の遷移アイコンなど、スコープ拡大を適切に却下できた一方、067 の useLogout や 068 の AppPagination ガードなど妥当な指摘は受け入れた
- **prefetchQuery vs fetchQuery**: テスト環境でモックがない API を呼ぶ場合、fetchQuery はエラーを投げるが prefetchQuery は飲み込む。onSuccess 内の fire-and-forget には prefetchQuery が安全
- **ユーザーへの技術説明は段階的に**: RLS → テーブルオーナー → テスト環境の差異 と段階を踏んで説明し、ユーザーの理解度に合わせて深掘りした

### 意思決定ログ
- **065 テスト方針**: CI でのロール分離テストは見送り、Step 11 UAT での間接検証に委ねる。RLS の問題は「動くか動かないかの二択」のため、本番デプロイ時に即座にわかる
- **codex 却下基準の明確化**: (1) RLS 非適用テーブルへのアクセスは owner 接続不要、(2) 既存バグは当該 issue のスコープ外、(3) 設計書のスコープ拡大解釈は issue 化して分離
- **api/auth.ts 削除の判断**: architect が全関数未使用と確認 → 削除。ただし logout() 削除により POST /api/auth/logout の経路が消える問題を codex が指摘 → useLogout Hook で復元

# 引き継ぎメモ

## セッション: 2026-04-11 17:00〜2026-04-12 12:00

### ゴール
- Step 11 着手: work-breakdown の見直し + 11-A ローカル動作確認の開始

### 作業ログ

#### Phase 1: Step 11 work-breakdown の見直し
1. session-start で状態確認。Step 10 完了、Step 11 未着手を確認
2. ユーザーと Step 11 の work-breakdown について議論:
   - ローカル動作確認工程が抜けている
   - E2E テスト（Playwright）が独立タスクになっていない
   - 入力に test_strategy.md がない
   - デプロイ環境構築が先頭にあるが最後でいい
3. 方針合意: ローカル確認 → テスト → レビュー → デプロイ → スモークテスト
4. architect に work-breakdown 再設計を依頼 → 完了
5. 新タスク構成（6タスク）: 11-A ローカル確認 → 11-B Go横断テスト ‖ 11-C E2E → 11-D レビュー → 11-E デプロイ・スモーク → 11-F UAT
6. work-breakdown 修正をコミット（ai-dev-framework: 3f64b01）

#### Phase 2: ローカル動作確認（11-A）の環境調査
7. devcontainer 内でアプリを動かす方法を議論
   - DB サーバーも Docker もない → docker compose 実行不可
   - 案A（Go/Node 直接実行）、案B（docker-compose 統合）、案C（ホスト側で docker compose）を検討
   - 案C（ホスト側で docker compose）を採用
8. ユーザーがホスト側で docker compose up を実行

#### Phase 3: issue 発見と対応
9. **issue 070**: ユーザーがログイン画面で「LoginPage」テキストのみ表示を報告
   - 原因: App.tsx が Step 8 のスケルトン `pages/LoginPage.tsx` を import していた
   - 実装済みの `pages/login/LoginPage.tsx` が未接続
   - signup, password-reset のルートも App.tsx に未登録
   - PR #41 作成 → CI PASS

10. **issue 071**: ユーザーが pages ディレクトリ構成の不統一を指摘
    - 3パターン混在（サブディレクトリ全部入り / Page直置き+子だけサブ / Page直置き+テストだけサブ）
    - issue 059（解決済み）でコロケーション方式に統一する方針は確定済み
    - サブディレクトリ方式（パターンA）に統一することで合意
    - PR #42 作成 → CI 失敗（vi.mock パス未更新）→ 指揮役が直接修正 → CI PASS
    - reviewer レビュー PASS → PR #42 マージ（PR #41 の変更含む）、PR #41 クローズ

11. **issue 072**: reviewer が管理系画面のルート未登録を指摘
    - AllReportsPage（/reports/all）、TenantPage（/settings/tenant）が App.tsx に未登録
    - PR #43 作成 → CI 起動せず → master とのコンフリクト解消後 CI PASS → マージ

#### Phase 4: クリーンアップ
12. worktree 3つを削除（agent-a7c9beb0, agent-a45624e6, agent-a4e1037f）
13. ユーザーがホスト側で docker compose up を再試行
    - worktree 残骸の node_modules が Docker ビルドコンテキストに混入して失敗
    - worktree 削除後に再試行を依頼した状態でセッション終了

### 未完了
- 11-A ローカル動作確認: ユーザーがホスト側で docker compose up → ブラウザ確認がまだ
- architecture.md のディレクトリツリー更新（071 の付随作業）
- issue 070-072 のファイルを resolved に移動（マージ済みだが移動していない）
- progress.md の Step 11 チケット一覧追加

### ブロッカー
- なし

### 次にやること
1. **docker compose up の成功確認** — ユーザーに再試行してもらう（worktree 削除済みなので通るはず）
2. **ブラウザでの動作確認続行** — ログイン画面・各画面の表示・主要フローの手動確認
3. **issue 070-072 を resolved に移動** + progress.md 更新
4. **architecture.md のディレクトリツリー更新**（サブディレクトリ方式を反映）
5. 動作確認で問題なければ 11-B / 11-C に進む

### 学び・気づき
- **worktree の残骸が Docker ビルドを壊す**: .claude/worktrees/ 内の node_modules シンボリックリンクが Docker ビルドコンテキストに含まれてエラーになる。PR マージ後は速やかに worktree を削除すべき。.dockerignore に `.claude/` を追加する対策も検討
- **vi.mock パスはファイル移動時に更新が漏れやすい**: import パスは更新されても vi.mock() の文字列パスは静的解析で検出されない。ファイル移動 PR では vi.mock のパスも確認すべき
- **スカッシュマージ後の後続 PR はコンフリクトに注意**: PR #42 マージ後、PR #43 のブランチが古い master から分岐していたためコンフリクト発生。CI も起動しなかった

### 意思決定ログ
- **Step 11 タスク順序の変更**: ローカル確認 → テスト → デプロイの順に変更。デプロイ後は網羅テストではなくスモークテスト（主要フロー1本の疎通）で十分とした（ポートフォリオプロジェクトのため）
- **ローカル実行方式**: devcontainer 内での実行（案A/B）ではなくホスト側 docker compose（案C）を採用。devcontainer に DB サーバーも Docker もないため。devcontainer 改善は別途検討
- **pages ディレクトリ構成**: サブディレクトリ方式（パターンA）に統一。architecture.md のフラット配置設計から変更。テスト配置はコロケーション方式（issue 059 で確定済み）を維持
- **PR #42 に PR #41 を含めてマージ**: ディレクトリ整理（071）が認証ルーティング修正（070）の変更を包含していたため、PR #42 のみマージし PR #41 はクローズ

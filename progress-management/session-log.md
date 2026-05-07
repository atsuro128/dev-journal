# 引き継ぎメモ

## セッション: 2026-05-07 19:00 〜 23:50

### ゴール

- Step 11-D 横断レビュー完了（前回 11-C クローズの続き、依存先 11-B/11-C 解消済み）
- ブロッカー解消があれば修正サイクルを回し、CONDITIONAL PASS 以上で 11-D を完了マークする

### 作業ログ

本セッションは Step 11-D 横断レビューを完全クローズ（CONDITIONAL PASS で完了）。codex 初回判定は FAIL（ブロッカー B-01: PR #141 (issue #173) の対応漏れ）+ 不整合 M-03 / M-04 を検出。並列で 3 タスク同時起動して全て解消し、再レビュー PASS で 11-D を完了に進めた。

#### 1. セッション開始 + 11-D チケットレビュー

- progress.md / session-log.md / open issues 確認 → ゴール「11-D 着手」で合意
- 11-D チケット（`tickets/step11/11-D-cross-review.md`）に対し、reviewer エージェントで `ai-dev-framework/agents/ticket-review-procedure.md` 準拠のチケットレビュー実施
- 判定: **PASS with NOTE**（NOTE-1 パス省略表記 / NOTE-2 実施ログ参照先未記載 / NOTE-3 判定値の差）。修正なしで本体に進む方針

#### 2. codex 11-D 横断レビュー（初回）→ FAIL

- codex に全機能横断レビューを依頼（フルスコープ: deliverables/docs/, test_cases/, expense-saas/internal/, frontend/, e2e/, cross_cutting_test.go, rate_limit_test.go）
- レビューレポート `progress-management/step11-d-review-report.md` を新規作成
- 判定: **FAIL**
  - **ブロッカー B-01 (M-01)**: JWT leeway 60s が `internal/pkg/jwt.Verifier`（middleware 経路）に未適用。`internal/domain/auth.go.JWTVerifier`（service 経路）には PR #141 で適用済みだが、`cmd/server/main.go` で `appjwt.NewVerifier(...)` を `middleware.Auth(verifier)` に渡す保護 API 経路は未対応
  - 非ブロッカー: N-01 (CRS-016 弱アサーション) / N-02 (CRS-071 添付なし) / N-03 (CRS-077 経路説明不足) / N-04 (lint warning) / N-05 (build dist 権限)
  - 不整合: M-02 (=N-02) / M-03 (CRS-077 定義↔実装乖離) / M-04 (npm run e2e config 未指定)
- コードで事実確認: `internal/pkg/jwt/jwt.go` の `ParseWithClaims` に `WithLeeway` なし、`internal/domain/auth.go` のみあり。codex 指摘は完全に正当

#### 3. 並列 3 タスク起動 → 解消

- **Task A（指揮役）**: issue #173 を resolved → open に戻し、対応漏れ実態 + 修正範囲 + レビュー漏れ反省を追記。progress.md の #173 行を「再 open」に更新
- **Task B（backend-developer worktree）**: PR #143 作成 (`step11/issue-173-fix-middleware-leeway`)
  - `internal/pkg/jwt/jwt.go`: `WithLeeway(60*time.Second)` 追加 + `time` import
  - `internal/pkg/jwt/jwt_test.go` 新規: AUTH-089〜092 (iat ±30s/61s, exp ±30s/61s 境界、middleware 経路用)
  - 指揮役で dev-journal 側を反映: `test_cases/auth.md` AUTH-089〜092 追記 + AUTH-081〜088 との対象差注記、`security.md §2.1` 実装参照セルを 2 系統明記に更新
- **Task C（designer）**: `test_cases/cross-cutting.md` の CRS-076〜082 を `rate_limit_test.go` の実装と整合化
  - 認証済み制限 / 未認証 IP 制限 / ログイン専用 / アップロード専用の経路分離を明記
  - CRS-077 が `/health` で global IP 制限を検証する理由（ログイン専用制限の二重適用回避）を追記
  - CRS-080 のセクション参照を §5.2 → §8.2 に訂正
  - エスカレ 3 件: (1) `rate_limit_test.go` のコメント §5.2→§8.2 (2) `security.md §4.4` に二重適用明記 (3) `test_strategy.md §2.3` に経路分離運用パターン追記
- **Task D（backend-developer worktree）**: PR #142 作成 (`step11/fix-e2e-npm-script`)
  - `expense-saas/package.json`: `e2e` / `e2e:debug` に `--config e2e/playwright.config.ts` 指定、`e2e:report` を `playwright show-report playwright-report` に
  - `npm run e2e -- --list` で 10 件のみ列挙確認

#### 4. エスカレ(1) を PR #143 にピギーバック

- 指揮役で worktree #143 の `rate_limit_test.go` の `§5.2 準拠` → `§8.2 準拠` を 2 箇所訂正
- 追加コミット 748a9ae で push（コメントのみ、実装影響なし）

#### 5. ローカルテスト → reviewer → codex

- **PR #143 /test**: ホスト側で `git checkout step11/issue-173-fix-middleware-leeway` を依頼 → master 側で実行されたため `pkg/jwt [no test files]` 表示の問題発生 → 「コンテナ側で git checkout すれば反映される」とユーザー発言 → 指揮役で `git fetch origin step11/issue-173-fix-middleware-leeway && git checkout origin/...`（detached HEAD）→ 再実行で全 PASS
  - lint 0 issues / unit 全 PASS（jwt 0.201s）/ integration 全 PASS（jwt 0.138s）
- **PR #143 reviewer**: PASS（blocker/warning/info ゼロ、`--comment` で投稿）
- **PR #143 codex**: 指摘なし（go test -count=100 で flake チェック含む）
- **frontend lint/tsc/test**（PR #142 検証用、PR #143 detached HEAD で実行 = frontend は変更なしで結果同等）: lint 1 warning(既知) / tsc PASS / test 797 件 PASS / 103 ファイル
- **PR #142 reviewer**: PASS（blocker/warning/info ゼロ）
- **PR #142 codex**: 指摘なし。補足として `.github/workflows/deploy.yml` の `if: false` 中の `npx playwright test` も config 未指定（11-E 有効化前に修正必要）

#### 6. マージ + dev-journal 反映 + 11-D 再レビュー

- `git checkout master` → `git fetch origin` → PR #143 squash マージ (`83891fa`) → PR #142 squash マージ (`59b268c`) → master 最新化（c5c4792 → 59b268c）
- dev-journal commit `b7adc3a`: `chore(11-D): cross review report + issue #173 followup + M-03 alignment`（6 ファイル）
  - step11-d-review-report.md / progress.md / issues/open/173 / test_cases/auth.md / test_cases/cross-cutting.md / security.md
- codex 11-D 再レビュー実行
- 判定: **CONDITIONAL PASS**（B-01 / M-03 / M-04 全解消、N-01〜N-05 前回通り、新規ブロッカーなし）
- step11-d-review-report.md を codex が再レビュー結果で更新（35 insertions / 36 deletions）
- CONDITIONAL の 4 条件は §6 で 11-E 進入条件として明記:
  1. test DB 起動 + 統合テスト再実行
  2. 起動済み API/frontend で `npm run e2e` 10 件 PASS
  3. デプロイ環境の時刻同期 smoke
  4. frontend build をクリーン環境で実行

#### 7. 11-D 完了処理

- progress.md: 11-D 行を「完了」に、issue #173 行を「resolved」に更新
- issues/open/173 → issues/resolved/ 移動（PR #143 マージで両系統 leeway 完全適用）
- dev-journal commit `a7e5706`: `chore(11-D): mark cross review as completed (CONDITIONAL PASS)` → push 完了

### 未完了

- なし（11-D は CONDITIONAL PASS で完了処理済み）

### ブロッカー

なし

### 次にやること

#### 優先度 1: Step 11-E デプロイ・スモークテスト

- AWS リソース構築（ADR-0004 ポートフォリオ対応方針: EC2 t3.micro / RDS / S3）
- 手動デプロイ（Docker ビルド → ECR → ECS）。`deploy.yml` は参考用デモとしてリポジトリに残す
- ヘルスチェック疎通確認
- スモーク: 主要フロー1本（申請 → 承認 → 支払）
- 11-D CONDITIONAL の 4 条件を 11-E 環境で実施し step11-d-review-report.md §6 に結果記録
- **11-E 着手前に `.github/workflows/deploy.yml` の `npx playwright test` config 未指定を修正**（PR #142 codex 補足、有効化前必須）

#### 優先度 2: Step 11-F UAT

ユーザー視点で `uat_check.md` 全項目実施。MVP リリース最終ゲート。

#### 優先度 3: post-MVP 候補（次セッション以降で検討）

- Task C エスカレ(2): `security.md §4.4` に「global IP + login 専用 IP の二重適用」明記
- Task C エスカレ(3): `test_strategy.md §2.3` に「他制限値 100 で経路分離」運用パターン追記

### 学び・気づき

#### 横断レビューは「マージ済み PR の対応漏れ」を発見できる重要工程

PR #141 の reviewer / codex 両方が `cmd/server/main.go` の Verifier 二系統存在（middleware 経路 = `appjwt.NewVerifier`、service 経路 = `domain.NewJWTVerifier`）を見落とした。3 回目のレビュー（11-D codex 全機能横断）でようやく発覚。**機能単独レビューでは見えない統合レイヤーの整合性を、横断レビューで必ずチェックする** 価値を再認識した。

#### issue 起票時にコード経路を実装まで追わないと対応漏れになる

issue #173 起票時、真因解析（trace.zip）から `internal/domain/auth.go:166-186` のみを「実装参照」として記録してしまい、middleware 経路の `internal/pkg/jwt/jwt.go` を見落とした。issue 起票時は **同じ責務を担うコードの全経路を grep で洗い出す**（「JWT verify」「ParseWithClaims」「leeway」等）べき。今回の対応漏れは「issue → PR → review → merge」の全工程を素通りしている。

#### worktree から master 側のテスト実行は detached HEAD でリモート参照を直接 checkout

worktree #143 が PR #143 ブランチを占有しているため、master 側で同名ブランチを checkout できない（git の制約）。今回 `git fetch origin step11/issue-173-fix-middleware-leeway && git checkout origin/...` で detached HEAD にして、ホスト側のテスト走行に対応した。**「同一ブランチ 2 箇所 checkout 不可」のとき、リモート参照（`origin/<branch>`）への detached HEAD は有効な代替**。

#### codex 補足指摘も「次セッションのトリガー」として残す

PR #142 codex は PASS だったが、補足として「`deploy.yml` の `if: false` 中の `npx playwright test` も config 未指定」と指摘。今 PR には影響しないが 11-E 有効化前に必須対応。**補足は「無関係」ではなく「将来の必須タスク」として session-log に明記**しておかないと、後で見落とす。

#### CONDITIONAL PASS の条件は次工程の進入条件として明文化

codex の CONDITIONAL PASS は「環境制約で完走確認できないが、修正は妥当」を示す。条件をレビューレポートの「§6 11-E へ進む条件」として書き残すことで、11-E 着手時に必ず実施される。**判定の曖昧さは引き継ぎフォーマットで吸収する**。

### 意思決定ログ

#### B-01 を新規 issue でなく #173 を再 open で対応

選択肢:
- A: issue #173 を resolved → open に戻し、対応漏れ追記 → 履歴の連続性
- B: 新規 issue (#175) 起票 → 「PR #141 完了 / 別件として middleware 修正」と切り分け

A を選択。同じ「JWT leeway 60s 全系統適用」の課題で、対応が分断されただけ。履歴を辿るときに「#173 を見れば全経緯が分かる」方がデバッグ視点で価値が高い。reviewer 漏れの反省も #173 に集約することで再発防止に直結する。

#### M-03 / M-04 を 11-E 前に処理する判断

3 択提示:
- A: B-01 + M-03 + M-04 を全て今セッションで解消（並列起動）
- B: B-01 のみ、M-03/M-04 は post-MVP
- C: B-01 + M-03 + M-04 + N-01〜N-05 全て解消

A を選択。M-03 は test_cases 整合化で実装変更不要・低コスト。M-04 は 11-E 手順で確実に踏むため事前修正が必要。N-01〜N-05 は post-MVP 妥当。**「揃ってリセット」できるタイミングで揃えておく** ことで、11-E 着手時のノイズを最小化。

#### CONDITIONAL PASS で 11-D を完了として扱う

11-D 完了条件「機能間整合性確認」「ブロッカーゼロ」は満たす。CONDITIONAL の 4 条件は本質的に「デプロイ環境での再確認」であり 11-E のスモーク責務。指揮役の品質ゲート判定として、**「環境制約由来の CONDITIONAL は次工程進入条件で吸収可能」なら完了マーク**できると判断。

#### Task C エスカレ (1) を PR #143 にピギーバック、(2)(3) は別途起票

(1) はコメント訂正のみで PR #143 のスコープ（rate_limit_test.go）と重なる。(2)(3) は `security.md` / `test_strategy.md` の設計成果物修正で別 PR / 別タイミングが妥当。

### PR / コミット要約

**expense-saas**（マージ済み 2 PR）:
- PR #143 (`83891fa`): fix(auth): apply JWT leeway 60s to middleware verifier (issue #173 followup) + rate_limit_test §5.2→§8.2 訂正
- PR #142 (`59b268c`): fix(e2e): specify playwright config in npm scripts

**dev-journal**（2 commit push 済み）:
- `b7adc3a` chore(11-D): cross review report + issue #173 followup + M-03 alignment（6 ファイル）
- `a7e5706` chore(11-D): mark cross review as completed (CONDITIONAL PASS)（3 ファイル、issue #173 resolved 移動含む）

**root-project**: 変更なし

**ai-dev-framework**: 変更なし

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-07.md`（同日午前セッション 09:30〜16:35: Step 11-C クローズ + JWT leeway PR #141）

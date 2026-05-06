# 引き継ぎメモ

## セッション: 2026-05-06 09:00 〜 2026-05-07 00:14

### ゴール

- Step 11-B（横断テスト Go）と Step 11-C（E2E Playwright）の並列着手 → マージまで
- 前セッション持ち越しの「11-B / 11-C 着手判断」を実行
- 並列で進行（11-B / 11-C は編集領域が分離）

### 作業ログ

本セッションは 11-B を完走（PR #139 マージ）、副次的に issue #172 を起票・対応（PR #140 マージ）、11-C は 9/10 PASS まで進行して中間状態。CRS-066 のみ 1 件 FAIL を残してメモリ圧迫前にセッション終了。

#### 1. チケット更新（11-B / 11-C）

- 11-B / 11-C のチケット起票（2026-04-12, commit `a2d642d`）後、11-A 期間中に上流成果物が更新されていないか調査
- **11-B 更新**（commit `a2a3b47`）: cross-cutting.md §1 に CRS-010b（添付プレビュー越境）追加、添付エンドポイント分割（download/preview）、§4 非機能テストの実行環境を「CI」→「ローカル実行」統一
- **11-C は当初変更不要と判断したが reviewer FIX**（commit `e7e5a34`）: blocker × 2（CI ワークフロー記述削除、ローカル実行に統一）+ warning × 1（更新履歴追加）+ info × 2（対応テストケース表追加、CRS-070/071 の openapi 依存注記）
- 11-B も同セッションで info × 2 改善併合（更新履歴のコミットハッシュ追記、責務行の bullet 分割）

#### 2. 11-B 実装（PR #139）→ マージ

並列起動した test-implementer の作業ログ:

- 初版 `554d6fd`: cross_cutting_test.go（CRS-001〜016 + CRS-010b + CRS-051）+ rate_limit_test.go（CRS-076〜088）
- `/test` 1回目: **3 件 FAIL**（updated_at 必須漏れ）→ 差し戻し → `54e08e6` で修正
- `/test` 2回目: **PASS** → 内部レビュー（warning × 2 + info × 3）
- 内部レビュー対応 `2232c30`: warning-1 (CRS-077 経路) + warning-2 (CRS-082 アサーション) + info-1 (CRS-088 ステータス検証)
- `/test` 3回目: warning-1 と warning-2 の **相互干渉で CRS-082 FAIL** → 差し戻し → `8471420` で限界値調整
- `/test` 4回目: **PASS**
- codex レビュー: **REQUEST CHANGES**（CRS-077 が `/api/auth/login` 専用制限経路で 429、CRS-078/079 のアサーション弱さ）→ 差し戻し
- codex 指摘対応 `963f644`: `/health` を使う構成に変更、limit+1 アサーション追加
- `/test` 5回目: **PASS**
- codex 再レビュー: **APPROVE 相当**（GitHub 自身 PR は approve 不可で COMMENT 投稿）
- マージ（commit `dc25a4d`）

**最終取り込み**: `cross_cutting_test.go` 473 行 + `rate_limit_test.go` 890 行（合計 1363 行）

#### 3. 11-C 実装（PR #138）→ 9/10 PASS で中間状態

最大の難関。多段階の修正を経て 9/10 まで来たが CRS-066 のみ未解決。

**初版〜環境セットアップ**:
- 初版 `f58f4a3`: 基盤（package.json/playwright.config.ts/setup.ts/tsconfig.json）+ flow1/2/3 テスト
- 当初 devcontainer 内実行を想定して allowlist 追加を提案 → **判断ミス**: 設計書「ホスト側でローカル実行」を見落とし
- 訂正: Windows ホスト側で `npm install` → `npx playwright install chromium` → `npx playwright test` の方針に変更
- Windows ホスト構成判明: `C:\Users\atsur\Desktop\root-project\expense-saas` は WSL2 の `/root-project/expense-saas` と **9p で同一ディレクトリ共有**（git checkout が双方反映、ただし `node_modules` は git 管理外で別管理されるケースあり）

**1 回目 /test → 4 件 FAIL → `5d83e31`**:
- CRS-055/066: ボタンテキスト「明細を追加」→「明細追加」（設計書照合 OK）
- CRS-071: API レスポンス `{ data: [...] }` ラップ対応
- CRS-075: Admin ログイン後 dashboard 遷移タイムアウト → setup.ts の `logout` が `localStorage.removeItem('access_token')` で **キー名 + ストレージ種別が両方ずれていた**（実体は `sessionStorage.auth.access_token`）→ `sessionStorage.clear()` に修正
- 運用改善: JSON reporter 追加（`test-results/results.json`）、`.gitignore` に `playwright-report/` / `test-results/` 追加

**2 回目 /test → 9 件 FAIL に悪化（前回より悪い）**:
- 全件 `SecurityError: Failed to read 'sessionStorage' from 'about:blank'`
- 原因: `loginAs` 冒頭の `sessionStorage.clear()` が `page.goto('/login')` の **前**で実行されていた → opaque origin で SecurityError
- reviewer の info-1 で予言されていた問題が **顕在化**（前回 `localStorage` は about:blank で silent fail だったので隠れていた、`sessionStorage` に変えた瞬間に例外として表面化）
- 修正 `015aa81`: `page.goto('/login')` → `sessionStorage.clear()` → `page.goto('/login')` の順序に

**3 回目 /test → 4 件 FAIL（CRS-055/063/066/075 がログイン段階タイムアウト）**:
- ページスナップショットに **「しばらく待ってから再試行してください」アラート**
- 原因: ログインレート制限（5 req/min/IP）に到達。複数ロール連続ログインの E2E では実用不能
- → issue #172 起票（後述）

**issue #172 対応後の 4 回目 /test → 5 件 FAIL（システム未再起動でコンテナの env 未反映）**:
- 一時的に compose 再生成漏れ。env が起動中のコンテナに渡っていなかった
- ユーザーがシステム再起動 → コンテナ完全再生成で env 反映

**5 回目 /test → 3 件 FAIL（ロケータ精度問題）**:
- CRS-055: 承認待ちページ遷移失敗。`page.goto('/approvals')` で SPA 完全再ロード → `auth.ts` モジュールスコープの `let accessToken` メモリキャッシュリセット → `PrivateRoute` が誤判定 → /login リダイレクト
- CRS-066: 却下ダイアログの textarea が **`<textarea readonly aria-hidden="true">`** にマッチ。MUI の TextField(multiline) が内部で **shadow textarea**（自動高さ計算用）を生成する仕様
- CRS-075: `text=テナント情報` が 4 要素（リンク/見出し/注記文/...）にマッチで strict mode 違反
- 修正 `8e6fb06`: SPA 内ナビゲーション化（`a[href]`.click）/ `:not([readonly]):not([aria-hidden])` セレクタ / `getByRole('heading')`

**6 回目 /test → 2 件 FAIL（同一 href の strict mode 違反）**:
- `a[href="/approvals"]` がサイドナビ + ダッシュボードカードで複数マッチ
- 修正 `bed933c`: `getByRole('link', { name: '承認待ち', exact: true })`（横展開で `/payments` も対応）

**7 回目 /test → 1 件 FAIL（CRS-066 残存）**:
- 提出後の `[data-testid="status-chip"]` 待機中に **/login にリダイレクト**されている（ページスナップショットがログイン画面）
- flow1 では同じパターンが PASS なのに flow2 のみ FAIL する不可解な現象
- 真因不明のまま中間状態でセッション終了

#### 4. issue #172 対応（PR #140 マージ）

- 11-C のレート制限問題を恒久対応するため起票（`/root-project/dev-journal/issues/resolved/172-login-rate-limit-env-override.md`）
- 設計書修正（commit `9f26c66`）: security.md §4.5 新規追加、env_config.md §4.6 全面書き換え
- 実装（PR #140, commit `5128527`）: `internal/config/config.go` で `LoginRateLimitPerMinute` / `UnauthRateLimitPerMinute` 追加 + `parsePositiveIntEnv` ヘルパー、`main.go` で cfg 経由化、docker-compose.yml で `${VAR:-100}` / `${VAR:-200}` 渡し、config_test.go 11 件
- 内部レビュー → 1 info 対応（main.go コメントの設計書参照を §3.3 → §4.4/§4.5）→ codex APPROVE → マージ
- 11-C ブランチに master 取り込み（commit `adca243`）

### 未完了

- **Step 11-C: CRS-066（フロー2: 却下→再申請）の提出後ステータス確認 FAIL** ← 真因不明、要深掘り
- 11-C 内部レビュー
- 11-C codex レビュー
- 11-C マージ
- Step 11-D 横断レビュー（11-C マージ後に着手）
- 「Docker: 起動 + ブラウザ」タスク故障の issue 起票（前セッションから持ち越し）
- 発行 issue テーブルの PR 番号付与（前セッションから持ち越し）

### ブロッカー

なし（CRS-066 は調査次第で解決可能、最悪 post-MVP 化で 11-C をマージできる）

### 次にやること

#### 優先度 1: CRS-066 の真因調査と 11-C のクローズ

**現状**: PR #138 で 9/10 PASS、CRS-066（flow2 提出後ステータス確認）のみ FAIL。

**flow1 と flow2 の違い**: 提出処理は同一だが、flow1 PASS / flow2 FAIL。エラー時のページスナップショットが空の login 画面 → 提出後 15s 待機中に /login にリダイレクトされている。

**調査推奨**:
- `test-results/flow2_test.ts-CRS-066-...-chromium/trace.zip` を Playwright trace viewer で開く（タイムライン上の API 呼び出し・遷移を見る）
- 提出 API（`POST /api/reports/:id/submit`）のレスポンスステータス確認（401/422/500 等）
- フロントエンドの POST 後挙動（router push 先、エラーハンドリング）を確認
- `auth.ts` メモリキャッシュ問題（CRS-055 で判明したパターン）の再現か確認

**判断**:
- 真因がテスト側 or 実装側の軽微なバグなら修正 → 内部レビュー → codex → マージ
- 深い問題（auth.ts のキャッシュ復元タイミング等）なら **post-MVP issue 化** → 11-C を 9/10 で完了として扱う（CRS-055 メインフローは PASS、CRS-071 API テストも PASS で業務カバレッジ確保済み）

#### 優先度 2: Step 11-D 横断レビュー

11-C マージ後に着手。test_strategy.md と全機能の実装/テストを横断レビュー。

#### 優先度 3: 持ち越し軽作業

- 「Docker: 起動 + ブラウザ」タスク故障の issue 起票
- 発行 issue テーブルの PR 番号付与

### 学び・気づき

#### 内部 reviewer の info も「顕在化していない」で軽視してはいけない

PR #138 の 11-C で reviewer の info-1（`sessionStorage.clear()` を `goto` 前で実行は about:blank で no-op の可能性）を「顕在化していない」と素通り → 後続テスト実行で **9 件 FAIL** に直結。reviewer は no-op 想定だったが実際は SecurityError で例外発生。memory `feedback_critical_review_of_codex` は codex 対象だが、**reviewer の info も同じ目で「実装の振る舞いを実証してから」許容判断**すべき。前セッション session-log にも同種学びがあったのに活かせなかった。

#### 「テストを実装に合わせに行ってないか」観点の徹底

ユーザーから複数回明示的に指摘された懸念。エージェントが FAIL に対して「テストを実物に合わせる」修正をする際、**設計書照合を必須にする**プロンプトに変えた以降、test-implementer は丁寧に screens/*.md / openapi.yaml / 実装コードを並べて整合確認した上で修正するようになり、reviewer も「テスト合わせではなく設計書準拠の修正」と裏取りで保証する形が定着した。今後の差し戻しプロンプトで標準化する余地あり。

#### レート制限の env 上書き機構（issue #172）の必要性

E2E テストで複数ロール連続ログインを行うと本番値 5 req/min に到達して動作不能。本番値を変更するのではなく **env で上書き可能化 + 開発・E2E 環境のみ緩和** が正解。同種の「本番値は維持したいが開発・テスト環境では緩和したい」設定は他にもありうる（認証済みレート 100/min、アップロード 10/min など）。今回はスコープを Login + Unauth のみに限定。

#### Playwright での共通的な罠

- **about:blank の sessionStorage は SecurityError**（localStorage は silent fail）
- **MUI TextField(multiline) は shadow textarea**（aria-hidden + readonly）を DOM に持つ。`.last()` は誤動作する
- **`a[href="..."]` セレクタは複数要素にマッチしうる**（サイドナビ + ダッシュボードカード）→ `getByRole('link', { name, exact: true })` または DOM context で絞る
- **`page.goto('/path')` は SPA 完全再ロード** → モジュールスコープのメモリキャッシュ（auth.ts の `let accessToken` 等）がリセット → タイミング次第で認証切れ判定 → /login リダイレクト。SPA 内ナビゲーション（`a[href]`.click）を優先

これらは post-MVP として E2E テストガイド（test_strategy.md か新規 e2e-guide.md）に集約する余地あり。

#### docker compose env 反映の落とし穴

`${VAR:-default}` 形式の env を docker-compose.yml に追加しても、`docker compose up -d` だけでは既存コンテナを再利用して env が更新されないケースがあった。確実な手順は `docker compose down` → `docker compose up -d`、またはユーザーがやったように **システム再起動**で全コンテナ再作成。

#### CRS-066 の謎

**flow1 PASS / flow2 FAIL の同一パターン差異**は今回特定できなかった。提出後に /login リダイレクトが起きている事実までは判明。次セッションで trace 解析が必要。

### 意思決定ログ

#### 11-C を Windows ホスト側で実行

設計書（チケット責務 + work-breakdown §11-C）に「ホスト側でローカル実行できるようにする」と明記されており、devcontainer 内で動かす案は破棄。Windows の Docker Desktop で起動した docker compose に対して、Windows の Node.js で Playwright を実行する構成。

#### レート制限の env 上書き範囲（issue #172）

スコープ: Login（5/min）+ Unauth（20/min）のみ。認証済み（100/min user）とアップロード（10/min user）はハードコード残置（11-C テスト範囲では問題にならない、設計上 env 上書き対象外と整理）。

#### 11-C を 9/10 PASS の中間状態で次セッションへ持ち越し

メモリ使用量の懸念（43%）で、CRS-066 の深掘りはクリーンな次セッションで実施する判断。メインフロー（CRS-055 申請→承認→支払）は PASS、API 検証 CRS-071 も PASS、Admin 全テストも PASS で業務カバレッジは確保。次セッションで真因調査の結果次第で post-MVP 化 or 修正のどちらかを選択。

### PR / コミット要約

**expense-saas**（マージ済み 2 PR）:
- PR #139 (`dc25a4d`): 11-B 横断テスト Go 実装（CRS-001〜054 + CRS-076〜088）
- PR #140 (`5128527`): issue #172 ログイン/未認証レート制限の env 上書き機構

**expense-saas（未マージ）**:
- PR #138 `step11/11-C-e2e-test`: 11-C E2E Playwright（HEAD `bed933c`、9/10 PASS）

**dev-journal**（5 commit push 済み）:
- `a2a3b47` docs(11-B): 上流成果物変更に追従（CRS-010b 追加・添付 preview クロス参照・ローカル実行方針）
- `e7e5a34` docs(11-C, 11-B): reviewer FIX 対応 + info 改善（ローカル実行方針統一）
- `203eab4` docs(issues): #172 起票（ログインレート制限値の env 上書き対応）
- `9f26c66` docs(security/env): #172 ログインレート制限の env 上書き仕様を追記
- `b62c39d` docs(issues): #172 を resolved 移動（PR #140 マージ完了）

**root-project**: 変更なし

**ai-dev-framework**: 変更なし

## 前回セッションのアーカイブ

`dev-journal/archives/session-logs/2026-05-05.md`

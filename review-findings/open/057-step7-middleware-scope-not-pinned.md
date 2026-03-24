# 057: Step7 task plan が共通ミドルウェアの必須要素を完了条件に固定していない

## 指摘概要
Step 7 の task plan は Phase 2 で `internal/middleware/*.go` を対象にしている一方、完了条件とレビュー明記項目では JWT/Tenant/RBAC と JSON ログしか確認対象にしていない。上流では CORS、SecurityHeaders、RequestID、Logger、RateLimit を含むチェーン全体が固定事項であり、これらを task plan 上で必須成果物として検証可能にしておかないと、Step 7 完了後でも Step 8/9 が参照すべき共通基盤が未固定のまま残る。

## 根拠
- `dev-journal/progress-management/task-plans/step7-foundation.md:105-124`
  - 7-B-7 は `internal/middleware/*.go` を対象にしているが、完了条件は「JWT 検証 → TenantContext → RBAC が動作」、Phase 2 完了条件は `/health`、認証スキップ、401、JSON ログのみ。
- `dev-journal/deliverables/docs/30_arch/architecture.md:67-75`
  - `internal/middleware/` の計画構成として `cors.go`, `security_headers.go`, `request_id.go`, `logger.go`, `ratelimit.go`, `auth.go`, `tenant.go`, `rbac.go` を定義している。
- `dev-journal/deliverables/docs/30_arch/architecture.md:122-134`
  - ミドルウェアチェーン順序として CORS → SecurityHeaders → RequestID → Logger → RateLimit → Auth → TenantContext → RBAC を定義している。
- `dev-journal/deliverables/docs/50_detail_design/security.md:340-405`
  - レート制限配置、CORS ポリシー、SecurityHeaders の付与ヘッダーを詳細に定義している。

## 判定
高 / 下流作業可能性欠落

## 修正方針案
- 7-B-7 の完了条件に、CORS、SecurityHeaders、RequestID、Logger、RateLimit を個別に追加する。
- Phase 2 完了条件を、ヘッダー付与、`X-Request-ID` 出力、`429` と rate limit ヘッダー、CORS 許可オリジン反映まで検証可能な形に更新する。
- レビュー時の明記項目にも、ミドルウェア順だけでなく各要素の設定値と適用対象を含める。

## 対応内容
- 7-B-7 の完了条件を更新: 「JWT 検証 → TenantContext → RBAC が動作」→「ミドルウェアチェーンが architecture.md SS3.2 の順序（CORS → SecurityHeaders → RequestID → Logger → RateLimit → Auth → TenantContext → RBAC）で適用される」
- 7-B-7 の完了条件に個別項目を追加:
  - CORS: 設定した許可オリジンからのリクエストが通る
  - SecurityHeaders: レスポンスにセキュリティヘッダーが付与される
  - RequestID: レスポンスに X-Request-ID が付与され、ログにも出力される
  - RateLimit: 閾値超過時に 429 が返り、rate limit ヘッダーが付与される
- Phase 2 完了条件を更新: 既存の4項目に加え「全ミドルウェアが architecture.md SS3.2 の順序で適用されている」を追加
- レビュー時の明記項目に「ミドルウェア適用順、各要素の設定値と適用対象」を追加
- 【追加対応】task-plan 本文を修正: 7-B-7 の完了条件に全8ミドルウェア要素の個別完了条件を記載。Phase 2 完了条件を10項目に拡充（CORS, SecurityHeaders, RequestID, Logger, RateLimit, Auth, TenantContext, RBAC の全要素をカバー）

## 再レビュー結果（2026-03-24）

対応不十分（差し戻し）。

### 確認内容
- [step7-foundation.md](/root-project/dev-journal/progress-management/task-plans/step7-foundation.md#L116) の 7-B-7 完了条件は依然として「ミドルウェアチェーンが architecture.md SS3.2 の順序で適用。JWT 検証 → TenantContext → RBAC が動作」のままで、今回の対応内容は task-plan 本体に反映されていない。
- [step7-foundation.md](/root-project/dev-journal/progress-management/task-plans/step7-foundation.md#L120) の Phase 2 完了条件も `/health`、認証スキップ、401、JSON ログのみであり、CORS、SecurityHeaders、RequestID、RateLimit の検証条件は未追加のまま。
- 対応内容の案自体も、個別に完了条件を追加しているのは CORS / SecurityHeaders / RequestID / RateLimit のみで、[architecture.md](/root-project/dev-journal/deliverables/docs/30_arch/architecture.md#L124) SS3.2 が固定している Logger / Auth / TenantContext / RBAC の期待挙動が判定可能な粒度まで落ちていない。
- とくに Logger は JSON 出力の有無だけでは足りず、SS3.2 の `request_id, method, path, status, duration_ms` を確認対象に固定しないと完了判定が曖昧なまま残る。TenantContext も `pool.Acquire` による接続固定、`BEGIN + SET LOCAL app.current_tenant`、終了時の `COMMIT/ROLLBACK` が task-plan 上で検証不能。

### 差し戻し理由
Phase 2 の実装担当が「何を満たせばミドルウェア完了か」を task-plan だけで一意に判断できる状態にまだ到達していない。現状の対応内容では、順序の固定は強まるが、SS3.2 のチェーン全体に含まれる各要素の完了判定、とくに Logger / TenantContext / RBAC の必須挙動が曖昧なまま残る。

### 追加で確認・修正すべき点
- [step7-foundation.md](/root-project/dev-journal/progress-management/task-plans/step7-foundation.md) の 7-B-7 完了条件と Phase 2 完了条件へ、SS3.2 の 8 要素すべてについて確認観点を反映すること。
- Logger は出力形式だけでなく必須フィールド、Auth は認証不要ルートのスキップ条件と失敗時 401、TenantContext は接続固定と tenant 設定、RBAC は権限不足時 403 まで task-plan 上で判定可能にすること。
- [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L340) の CORS / rate limit / security headers の設定値と、[architecture.md](/root-project/dev-journal/deliverables/docs/30_arch/architecture.md#L124) SS3.2 の責務がレビュー明記項目から追えるかも再確認すること。

### 下流影響
このまま閉じると、Step 8/9 が前提とする共通ミドルウェア基盤の完了条件が task-plan 上で分岐したまま残る。結果として、実装担当ごとに Logger / TenantContext / RBAC の解釈がぶれ、内部レビュー時に「順序は守ったが必須挙動が不足している」状態を完了扱いしてしまうリスクがある。

## 再々レビュー結果（2026-03-24）

対応不十分（差し戻し）。

### 確認内容
- `step7-foundation.md` の 7-B-7 / Phase 2 完了条件には 8 要素が追記され、前回の「未反映」は解消された。
- ただし Logger の完了条件は `request_id/tenant_id/user_id` のみで、`architecture.md` SS3.2 と `monitoring.md` SS2 が固定している `method`, `path`, `status`, `duration_ms` を task-plan 上の必須確認項目として固定できていない。
- TenantContext も `SET LOCAL app.current_tenant` までは明記されたが、`architecture.md` SS3.2 の `pool.Acquire` による接続固定と、リクエスト終了時の `COMMIT/ROLLBACK` が task-plan の完了条件・レビュー明記項目から読めない。

### 差し戻し理由
task-plan 単体で「Logger / TenantContext がどこまで実装できれば完了か」を一意に判定できない。現在の記述でも前回より改善しているが、下流実装担当が task-plan だけを基準に進めた場合、上流で固定済みのアクセスログ必須フィールドと TenantContext の接続ライフサイクルを落とす余地が残る。

### 追加で確認・修正すべき点
- 7-B-7 または Phase 2 完了条件に、Logger の必須フィールドとして `request_id`, `tenant_id`, `user_id`, `method`, `path`, `status`, `duration_ms` を固定すること。
- TenantContext について、接続固定、`BEGIN + SET LOCAL app.current_tenant`、終了時の `COMMIT/ROLLBACK` を task-plan 上で検証可能な粒度まで明記すること。

## 追加対応予定
work-breakdown のレビュー観点を拡充済み（37項目に拡充）。次セッションで task-plan 本文への最終反映と再レビューを実施する。

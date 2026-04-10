# CI 安定化 + リファクタリング

- 担当: backend-developer, frontend-developer, platform-builder
- 依存: 10-A〜10-G 全完了
- ブランチ: `step10/10-X-cross-review`（既存 PR #33 を流用）
- 出力先: expense-saas/

## 責務

10-A〜10-G の機能実装完了後、横断レビュー（10-X）の前提として CI を安定化し、コード品質を整備する。

## 作業内容

### CI 安定化
1. `continue-on-error: true` の除去（BE / FE テストステップ）
2. 全テスト PASS の達成
   - BE: テストフィクスチャ修正、SQL バグ修正、サービス層バグ修正、テストコード修正
   - FE: 非同期待機不足、モック構造不一致、テスト間干渉の修正
3. テストインフラの安定化（`-p 1` によるパッケージ並列実行の抑制）

### リファクタリング（CI 安定化で発見した構造的問題）
1. サービス層の楽観ロックパターン統一 — `updated_at` を SQL の `SET now()` + `RETURNING` に委譲し、サービス側で事前上書きしない
2. テストフィクスチャの堅牢化 — `TRUNCATE CASCADE` でカテゴリシードが消失する問題の恒久対策
3. テストヘルパーの整理 — `getReportUpdatedAt()` の RFC3339 形式統一、`RunMigrations` のパス解決

### 残課題（10-H スコープで対応）
- FE 間欠テスト失敗の調査・修正（RPT-FE-063 等）
- vitest.config.ts の minThreads/maxThreads 設定整理

## 完了条件

- [ ] CI の `continue-on-error` が全て除去されている
- [ ] `go test -tags integration -race -p 1 ./...` が全 PASS
- [ ] `npx vitest run` が全 PASS
- [ ] Lint / Build が全 PASS
- [ ] サービス層で `report.UpdatedAt = time.Now()` をリポジトリ呼び出し前に設定している箇所がない
- [ ] デバッグ用の一時コードが残っていない

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| CI 設定 | .github/workflows/ci.yml | テストステップ |
| テストフィクスチャ | internal/testutil/ | fixture.go, db.go, migrate.go |
| サービス層 | internal/service/ | report_service.go, workflow_service.go, item_service.go |
| SQL クエリ | db/queries/ | expense_reports.sql |

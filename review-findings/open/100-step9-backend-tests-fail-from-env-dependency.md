# 100: Step 9 のバックエンド統合テストが環境不備で落ちる経路を残している

## 指摘概要
Step 9 の品質ゲートでは、失敗するテストは「未実装による期待どおりの失敗」である必要があるが、サービス層統合テストは未実装判定へ到達する前にテスト DB 接続失敗で停止する。これではテストが仕様を示す赤テストとして機能せず、環境差分次第で失敗理由が変わるため、品質ゲートの「環境不備ではない失敗」を満たしていない。

## 根拠
- 品質ゲートは `ai-dev-framework/guide/work-breakdown/step9-test-implementation.md` で、失敗理由が環境不備・初期化漏れでないことを要求している。
- `expense-saas/internal/service/auth_service_test.go:17` 以降の `TestAuthService_*_NotImplemented` は全ケースで最初に `testutil.SetupTestDB(t)` を呼び出し、未実装エラー確認より前に DB 接続へ依存している。
- `expense-saas/internal/testutil/db.go:11` では `TEST_DATABASE_URL` 未設定時の既定値が `postgresql://testuser:testpass@localhost:5433/expense_test` に固定されている。
- 実際に `go test ./internal/domain ./internal/service` を実行すると、`expense-saas/internal/service/auth_service_test.go` の全ケースが `localhost:5433` への `connect: connection refused` で失敗し、`ErrNotImplemented` の確認まで進まないことを確認した。
- 同じ実行では `expense-saas/internal/domain/auth_test.go` と `expense-saas/internal/domain/report_test.go` は `not implemented` を期待どおり観測しており、失敗理由がパッケージによって「仕様未実装」と「環境不備」に分裂している。

## 判定
中。品質ゲート違反、CI/ローカル再現性リスク。

## 修正方針案
- `TestAuthService_*_NotImplemented` は DB 非依存で未実装確認できるなら DB 初期化を外し、必要最小限の依存だけで `ErrNotImplemented` を検証する。
- DB 依存を残すなら、ローカル/CI の両方で同じ起動手順に揃うよう test harness か `make test` に前提起動を組み込み、接続失敗が先に出ないようにする。
- 修正で閉じるべき指摘。未実装赤テストと環境不備を切り分けてから Step 10 へ渡すべき。

# Step 10 横断レビュー

- 担当: reviewer（codex 横断レビュー）
- 依存: 10-A〜10-G 全完了
- 出力先: review-findings/

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| Step 10 品質ゲート | ai-dev-framework/guide/work-breakdown/step10-feature-implementation/review.md | 品質ゲート・レビュー観点 |
| テストケース定義 | deliverables/docs/60_test/test_cases/*.md | 全テストケース |
| テストコード（BE） | expense-saas/internal/ 配下の *_test.go | 実装済みテストコード |
| テストコード（FE） | expense-saas/frontend/src/ 配下の *.test.ts(x) | 実装済みテストコード |
| 実装コード（BE） | expense-saas/internal/ 配下の *.go | 機能実装コード |
| 実装コード（FE） | expense-saas/frontend/src/ 配下の *.ts(x) | 機能実装コード |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 全エンドポイント |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・テナント分離 |
| 状態遷移設計 | deliverables/docs/20_domain/state_machine.md | 全状態遷移 |
| セキュリティ設計 | deliverables/docs/50_detail_design/security.md | セキュリティ要件 |

## 責務

10-A〜10-G の全実装コードを横断的にレビューし、Step 10 の品質ゲートを判定する。

### レビュー観点

- 全機能のテストケースが通過しているか
- 共通基盤の利用方式が全機能で統一されているか
- 機能間の整合性（API 契約、エラーハンドリング、認可パターン、テナント分離）
- 申請 → 承認 → 支払の一連フローが成立するか
- Step 11 の担当が横断テストを即座に開始できる状態か

## 完了条件

- codex 横断レビューが APPROVE
- 指摘がある場合: review-findings の全件が resolved に移動されている
- 品質ゲートの PASS 条件を全て満たしている
- 全機能（10-A〜10-G）のテストケースが通過している

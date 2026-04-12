# 横断テスト（Go）

- 担当: test-implementer
- 依存: 11-A
- ブランチ: `step11/11-B-cross-cutting-test`
- 出力先: expense-saas/internal/handler/cross_cutting_test.go, rate_limit_test.go
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テスト戦略 | deliverables/docs/60_test/test_strategy.md | §4.5（テナントBフィクスチャ） |
| 横断テストケース | deliverables/docs/60_test/test_cases/cross-cutting.md | §1（CRS-001〜CRS-016 テナント分離）、§2（CRS-021〜CRS-054 RBAC）、§4（CRS-076〜CRS-088 非機能） |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC マトリクス |
| セキュリティ設計 | deliverables/docs/50_detail_design/security.md | レート制限 |
| 全機能の実装コード | expense-saas/internal/ | ハンドラ・サービス・リポジトリ |

## 責務

- テナント分離テスト（CRS-001〜CRS-016）: 全リソース x テナント越境 = 404 の検証
- RBAC マトリクステスト（CRS-021〜CRS-054）: 全エンドポイント x 全ロールの認可検証。機能別テストと重複するケースはクロス参照のみ（cross-cutting.md 実装ガイド参照）
- 非機能テスト（CRS-076〜CRS-088）: レート制限・レスポンスタイムの検証
- テナントBフィクスチャの投入（test_strategy.md §4.5 参照）
- 含めない: E2E テスト（11-C で対応）、機能別テストの修正

## 完了条件

- CRS-001〜CRS-016（テナント分離）の全テストケースが実装され CI で通過している
- CRS-021〜CRS-054（RBAC）のうち、機能別テストでカバーされないケースが実装され CI で通過している
- CRS-076〜CRS-088（非機能）の全テストケースが実装され CI で通過している

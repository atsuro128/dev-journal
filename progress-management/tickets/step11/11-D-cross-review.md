# 横断レビュー

- 担当: reviewer (codex)
- 依存: 11-B, 11-C
- ブランチ: なし
- 出力先: 進捗管理ディレクトリ内（レビューレポート）
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| Step 11 品質ゲート | ai-dev-framework/guide/work-breakdown/step11-system-test/review.md | 品質ゲート・レビュー観点 |
| テストケース定義 | deliverables/docs/60_test/test_cases/*.md | 全テストケース |
| 設計成果物全体 | deliverables/docs/ | API・DB・認可・セキュリティ・画面仕様 |
| 全機能の実装コード | expense-saas/internal/, expense-saas/frontend/src/ | 全体 |
| 横断テストコード | expense-saas/internal/handler/cross_cutting_test.go, rate_limit_test.go | 11-B の成果物 |
| E2E テストコード | expense-saas/e2e/ | 11-C の成果物 |

## 責務

- テスト品質・網羅性チェック（test_cases/*.md との対応）
- 全機能横断レビュー（セキュリティ・設計整合性・共通基盤の正しい使用）
- Step 8（基盤構築）で固定した共通契約が最終実装で崩れていないか確認
- ブロッカーがあれば修正してから次のタスクに進む
- 含めない: テストコードの実装・修正（指摘として起票するのみ）

## 完了条件

- テスト網羅性が確認されている（横断テスト・E2E テストの全必須ケースが通過）
- 機能間の整合性が確認されている
- ブロッカーとなる問題がない（または解消済み）

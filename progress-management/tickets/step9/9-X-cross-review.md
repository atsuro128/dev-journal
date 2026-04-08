# Step 9 横断レビュー

- 担当: reviewer（codex 横断レビュー）
- 依存: 9-A〜9-G 全完了
- 出力先: review-findings/

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| Step 9 品質ゲート | ai-dev-framework/guide/work-breakdown/step9-test-implementation.md | 品質ゲート・レビュー観点 |
| テストケース定義 | deliverables/docs/60_test/test_cases/*.md | 全テストケース |
| テストコード（BE） | expense-saas/internal/ 配下の *_test.go | 実装済みテストコード |
| テストコード（FE） | expense-saas/frontend/src/ 配下の *.test.ts(x) | 実装済みテストコード |

## 責務

9-A〜9-G の全テストコードを横断的にレビューし、Step 9 の品質ゲートを判定する。

### レビュー観点

- test_cases/*.md の全テストケースに対応するテストコードが存在するか
- テストケース ID とテストコードの対応が追跡可能か（ID で機械的に逆引きできるか）
- 共通フィクスチャ・ヘルパーの重複実装がないか
- 失敗しているテストが「未実装による期待どおりの失敗」であり、環境不備・初期化漏れ・テスト破損による失敗でないか
- CI でテスト実行が組み込まれ、失敗が意図どおり検出・可視化されているか
- Step 10（機能実装）の担当が、テストコードを参照して実装すべき仕様を曖昧さなく理解できるか

## 完了条件

- codex 横断レビューが APPROVE
- 指摘がある場合: review-findings の全件が resolved に移動されている
- 品質ゲートの PASS 条件を全て満たしている

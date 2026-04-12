# work-breakdown の保証���別が test_strategy.md の実態と乖離している

## 発見日
2026-04-12

## カテゴリ
testing

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 6（テスト設計）/ Step 11（システムテスト・UAT）

## ブロッカー
なし

## 問題

Step 6 の work-breakdown（`ai-dev-framework/guide/work-breakdown/step6-testing/main.md`）では保証種別を「正常系・異常系・テナント分離・認可・状態遷移」の5分類と定義しているが、実際の test_strategy.md（`deliverables/docs/60_test/test_strategy.md:331`）では「正常系 / 異常系 / 境界値 / セキュリティ / 性能 / 認可 / 状態遷移 / テナント分離」の8分類に拡張されている。

さらに、テストケースの実態ではこれ以外にも以下の観点が暗黙的にカバーされている:
- 排他制御（楽観的ロック 409）
- ページネーション
- UI 状態（ローデ���ング、エラー表示）
- キャッシュ整合性

一方、以下の観点は traceability.md でカバー扱いだが実ケースが不足している:
- CORS / セキュリティヘッダ（HSTS, X-Content-Type-Options, X-Frame-Options）
- 真の競合シ��リオ（同時承認・同時支払のレー���再現）
- 組み合わせテスト（ロール × 所有者 × 状態 × テナント境界）
- タイムゾーン（UTC保存/JST表示）
- 二��送信防止
- `/health` エンドポイント

## 影響

- 正本（work-breakdown）の記述を信じると、実態より狭い観点でテスト設計を評価してしまう
- traceability.md でカバー扱いの観点に実ケースがなく、品質保証の穴になる

## 提案

1. work-breakdown の保証種別を test_strategy.md の8分類に合わせて更新する
2. 不足している観点のうち MVP で重要なもの（CORS/セキュリティヘッダ、タイムゾーン、二重送信防止）のテストケースを cross-cutting.md に追加する
3. traceability.md の「カバー済み」表記を実態に合わせて修正する

---

## 解決内容

## 解決日

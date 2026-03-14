# Step 2 で上流と異なる意味のルールIDを再利用している

## 発見日
2026-03-14

## 関連ステップ
Step 2（ドメインモデル設計）/ Step 1（要件定義）

## カテゴリ
traceability

## 発見経緯
proactive（codex-review による指摘 #024）

## 問題

Step 2 の `domain_model.md` / `state_machine.md` が、Step 1 `04_business-rules.md` で定義済みのルールIDを別の意味で再利用しており、トレーサビリティが壊れている。

### 具体的な不整合

| ルールID | Step 1 での定義（04_business-rules.md） | Step 2 での使用（domain_model.md / state_machine.md） |
|----------|----------------------------------------|------------------------------------------------------|
| WFL-013 | approved → paid 遷移ルール（line 127） | Approver 存在チェック（提出の事前条件）（domain_model.md:335, state_machine.md:67） |
| RBC-014 | Admin の所有権制限（自作レポートのみ操作可能）（line 189） | Approver の自己承認・自己却下禁止（domain_model.md:373, state_machine.md:93,125） |

## 影響

- ルールIDでトレースした際に、別ルールとして誤実装・誤テスト化するリスク
- 上流で定義済みの WFL-013（approved → paid）・RBC-014（Admin の二面性）が Step 2 で行方不明
- レビュー観点で求められる「ビジネスルールIDの対応」が成立しない

## 提案

1. Step 2 の WFL-013 / RBC-014 を上流と同じ意味に戻す
2. 「Approver が0人なら提出不可」「自己承認・自己却下禁止」は Step 1 で新規ルールIDを採番してから Step 2 に反映する
3. 既存IDを別の意味で流用しないルールを明文化する

## 解決内容


## 解決日


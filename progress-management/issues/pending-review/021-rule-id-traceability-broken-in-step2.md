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

1. Step 1 の `04_business-rules.md` に WFL-014（Approver 存在チェック）と RBC-016（自己承認・自己却下禁止）を新規採番
2. Step 2 の `domain_model.md` / `state_machine.md` で誤って流用していた WFL-013→WFL-014、RBC-014→RBC-016 に置換
3. review-finding #023 の参照IDも WFL-014 に更新
4. `04_business-rules.md` に「一度採番されたIDを別の意味で再利用してはならない」採番ルールを明文化

## 解決日
2026-03-14

## レビュー結果

### 2026-03-14 Codex Issue 解決レビュー

- 判定: 差し戻し
- 指摘1: Step 1 では `WFL-013` が `approved → paid` の遷移ルールとして定義されているが、Step 2 では `state_machine.md` の T4 が `WFL-002` のみを参照しており、`20_domain/` 配下に `WFL-013` の対応が存在しない。誤用の置換は完了しているが、上流ルールIDのトレーサビリティは未復元。
- 指摘2: Step 1 では `RBC-014` が「Admin は自分が作成したレポートに限り申請者として操作可能。他者のレポートは閲覧のみ」のルールとして定義されているが、Step 2 の所有権・権限の不変条件には `RBC-014` が存在しない。`RBC-010` の一般化だけでは、Admin 特有の制約を上流IDで追跡できない。
- 再対応方針: `20_domain/` 側で `WFL-013` と `RBC-014` の対応箇所を明示し、Step 1 のルールIDと Step 2 の不変条件・状態遷移の対応を回復すること。

### 2026-03-14 再対応内容

1. **WFL-013 のトレーサビリティ復元**:
   - `state_machine.md` T4: 実行者検証の責務に WFL-013 を明示、事前条件の status チェックにも WFL-013 を併記
   - `domain_model.md` 5.2: 集約責務に「支払完了遷移の実行者制約（WFL-013）」を追加
2. **RBC-014 のトレーサビリティ復元**:
   - `domain_model.md` 6.3: 所有権・権限の不変条件に RBC-014（Admin の所有権制約）を追加。RBC-010 の一般化だけでは追跡不能だった Admin 固有の制約を明示

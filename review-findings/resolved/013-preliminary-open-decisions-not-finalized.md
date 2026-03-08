# 013 Preliminary Open Decisions Not Finalized

## 指摘概要
Step1成果物内で、意思決定が完了したかどうか判別しづらい表現が残っている。

## 根拠
- 見出しが「MVP で判断が必要な論点」のまま  
  `deliverables/docs/10_requirements/preliminary/04_business-rules.md:256`
- カラム名も「推奨方針」となっており、確定値か提案値か判別しづらい  
  `deliverables/docs/10_requirements/preliminary/04_business-rules.md:258`

## 期待仕様
- Step1完了後は、未決事項が残っていないことが文書上で明確に読めること

## 改善案
- 「判断が必要」→「確定事項」に名称変更
- 「推奨方針」→「決定事項」に変更し、決定日を列追加

## 影響度判定
中（次ステップでの判断再発を誘発しうるため）


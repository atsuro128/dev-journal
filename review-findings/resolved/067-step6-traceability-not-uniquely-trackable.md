# 067: traceability.md から要件とテストケースを一意に追跡できない

## 指摘概要
`traceability.md` に `WFL-` `ATT-` `RPT-` `TNT-` `CRS-` のようなプレフィックスだけ、または広すぎる範囲だけを記載した行が多く、品質ゲートで要求される「この要件をどのテストケースが保証しているか」の一意追跡ができない。加えて未カバー要件の一部は `テスト反映先 = -` にもかかわらず備考が空で、未カバー注記の要件も満たしていない。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:97-101`
  - traceability.md には具体的な設計反映先とテスト反映先を記載するよう要求。
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:146-153`
  - `traceability.md` から一意に追跡できること、未カバー要件を備考欄に明記することを PASS 条件 / FAIL 条件として明記。
- `dev-journal/deliverables/docs/60_test/traceability.md:67-80`
  - `WFL-F01/F02/F03`, `ATT-F02/F03/F04` が `test_cases/workflow.md#WFL-` や `test_cases/attachments.md#ATT-` のような具体性のない参照になっている。
- `dev-journal/deliverables/docs/60_test/traceability.md:102-126`
  - `RBC-002`, `RBC-011`, `RBC-012`, `WFL-001` ほか多数が同様にプレフィックスのみで、個別ケース特定ができない。
- `dev-journal/deliverables/docs/60_test/traceability.md:245-246`
  - `NFR-DATA-006`, `NFR-DATA-007` は `テスト反映先` が `-` なのに備考欄が空で、未カバー理由が明記されていない。

## 判定
高 / トレーサビリティ不備

## 修正方針案
各要件・ルールごとに、対応するテストIDを具体的な列挙または十分に狭い範囲で記載し、必要なら 1 要件を複数行に分割する。未カバー要件は `備考` に「運用確認」「Step 7 以降」「自動テスト対象外」などの理由と扱いを明記する。

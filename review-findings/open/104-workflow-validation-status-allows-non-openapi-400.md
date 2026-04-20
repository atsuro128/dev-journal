# 104: workflow の request validation ケースが OpenAPI にない 400 を許容している

## 指摘概要
`workflow.md` の `updated_at` 欠落や `reason` 文字数超過のテスト期待が「400 または 422」となっており、OpenAPI 正本に定義されていない 400 応答を許容している。

今回の工程 A は issue 121 の設計成果物側対応として、422 バリデーションエラーの `details` 契約を test_cases に反映する作業である。ここで 400 を許容したままにすると、後続の実装・テスト工程が 400 details なしの実装でも通せる余地が残り、OpenAPI の 422 契約と issue 121 の完了条件を十分に固定できない。

## 根拠
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:114`
  - `WFL-025` が `400 または 422（バリデーションエラー）。422 VALIDATION_ERROR の場合は details[].field = "updated_at"` を許容している。
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:133`
  - `WFL-030` が `400 または 422（バリデーションエラー）。422 VALIDATION_ERROR の場合は details[].field = "reason"` を許容している。
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:222`
  - `WFL-063` が `400 または 422（バリデーションエラー）。422 VALIDATION_ERROR の場合は details[].field = "updated_at"` を許容している。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1505`-`1520`
  - `approveReport` の requestBody は `updated_at` required。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1521`-`1545`
  - `approveReport` の responses は 401/403/404/409/422 で、400 は定義されていない。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1584`-`1589`
  - `rejectReport` の requestBody は `RejectRequest`。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:2313`-`2324`
  - `RejectRequest.reason` は required かつ maxLength 1000。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1701`-`1712`
  - `markReportAsPaid` の requestBody は `updated_at` required。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1736`-`1741`
  - `markReportAsPaid` の responses は 422 を定義しているが、400 は定義していない。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1856`-`1859`
  - Error.details は 400/422 のバリデーションエラー時に付与する契約。

## 判定
- 重大度: 中
- 分類: OpenAPI 契約不整合 / テスト期待の曖昧さ

## 修正方針案
`workflow.md` の request validation ケースは OpenAPI 正本に合わせて 422 に固定する。

- `WFL-025`: `422 VALIDATION_ERROR。details[].field = "updated_at" を含む`
- `WFL-030`: `422 VALIDATION_ERROR。details[].field = "reason" を含む`
- `WFL-063`: `422 VALIDATION_ERROR。details[].field = "updated_at" を含む`

もし実装方針として 400 を採用したい場合は、先に `openapi.yaml` の各 operation に 400 応答を追加し、400 時も `details` を返すかどうかを正本側で明示してから test_cases を合わせる。

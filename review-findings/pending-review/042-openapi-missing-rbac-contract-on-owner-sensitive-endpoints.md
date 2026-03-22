# 042: OpenAPI から所有権付き RBAC 条件を読み取れない

## 指摘概要
`openapi.yaml` では主要な保護対象 API に Bearer 認証と汎用的な `403/404` だけが定義されており、どのロールがどの条件で実行可能かが仕様として落ちていない。`GET /api/reports/{id}`、明細 CRUD、添付操作などは所有権・状態・ロールの組み合わせが実装成否を左右するため、OpenAPI だけでは実装者・テスターが認可条件を確定できない。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:557` `GET /api/reports/{id}` は「レポート詳細を取得する」とのみ記載し、誰が閲覧可能かを記していない。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:585` `PUT /api/reports/{id}` は draft 更新だけを記し、所有者ロール限定であることを記していない。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:699` `POST /api/reports/{id}/items` 以降の明細 API も、所有者かつ draft 状態のみという条件が仕様化されていない。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:797` `POST /api/reports/{id}/items/{itemId}/attachments` と `GET/DELETE` 系も、閲覧・削除・アップロードのロール条件が未記載。
- `dev-journal/deliverables/docs/40_basic_design/screens/report.md:677` ではレポート詳細画面の操作可否がロール×所有権で細かく分岐している。
- `dev-journal/deliverables/docs/50_detail_design/files.md:202` では添付アップロードの必要ロールを `Member, Approver, Admin, Accounting（所有者のみ）` と明記している。

## 判定
中 / 詳細設計の仕様欠落

## 修正方針案
OpenAPI の各保護対象エンドポイントに、少なくとも以下を明示する。
- 許可ロール
- 所有者条件の有無
- 状態条件（draft/submitted/approved など）
- テナント境界越え時は 404、同一テナント内の権限不足は 403 であること

機械可読性を重視するなら `x-required-roles` / `x-ownership` / `x-state-preconditions` のような拡張フィールドを追加し、説明文にも要約を置く。

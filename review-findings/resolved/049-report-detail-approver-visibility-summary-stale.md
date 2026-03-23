# 049: `report-detail.md` の Approver 閲覧範囲要約が認可設計の更新に追随していない

## 指摘概要
`rbac.md` / `authz.md` / `openapi.yaml` では、Approver は「自分が承認/却下したレポート」を状態変更後も追跡閲覧できる設計に統一された。一方、`screens/report-detail.md` のアクセス制御要約は「他者の submitted 状態レポートを閲覧 + 承認/却下が可能」とだけ記載しており、approved / rejected / paid に遷移した追跡閲覧ケースを表現できていない。詳細設計成果物間で同一画面の可視範囲説明が不一致になっている。

## 根拠
- `dev-journal/deliverables/docs/10_requirements/rbac.md:58`
  > Approver は自分が承認/却下したレポートを、状態変更後（approved/rejected/paid）も追跡閲覧できる
- `dev-journal/deliverables/docs/10_requirements/rbac.md:149`
  > RBC-015 ... 自分が承認/却下したレポートは状態変更後も追跡閲覧可能
- `dev-journal/deliverables/docs/50_detail_design/authz.md:565`
  > Approver | 自分のレポート + submitted 状態のテナント内全レポート + 自分が承認/却下したレポート
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md:370`
  > Approver は submitted 状態のレポートを閲覧 + 承認/却下が可能

## 判定
- 重大度: 中
- 分類: 内部整合性 / 画面詳細仕様

## 修正方針案
- `screens/report-detail.md` のアクセス制御要約に、Approver の追跡閲覧条件（`approved_by` / `rejected_by` に自分が記録されているレポートは状態変更後も閲覧可能）を追記する。
- 可能ならロール別表示差異マトリクスにも同条件を反映し、実装者が「閲覧可だが承認/却下ボタンは非表示」のケースを誤解しないようにする。

# 047: Approver の詳細閲覧範囲が上流 RBAC を超えて拡張されている

## 指摘概要
`authz.md` と `openapi.yaml` では、Approver が `submitted` 以外にも「自分が過去に承認/却下したレポート」を詳細閲覧できる設計になっている。しかし上流の `rbac.md` / `requirements.md` で認められている Approver の業務閲覧範囲は `submitted` レポートの承認待ち対象までであり、approved / rejected / paid レポートへの追加アクセス根拠が存在しない。結果として、Step 5 が上流要件にない閲覧権限を新設している。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/authz.md:565`
  > Approver | 自分のレポート + submitted 状態のテナント内全レポート + 自分が承認/却下したレポート
- `dev-journal/deliverables/docs/50_detail_design/authz.md:579`
  > `GET /api/reports/{id}`（詳細） | 自分のレポート + submitted + 過去に自分が承認/却下したレポート
- `dev-journal/deliverables/docs/50_detail_design/authz.md:616`
  > `OR report.approved_by == actor.user_id OR report.rejected_by == actor.user_id`
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:620`
  > Approver: 自分のレポート + 同一テナントの submitted 状態レポート + 自分が承認/却下したレポート
- `dev-journal/deliverables/docs/10_requirements/rbac.md:57`
  > MVP では Approver は同一テナント内の submitted 状態のレポートをすべて閲覧・承認・却下できる
- `dev-journal/deliverables/docs/10_requirements/rbac.md:102`
  > 明細閲覧 ... △ 承認対象のレポートのみ
- `dev-journal/deliverables/docs/10_requirements/requirements.md:221`
  > WFL-F04 | 承認待ち一覧 | Approver が同テナントの submitted レポート一覧を全件取得

## 判定
- 重大度: 高
- 分類: 上流整合性 / 認可

## 修正方針案
- Approver の他者レポート閲覧範囲を上流通り `submitted` の承認対象に限定する。
- もし「自分が過去に判断したレポートの追跡閲覧」を認めたいなら、Step 1 の `rbac.md` / `requirements.md` / usecase に明示追加したうえで設計全体を再整合させる。

## 対応内容
上流（rbac.md）に追跡閲覧の要件を追記して正式化。コミット `1f4aa58` で既に openapi.yaml/security.md/files.md は修正済みだったが、rbac.md への反映が漏れていた。

- rbac.md SS2.3 に「Approver は自分が承認/却下したレポートを状態変更後も追跡閲覧可能」を追記
- rbac.md SS4.2 RBC-015 に追跡閲覧の記述を追加

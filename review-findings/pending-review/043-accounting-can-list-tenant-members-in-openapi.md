# 043: tenant members API が Accounting に開放されており上流 RBAC と衝突している

## 指摘概要
`/api/tenant/members` が `Admin, Accounting` に開放されているが、上流の RBAC 要件ではメンバー一覧は Phase 3 機能かつ Admin のみである。今回の OpenAPI は Accounting にテナント全メンバー一覧を返せる仕様になっており、上流要件より権限を拡大している。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1222` `/api/tenant/members` の説明に `アクセス可能ロール: Admin, Accounting` とある。
- `dev-journal/deliverables/docs/10_requirements/rbac.md:109` 管理機能マトリクスでは `メンバー一覧（Phase 3）` は Admin のみ許可、Accounting は禁止となっている。
- `dev-journal/deliverables/docs/40_basic_design/screens/admin.md:90` では全レポート一覧の申請者フィルタのために「テナント内メンバー一覧」を使う前提になっており、画面都合で RBAC を拡張した状態になっている。

## 判定
中 / 上流要件との不整合

## 修正方針案
次のいずれかに統一する。
- RBAC を優先し、`/api/tenant/members` は Admin のみに制限する。Accounting 側の申請者フィルタは別方式にする（既存一覧データから候補生成、あるいはフィルタ UI をテキスト検索化）。
- もし Accounting に一覧取得を許可する判断なら、`rbac.md` と関連要件・画面仕様を更新して例外ではなく正式仕様に格上げする。

現状のまま実装に進むと、権限制御の期待値が文書ごとに分岐する。

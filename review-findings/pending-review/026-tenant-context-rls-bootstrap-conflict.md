---
step: 3
severity: high
status: open
---

# 026: TenantContext 初期化フローが RLS 前提と循環している

## 指摘概要

Step 3 のテナント分離設計では、`tenant_memberships` にも RLS を適用する方針を採っています。一方で、認証済みリクエスト時の `TenantContext` は `tenant_memberships` を読んで `tenant_id` を取得してから `SET app.current_tenant` を実行する設計になっています。RLS の適用前に RLS 対象テーブルを参照する循環依存になっており、このままではリクエスト時のテナントコンテキストを初期化できません。

## 根拠

- ADR-0003 では `tenant_memberships` を RLS 適用対象としている
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:42`
- 同じ ADR-0003 では、`SET app.current_tenant` を先に実行してから以降のクエリに RLS を自動適用する前提になっている
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:74`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:76`
- しかし architecture.md のミドルウェア設計では、`JWT の user_id → TenantMembership → tenant_id` を取得してから `SET app.current_tenant` を実行すると記載している
  - `dev-journal/deliverables/docs/30_arch/architecture.md:173`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:174`
- diagrams.md のリクエスト処理フローでも、`SELECT tenant_id FROM tenant_memberships` の後に `SET app.current_tenant` を実行している
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:68`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:70`
- 一方で architecture.md の「テナント分離の実行フロー」と diagrams.md の「テナント分離フロー」は、JWT claims から直接 `tenant_id` を取得して `SET app.current_tenant` する前提になっており、文書内でも方針が分裂している
  - `dev-journal/deliverables/docs/30_arch/architecture.md:237`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:238`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:141`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:142`

## 判定

- 重大度: 高
- 分類: 上流整合性 / セキュリティ設計 / 文書内不整合

RLS の成立条件そのものが崩れており、認証済みリクエストの正常系を設計どおりに実装できません。さらに、RLS を回避する未記載の特別経路を実装すると、二重保証の前提も曖昧になります。

## 修正方針案

- `TenantContext` の初期化方式を 1 つに統一する
- 現状の ADR と整合させるなら、認証済みリクエストでは JWT claims の `tenant_id` をそのまま使って `SET app.current_tenant` を実行する
- もし毎回 `tenant_memberships` を引いて所属確認したいなら、RLS 適用前に安全に参照できる専用経路を ADR に明記する
  - 例: RLS 非適用の専用ロール、`SECURITY DEFINER` 関数、認証専用コネクション
- architecture.md / diagrams.md / ADR-0003 の 3 文書で同じ初期化順序になるように修正する

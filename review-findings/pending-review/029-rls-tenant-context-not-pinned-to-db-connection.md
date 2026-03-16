---
step: 3
severity: high
status: open
---

# 029: TenantContext の RLS 設定が実際の DB 接続に束縛されていない

## 指摘概要

`TenantContext` ミドルウェアで `SET app.current_tenant` / `RESET app.current_tenant` を実行する方針になっていますが、その設定を行った接続を以後の repository/sqlc クエリでも使い続ける設計が定義されていません。通常のコネクションプール経由の実装では、ミドルウェアで設定したセッション変数と、後続クエリが実行される接続が一致する保証がないため、RLS が設計どおりに効かないリスクがあります。

## 根拠

- architecture.md では、ミドルウェア段で `TenantContext` が `SET app.current_tenant` を実行し、その後に handler / service / repository が処理を続ける構成としている
  - `dev-journal/deliverables/docs/30_arch/architecture.md:41`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:43`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:146`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:173`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:175`
- 同じ architecture.md では repository 層は sqlc ベースで tenant filter を強制するだけで、接続固定やトランザクション境界は書かれていない
  - `dev-journal/deliverables/docs/30_arch/architecture.md:66`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:67`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:68`
- ADR-0003 でも、プールから接続取得 → `SET app.current_tenant` → 業務処理 → `RESET` という前提だが、後続クエリを同一接続へ束縛する方式は定義していない
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:20`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:21`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:63`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:70`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:72`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:76`

## 判定

- 重大度: 高
- 分類: セキュリティ設計 / 実装成立性

RLS を二重保証の中核に据えているにもかかわらず、`SET` したセッション変数が実際の SQL 実行接続へ確実に引き継がれない設計になっています。このままだとテナント分離が「文書上は成立しているが、実装時には接続プール次第で破綻する」状態になります。

## 修正方針案

- RLS コンテキストの適用単位を明文化する
  - 例: リクエスト単位で専用コネクションを `Acquire` し、その接続を handler/service/repository へ明示的に渡す
  - 例: リクエスト単位トランザクションを開始し、`SET LOCAL app.current_tenant` を使う
- `sqlc` 生成コードをどの DB ハンドルで実行するかまで含めて、接続固定の責務を architecture.md / ADR-0003 / diagrams.md で統一する
- `RESET` 漏れ対策だけでなく、「同一接続で実行される保証」の設計を追加する

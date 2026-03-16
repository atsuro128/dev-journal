---
step: 3
severity: high
status: resolved
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

## 再レビュー結果（2026-03-16）

対応不十分（差し戻し）。

### 確認内容
- `architecture.md` のミドルウェアチェーンには、`pool.Acquire()`・`BEGIN + SET LOCAL`・`context` 伝播・`COMMIT/ROLLBACK` が追記された
  - `dev-journal/deliverables/docs/30_arch/architecture.md:173`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:176`
- `ADR-0003` にも、リクエスト単位でコネクションを固定し、同一コネクション・同一トランザクションで実行する方針が追記された
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:71`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:83`
- ただし同じ `architecture.md` の認証付きリクエスト例とテナント分離フローには、旧設計の `SET` / `RESET` だけの説明が残っている
  - `dev-journal/deliverables/docs/30_arch/architecture.md:217`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:221`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:227`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:237`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:239`
- `diagrams.md` のテナント分離フロー図も、`SET app.current_tenant` と `RESET app.current_tenant` の旧フローのまま
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:143`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:161`

### 判定理由
接続固定を明文化する修正自体は入っているが、同一成果物の中に「接続固定あり」と「接続固定なし」の両方の説明が共存している。現状では実装者が旧フローを参照してしまう余地が残るため、本指摘は未解消と判定する。

---

## 再レビュー結果（2026-03-16 / 2回目）

対応妥当（クローズ）。

### 確認内容
- `architecture.md` の認証付きリクエスト例とテナント分離フローが、`Acquire conn + BEGIN + SET LOCAL` と同一コネクション前提に統一された
  - `dev-journal/deliverables/docs/30_arch/architecture.md:218`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:223`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:242`
- `diagrams.md` も `Acquire conn + BEGIN + SET LOCAL` / `COMMIT` / `Release conn` の流れに更新され、旧来の `SET` / `RESET` のみの図が解消された
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:120`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:143`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:161`
- `ADR-0003` は引き続き `pgxpool.Acquire()` による接続固定と `SET LOCAL` を明記している
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:71`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:83`

### 判定理由
Step 3 成果物間で、RLS テナントコンテキストを同一コネクション・同一トランザクションに束縛する設計へ統一されたため。

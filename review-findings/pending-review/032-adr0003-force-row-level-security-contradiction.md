---
step: 3
severity: high
status: open
---

# 032: ADR-0003 の FORCE ROW LEVEL SECURITY 記述が認証バイパス方針と矛盾している

## 指摘概要

`ADR-0003` では、認証処理で `tenant_memberships` を参照するためにテーブルオーナーロールで RLS をバイパスする方針を採っています。しかし同じ ADR の SQL サンプルは `ALTER TABLE ... FORCE ROW LEVEL SECURITY;` を含んでおり、さらにリスク緩和表では `FORCE ROW LEVEL SECURITY` を「業務用ロールにのみ適用」と記載しています。`FORCE ROW LEVEL SECURITY` はテーブル単位の設定であり、サンプルどおりに実装するとオーナーロールのバイパス前提と両立しません。

## 根拠

- SQL サンプルには `FORCE ROW LEVEL SECURITY` が含まれている
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:52`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:53`
- 直後の説明では、認証処理のため `FORCE ROW LEVEL SECURITY` は使用しないとしている
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:59`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:60`
- 認証フローでは、テーブルオーナーロールで `tenant_memberships` を直接参照して RLS をバイパスする前提を置いている
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:94`
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:101`
- リスク緩和表では `FORCE ROW LEVEL SECURITY` を「業務用ロールにのみ適用」と記述しているが、テーブル単位設定として読むと成立しない
  - `dev-journal/deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md:114`

## 判定

- 重大度: 高
- 分類: 設計矛盾 / 実装成立性

認証時の RLS バイパス可否はログイン実装の成立条件です。ADR のサンプルと説明が逆方向を向いているため、このままでは実装者がどちらを採るべきか判断できず、認証または RLS 設定のどちらかを壊すリスクがあります。

## 修正方針案

- `FORCE ROW LEVEL SECURITY` を採用しないなら、SQL サンプルから該当行を削除し、説明文・リスク表と統一する
- 逆に `FORCE ROW LEVEL SECURITY` を採用するなら、認証時の `tenant_memberships` 参照方法をオーナーロール以外の別方式へ再設計する
- 少なくとも ADR-0003 内で、テーブルオーナーが RLS をバイパスできる前提と `FORCE` 設定の関係を一意に読める記述へ修正する

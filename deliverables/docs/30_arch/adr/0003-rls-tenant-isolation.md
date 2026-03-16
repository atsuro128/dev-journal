# ADR-0003: RLS テナント分離

## ステータス
承認済

## 日付
2026-03-16

## 背景（ペイン）
- ADR-0002 で Shared DB + tenant_id 方式を採用した
- アプリケーション層の `WHERE tenant_id = ?` だけではバグや実装漏れによるテナント越えを防げない
- 要件 TNT-004「PostgreSQL RLS でアプリ層の保証を二重化」を実現する具体的な方式を決定する必要がある
- ドメインモデルでは ExpenseReport, ExpenseItem, Attachment が tenant_id を冗長保持しており、RLS を JOIN なしで適用可能な構造になっている

## 検討した選択肢

### RLS のテナントコンテキスト伝達方式

#### 案A: SET app.current_tenant（セッション変数方式）
- **概要**: PostgreSQL のカスタムセッション変数 `app.current_tenant` にテナントIDを設定し、RLS ポリシーで `current_setting('app.current_tenant')` を参照
- **利点**: コネクション取得時に1回設定するだけで、以降の全クエリに自動適用される。アプリケーションコードの各クエリに tenant_id を渡す必要がない（RLS 層）。PostgreSQL の標準機能で追加モジュール不要
- **欠点**: コネクションプール使用時にセッション変数のリセット漏れがテナント越えに直結（→ 緩和策あり）

#### 案B: 全クエリに tenant_id パラメータを渡す（RLS なし）
- **概要**: アプリケーション層の WHERE 句のみでテナント分離を保証
- **利点**: シンプル、DB 側の設定不要
- **欠点**: 二重保証にならない。1つの WHERE 句漏れがテナント越えに直結。TNT-004 を充足しない

#### 案C: DB ユーザーをテナントごとに作成
- **概要**: テナントごとに PostgreSQL ユーザーを作成し、GRANT でアクセス範囲を制御
- **利点**: PostgreSQL の権限モデルで厳密に分離
- **欠点**: テナント追加のたびにユーザー作成が必要、コネクションプールの複雑化、RDS の接続数制限に抵触するリスク

## 決定
**案A: SET app.current_tenant（セッション変数方式）** を採用し、アプリケーション層の WHERE tenant_id との二重保証とする。

### RLS ポリシーの適用対象

| テーブル | tenant_id | RLS 適用 | 備考 |
|----------|-----------|----------|------|
| tenants | tenant_id (PK) | Yes | 自テナント情報のみ参照可 |
| tenant_memberships | tenant_id | Yes | 自テナントのメンバーのみ参照可 |
| expense_reports | tenant_id | Yes | テナント分離の中核 |
| expense_items | tenant_id | Yes | 冗長保持により JOIN 不要で RLS 適用 |
| attachments | tenant_id | Yes | 冗長保持により JOIN 不要で RLS 適用 |
| users | (なし) | No | tenant_id を持たない。TenantMembership 経由でアクセス制御 |

### RLS ポリシー設計

```sql
-- 基本パターン（全対象テーブルに適用）
ALTER TABLE expense_reports ENABLE ROW LEVEL SECURITY;
ALTER TABLE expense_reports FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON expense_reports
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

- `ENABLE ROW LEVEL SECURITY`: テーブルオーナー以外に RLS を有効化
- `FORCE ROW LEVEL SECURITY` は **使用しない**: テーブルオーナーロールは認証処理（ログイン・サインアップ）での `tenant_memberships` 参照に必要なため、RLS をバイパスできる状態を維持する。業務用ロール（アプリケーション接続）には `ENABLE` のみで RLS が適用される
- ポリシーは `USING` 句のみ（SELECT/UPDATE/DELETE に適用）。INSERT 時は `WITH CHECK` も同一条件で適用
- **DB ロールの分離**: テーブルオーナーロール（マイグレーション + 認証用）と業務用ロール（通常リクエスト用）の2ロール構成

### コネクション管理フロー（認証済みリクエスト）

```
リクエスト受信
  ↓
JWT claims から tenant_id を取得（ミドルウェア）
  ↓
pool.Acquire() で専用コネクションを取得 ← ★ リクエスト単位で固定
  ↓
BEGIN + SET LOCAL app.current_tenant = '{tenant_id}'
  ← ★ SET LOCAL はトランザクション内でのみ有効（COMMIT/ROLLBACK で自動リセット）
  ↓
ビジネスロジック実行（全クエリが同一コネクション・同一トランザクション内で実行）
  ↓
COMMIT or ROLLBACK  ← ★ SET LOCAL は自動的にリセットされる
  ↓
conn.Release() でコネクションをプールに返却
```

**接続固定の保証方式**: Go の `pgxpool.Pool.Acquire()` でリクエスト単位にコネクションを固定し、`context.Context` 経由で handler → service → repository に伝播する。`SET LOCAL` はトランザクションスコープなので、COMMIT/ROLLBACK 後に自動リセットされ、`RESET` 漏れのリスクを排除する。

### 認証エンドポイントの RLS バイパス

ログイン・サインアップ等の認証エンドポイントでは、JWT が未発行のため `app.current_tenant` を設定できない。`tenant_memberships`（RLS 適用対象）から `role, tenant_id` を取得する必要があるため、以下の方式で RLS をバイパスする。

```
[ログイン / サインアップ]
  ↓
users テーブルから user 取得（RLS 非適用）
  ↓
tenant_memberships から role, tenant_id 取得
  ← ★ RLS バイパス: テーブルオーナーロール（マイグレーション用）で直接参照
  ← user_id での絞り込みにより、該当ユーザーの所属テナントのみ取得
  ↓
JWT 生成（claims に tenant_id, role を含める）
```

**バイパス方式**: 認証処理専用に、RLS が適用されない DB ロール（テーブルオーナー）の接続を使用する。このロールは認証サービス内でのみ使用し、業務用リポジトリ層には公開しない。`FORCE ROW LEVEL SECURITY` はテーブルオーナーには適用しないことで、認証時の参照を許可する。

## 理由
1. **二重保証の実現**: アプリ層（リポジトリの WHERE tenant_id）+ DB層（RLS）で、どちらかに漏れがあっても他方で防止。セキュリティの深層防御の原則に合致
2. **tenant_id 冗長保持との整合**: domain_model.md で ExpenseItem, Attachment に tenant_id を冗長保持している設計理由が「RLS を JOIN なしで適用するため」であり、本方式と完全に整合
3. **sqlc との相性**: sqlc で生成するクエリには WHERE tenant_id を含めつつ、RLS がセーフティネットとして機能する。sqlc は SQL を直接書くため、RLS の挙動を正確に把握しやすい

### リスクと緩和策

| リスク | 緩和策 |
|--------|--------|
| コネクション返却時のリセット漏れ | `SET LOCAL` + トランザクションスコープにより、COMMIT/ROLLBACK で自動リセット。`RESET` の明示的呼び出しが不要 |
| 別コネクションでのクエリ実行 | `pgxpool.Acquire()` でリクエスト単位にコネクションを固定し、`context.Context` 経由で全層に伝播。プールからの自動取得は使用しない |
| マイグレーション時の RLS 干渉 | マイグレーション専用ユーザー（テーブルオーナー）を使用。`FORCE ROW LEVEL SECURITY` は業務用ロールにのみ適用 |
| 認証エンドポイントでの RLS 循環 | 認証処理はテーブルオーナーロールの専用接続を使用し、RLS をバイパス。業務ロジックには公開しない |
| テスト時の RLS | テスト用にテナントコンテキストを設定するヘルパーを用意。テナント分離テスト（requirements.md §テスト）で RLS の動作を検証 |

## 影響・結果
- 全リクエストのミドルウェアで `SET app.current_tenant` の実行が必須。パフォーマンスへの影響は軽微（SET は即座に完了）
- DB マイグレーションに RLS ポリシーの CREATE/ALTER を含める。golang-migrate で管理
- テナント分離テストは RLS の動作検証を含む（別テナントのデータが見えないことを確認）
- 管理者用のクロステナントクエリ（将来的な SaaS 運営側の管理画面等）が必要になった場合は、RLS をバイパスする専用ロールを作成する

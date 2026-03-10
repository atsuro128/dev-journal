# ドメイン設計 判断ログ

Step 2（ドメイン設計）で行った設計判断を記録する。

---

## D-001: tenant_id の冗長保持

| 項目 | 内容 |
|------|------|
| 対象 | ExpenseItem, Attachment の tenant_id カラム |
| 採用案 | 子テーブルにも tenant_id を冗長保持する |
| 不採用案 | 親テーブル（ExpenseReport）の tenant_id のみで管理し、JOIN で取得する |
| 理由 | PostgreSQL RLS ポリシーが各テーブル単独で適用されるため、JOIN なしで RLS が機能する設計が必要。パフォーマンスの観点でも JOIN 回避が有利 |
| トレードオフ | データの冗長性が増すが、テナント分離の確実性とクエリ効率が勝る |

---

## D-002: 集約の境界 — ExpenseReport を Aggregate Root とする

| 項目 | 内容 |
|------|------|
| 対象 | ExpenseReport / ExpenseItem / Attachment の集約設計 |
| 採用案 | ExpenseReport を Aggregate Root とし、ExpenseItem と Attachment を子エンティティとする |
| 不採用案A | 各エンティティを独立した集約にする |
| 不採用案B | ExpenseItem を独立集約にし、Attachment のみ ExpenseReport 配下にする |
| 理由 | 状態遷移（draft 以外では明細・添付の変更不可）がレポート単位で制御されるため、レポートを root として一体管理するのが自然。合計金額の自動計算もレポート集約内で完結する |

---

## D-003: 再申請は新規レポートとして作成する

| 項目 | 内容 |
|------|------|
| 対象 | 却下後の再申請方式 |
| 採用案 | 元レポートを rejected のまま保持し、新規レポート（draft）を作成。reference_report_id で元レポートを参照 |
| 不採用案 | 却下レポートを draft に戻して再編集可能にする |
| 理由 | 監査証跡の保持。元の却下レポートと却下理由が永続的に残るため、変更履歴の追跡が容易。状態遷移の複雑化（rejected → draft という逆方向遷移）を避けられる |

---

## D-004: 状態遷移メソッドを遷移ごとに個別実装する

| 項目 | 内容 |
|------|------|
| 対象 | ExpenseReport の状態遷移 API 設計 |
| 採用案 | submit(), approve(), reject(), mark_as_paid() の個別メソッド |
| 不採用案 | transition_to(target_status) の汎用メソッド |
| 理由 | 遷移ごとに事前条件・入力・事後処理が異なる（reject は理由必須、approve はコメント任意、mark_as_paid は入力なし）。個別メソッドの方が型安全性が高く、テストも書きやすい |

---

## D-005: 楽観的ロックによる同時操作制御

| 項目 | 内容 |
|------|------|
| 対象 | 複数ユーザーの同時操作競合 |
| 採用案 | updated_at による楽観的ロック |
| 不採用案 | DB の排他ロック（SELECT FOR UPDATE） |
| 理由 | 経費精算の同時操作頻度は低い。楽観的ロックは実装がシンプルで、ロック待ちによるパフォーマンス劣化がない。競合時はエラーを返してユーザーにリトライを促す |

---

## D-006: User テーブルに tenant_id を持たない

| 項目 | 内容 |
|------|------|
| 対象 | User と Tenant の関連設計 |
| 採用案 | TenantMembership テーブルで User と Tenant を関連付ける。User テーブルには tenant_id を持たない |
| 不採用案 | User テーブルに直接 tenant_id を持つ |
| 理由 | MVP では 1ユーザー = 1テナントだが、将来的に1ユーザーが複数テナントに所属する拡張を阻害しない。TenantMembership テーブルにロール情報も持たせることで、テナント×ロールの管理が一箇所に集約される |

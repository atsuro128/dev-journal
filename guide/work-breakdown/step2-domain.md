# Step 2: ドメイン設計（データとルールの核） — 作業分解

## 概要

実装に入る前に、主要データ構造と不変条件（ルール）を固める。

## 成果物（出力）

| 成果物 | パス |
|--------|------|
| 主要エンティティと関係 | `deliverables/docs/20_domain/domain_model.md` |
| 状態遷移の詳細・禁止操作 | `deliverables/docs/20_domain/state_machine.md` |

## 重要な不変条件（例）

- テナント（workspace）を跨ぐ参照は禁止
- 承認中は編集制限（状態によって操作可否が変わる）
- 添付URL発行前に認可チェック必須
- 監査ログは INSERT ONLY（更新・削除不可）
- 不正な状態遷移（例：`draft` → `approved`）はドメイン層で拒否

## 【root-project 整備】

- `references/decisions/`：ドメイン設計上の採用/不採用の判断ログをファイルとして記録
  （例：`20_domain-invariants.md` — テナント分離をアプリ層に寄せる vs RLS に寄せる等の判断）

## 完了条件

- 主要エンティティと責務が説明できる
- 重要ルールが文章化されている
- audit_logs の INSERT ONLY 制約が明記されている

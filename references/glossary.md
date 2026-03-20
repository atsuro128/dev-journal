# プロジェクトで使用する用語集

> AI参照用。詳細は `deliverables/docs/01_glossary.md` を参照。

## コアドメイン用語

| 用語 | 英語 | コード上の表現 |
|---|---|---|
| テナント | Tenant | テーブル: `tenants`, カラム: `tenant_id` |
| 経費レポート | Expense Report | テーブル: `expense_reports` |
| 経費明細 | Expense Item | テーブル: `expense_items` |
| 領収書/添付 | Attachment | テーブル: `attachments` |

## ロール

| ロール | DB値 | 主な権限 |
|---|---|---|
| Admin | `admin` | 全操作 + メンバー管理 + 招待 |
| Approver | `approver` | 承認・却下 |
| Member | `member` | 自分の経費の申請・編集 |
| Accounting | `accounting` | 自分の経費申請・経費閲覧・エクスポート・支払済み更新 |

## 状態遷移

| 状態 | DB値 | 遷移元 → 遷移先 |
|---|---|---|
| 下書き | `draft` | (初期状態) → `submitted` |
| 提出済み | `submitted` | `draft` → ここ → `approved` or `rejected` |
| 承認済み | `approved` | `submitted` → ここ → `paid` |
| 却下 | `rejected` | `submitted` → ここ（終端。再申請は新規レポート） |
| 支払済み | `paid` | `approved` → ここ（終端） |

## 操作用語の統一

| 操作 | 使う表現 | 使わない表現 |
|---|---|---|
| 承認者が申請を差し戻す | 却下 (reject) | 差戻し、リジェクト |
| 却下後に再度申請する | 再申請 (resubmit) | 再提出、やり直し |
| 経理が支払記録する | 支払完了 (mark as paid) | 精算、清算 |
| レポートを承認者に出す | 提出 (submit) | 申請、送信 |

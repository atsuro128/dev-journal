---
step: 3
severity: medium
status: open
---

# 028: Single-AZ RDS の採用理由が 99.5% 可用性要件を支えられていない

## 指摘概要

非機能要件では月間 99.5% の稼働率を求めていますが、ADR-0004 は MVP で RDS を Single-AZ 運用にすると決めています。可用性要件とのトレースが不足しており、現状の文書では「DB 障害時にどう 99.5% を守るのか」が説明されていません。さらに、ADR-0004 は ALB の `/health` チェックを可用性要件への対応理由として挙げていますが、これは API タスクの異常検知であり、Single-AZ RDS の単一点障害を解消しません。

## 根拠

- 要件定義では可用性目標を 99.5% とし、`/health` と DB 自動バックアップを要求している
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:428`
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:429`
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:430`
- ADR-0004 自身も 99.5% を前提条件としている
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:11`
- しかし決定内容では RDS を `Multi-AZ は初期は無効`、コスト表でも `Single-AZ` としている
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:67`
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:92`
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:102`
- そのうえ理由欄では、ALB の `/health` チェックで可用性要件に対応すると説明している
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:81`

## 判定

- 重大度: 中
- 分類: 非機能要件トレーサビリティ / 可用性設計

現在の記述では、アプリ層の多重化はあっても DB は単一点障害のままです。可用性要件を満たす設計なのか、あるいは MVP ではリスク受容するのかが判断できません。

## 修正方針案

- 次のいずれかを明示する
  - MVP でも可用性要件を満たすため Multi-AZ を採用する
  - 99.5% を Single-AZ で満たせる根拠と障害時運用を具体化する
  - MVP では可用性目標を緩和し、要件定義側と合意のうえで更新する
- あわせて、ALB ヘルスチェックが担保する範囲と、DB 障害に対する対策範囲を分けて記述する

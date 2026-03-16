---
step: 3
severity: medium
status: resolved
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

## 再レビュー結果（2026-03-16）

- `ADR-0004` に可用性リスク受容の説明と、ALB ヘルスチェックが担う範囲の切り分けは追記された
- ただし `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:102` の「RDS Single-AZ の AWS SLA は 99.95%」は事実誤認
- AWS 公式の Amazon RDS SLA では、99.95% は Multi-AZ DB Instance / Cluster の値であり、Single-DB Instance SLA は 99.5%
- 可用性要件を支える根拠として引用している数値が誤っているため、本指摘は未解消と判定する

---

## 再レビュー結果（2026-03-16 / 2回目）

対応妥当（クローズ）。

### 確認内容
- `ADR-0004` の可用性リスク受容において、Single-DB Instance は 99.5%、Multi-AZ は 99.95% と修正済み
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:102`
- ALB ヘルスチェックは ECS 層の異常検知であり、DB 層の可用性とは別役割であることを明示済み
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:103`
- 可用性要件 99.5% と、MVP では Single-AZ をリスク受容で採用する判断の関係が ADR 上で説明されている
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:428`
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:102`

### 判定理由
当初の未解消理由だった SLA 数値の誤記が解消され、可用性要件と Single-AZ 採用判断のトレーサビリティも ADR 上で成立したため。

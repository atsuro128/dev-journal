---
step: 3
severity: medium
status: open
---

# 031: Logs Insights 集計だけでは定義済みの p95/5xx アラートを継続監視できない

## 指摘概要

監視戦略ではカスタムメトリクスを「構造化ログを CloudWatch Logs Insights で集計する」方針にしていますが、同じ ADR 内で CloudWatch のカスタムメトリクス送信には EMF か `PutMetricData` が必要と認めています。にもかかわらず、p95 レスポンスタイムや 5xx レートに対するアラートを定義しており、アラーム元となる時系列メトリクスの生成方法が設計上抜けています。

## 根拠

- ADR-0005 は CloudWatch 案の欠点として、カスタムメトリクス送信には EMF か `PutMetricData` が必要だと明記している
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:36`
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:39`
- しかし決定では、カスタムメトリクスを Logs Insights 集計のみで取得するとしている
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:95`
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:97`
- そのうえで p95 レスポンスタイム、5xx レートに閾値付きアラートを張る設計になっている
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:103`
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:105`
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:144`
  - `dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md:145`
- architecture.md も ADR-0005 の結論として CloudWatch Alarms を前提にしているが、同じくメトリクス生成方式は補っていない
  - `dev-journal/deliverables/docs/30_arch/architecture.md:523`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:524`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:526`

## 判定

- 重大度: 中
- 分類: 監視設計 / 非機能要件トレーサビリティ

要件では API p95 500ms 以下を継続監視すべきですが、現在の設計だとダッシュボード閲覧用の集計とアラーム発火用の時系列メトリクスが混同されています。このままでは「測れる」が「監視できない」設計になります。

## 修正方針案

- p95/5xx アラート用のメトリクス生成方式を明示する
  - 例: EMF をログへ埋め込み、CloudWatch Metrics へ自動変換する
  - 例: アプリケーションから `PutMetricData` を送る
  - 例: 監視対象を Logs Insights でなくメトリクスフィルタで表現可能な指標に限定する
- 「ダッシュボード用途の集計」と「CloudWatch Alarm のソース」を ADR-0005 上で分けて記述する
- architecture.md の監視方針表も、アラームの根拠メトリクスを追加して整合させる

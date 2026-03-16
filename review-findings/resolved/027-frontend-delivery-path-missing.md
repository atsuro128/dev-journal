---
step: 3
severity: medium
status: open
---

# 027: 本番構成に SPA 配信経路が定義されていない

## 指摘概要

Step 3 は「技術選定と全体構成を決定する」フェーズですが、本番で React/Vite の成果物をどこから配信するかが設計に現れていません。`frontend/` ディレクトリと `Vite build` は定義されている一方で、システム構成図・ADR-0004 の本番構成は `ALB -> Go API on ECS Fargate` と領収書用 S3 しか示しておらず、ブラウザが初期 HTML/JS/CSS を取得する経路が未確定です。

## 根拠

- architecture.md ではフロントエンドを独立した `expense-saas/frontend/` として設計している
  - `dev-journal/deliverables/docs/30_arch/architecture.md:291`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:321`
- diagrams.md のシステム構成図は、ブラウザから ALB を経て Go API タスクへ到達する構成だけを示しており、SPA 配信用のホストが存在しない
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:7`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:12`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:15`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:29`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:30`
- ADR-0004 の決定でも、コンピュートは `ALB -> Fargate タスク（Go バイナリ）`、ストレージは領収書保存用 S3 のみで、SPA 配信先が書かれていない
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:66`
  - `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:68`
- それにもかかわらずデプロイパイプライン図では `Vite build` を行っており、生成物の配置先が未接続のままになっている
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:193`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:194`

## 判定

- 重大度: 中
- 分類: 構成欠落 / デプロイ設計

このままでは「フロントエンドをどこから配信するのか」が未決定のため、Step 3 の成果物だけでは本番構成を再現できません。Step 4 以降の画面設計や Step 7 のデプロイ設計にも直結します。

## 修正方針案

- SPA 配信方式を明示する
  - 例1: Go アプリが `Vite build` の静的成果物を同一コンテナから配信する
  - 例2: S3 + CDN で静的配信し、API は ALB/ECS に分離する
  - 例3: フロントエンド専用コンテナ/サービスを別系統で配置する
- 選んだ方式に合わせて、以下を architecture.md / diagrams.md / ADR-0004 に反映する
  - ランタイム構成
  - ルーティング
  - ビルド成果物の配置先
  - デプロイパイプライン

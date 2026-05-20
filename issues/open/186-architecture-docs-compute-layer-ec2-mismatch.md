# 設計書のコンピュート層表記が ECS Fargate のままで実装実態（EC2 t3.micro 単一）と乖離

## 発見日
2026-05-20

## カテゴリ
architecture

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 11-E / issue #185 T5（設計書整合）/ ADR-0004（インフラ構成・ポートフォリオ対応）

## ブロッカー
なし

## 問題

複数の設計成果物がコンピュート層を「ECS Fargate」「2 タスク（Task1 / Task2）」と表記しているが、実装実態は EC2 t3.micro 単一インスタンスである。

| ファイル | 該当箇所 | 現状の表記 |
|---|---|---|
| `deliverables/docs/30_arch/architecture.md` | §2 ほか | 「ECS Fargate 上の Go API サーバー」「Task」 |
| `deliverables/docs/30_arch/diagrams.md` | §1 システム構成図 | `subgraph ECS["ECS Fargate"]` / `Task1` / `Task2` |
| `deliverables/docs/70_operations/env_config.md` | §3.1 コンピュート行 | 「ECS Fargate（0.25 vCPU / 0.5 GB x 1〜2 タスク）」 |

一方、実装済み Terraform（`expense-saas/infra/terraform/ec2.tf`）は単一 EC2 t3.micro インスタンスで、ALB ターゲットグループにも EC2 1 台のみがアタッチされている。ADR-0004 のポートフォリオ対応方針でも「EC2 t3.micro」を明記している。

## 影響

- 設計成果物（構成図・環境設定）と実装・ADR-0004 の間で二重表記となり、ポートフォリオの設計説明時に混乱を招く。
- issue #185 T5 で CloudFront を構成図に追加した際、同じ成果物群の中でコンピュート層だけが実態と乖離したままとなり整合性を損なっている（issue #185 T5 内部レビュー W-1 で検出）。

## 背景

ADR-0004 で初期はマネージドコンテナ（ECS Fargate）を想定していたが、ポートフォリオ対応方針として EC2 t3.micro に変更された。この変更が architecture.md / diagrams.md / env_config.md のコンピュート層表記に反映されていない（issue #185 以前からの既存乖離）。issue #185 T5 のスコープは CloudFront 追加に限定されていたため、本乖離は T5 では是正せず別 issue として切り出した。

## 提案

architecture.md / diagrams.md / env_config.md のコンピュート層表記を、実装実態・ADR-0004 に合わせて EC2 t3.micro 単一インスタンスに統一する。「Task」「ECS」等の ECS 由来の用語も EC2 ベースの記述に置き換える。

post-MVP 対応で可（影響度低、実装・正本 ADR には乖離なし、設計成果物の表記のみの問題）。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

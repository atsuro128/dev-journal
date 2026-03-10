# 020: 提出時の「Approver 0人禁止」がドメイン条件に落ちていない

## 指摘概要
Step 1 では「Approver が 0 人のテナントでは提出不可」と整理されているが、Step 2 の状態遷移設計では提出前提条件に反映されていない。現状の設計だと承認不能な `submitted` レポートを生成できる。

## 根拠
- 要件定義の MVP スコープ判断で、詰み対策として「Approver 0人テナントでは提出不可」としている
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:254`
- ユースケースでも提出時の例外系として同ルールを定義している
  - `dev-journal/deliverables/docs/10_requirements/usecases.md:186-189`
- しかし Step 2 の提出前条件は `draft` と「明細1件以上」だけで、Approver 存在チェックがない
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:61-67`
- 集約責務でも提出事前条件は「明細1件以上」のみになっている
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:327-331`

## 判定
承認フローを詰ませる業務ルールの落ち漏れ（中）。

## 修正方針案
提出の事前条件に「同一テナントに承認可能な Approver が1人以上存在すること」を追加し、責務がドメイン層かアプリケーションサービス層かも明示する。

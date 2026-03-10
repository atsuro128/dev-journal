# 021: 署名付きURL発行前の認可チェックが Step 2 の不変条件に落ちていない

## 指摘概要
要件定義では `ATT-011` として「署名付きURL発行前に認可チェック必須」を定義しているが、Step 2 では添付の不変条件・責務分担として明文化されていない。添付アクセス制御の設計が未確定のまま残っている。

## 根拠
- Step 1 の要件定義では `ATT-011` が必須ルールとして定義されている
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:300-305`
- しかし `domain_model.md` の集約責務は添付について「draft 以外では追加・削除不可」までしか定義していない
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:327-331`
- 所有権・権限の不変条件にも、添付ダウンロードや署名付きURL発行前チェックに関するルールがない
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:364-379`
- `state_machine.md` でも添付は状態別の可否だけで、URL発行前の認可責務が明示されていない
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:246-254`

## 判定
重要なセキュリティルールのドメイン設計への落とし込み漏れ（中）。

## 修正方針案
Step 2 の不変条件か責務マッピングに `ATT-011` を追加し、「誰のどのレポートに紐づく添付なら URL 発行可能か」を ownership / tenant / role の観点で文章化する。

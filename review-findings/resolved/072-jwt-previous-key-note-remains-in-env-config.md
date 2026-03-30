# 072: env_config に削除済み変数 `JWT_PUBLIC_KEY_PREVIOUS` の記述が残っている

## 指摘概要

`env_config.md` から `JWT_PUBLIC_KEY_PREVIOUS` を削除したはずだが、本文中に「Phase 3 で追加予定」として 2 箇所残っている。今回の変更方針では、`security.md` 側で鍵ローテーションを Phase 3 に移したうえで、`env_config.md` から当該変数を除外するのが合意事項だったため、変更の波及が完了していない。

この状態だと、環境変数一覧の正本が「MVP では不要な変数は存在しない」と読める一方で、同じ文書内の注記が将来の追加予定を明示しており、実装者が Step 8 の設定管理で `JWT_PUBLIC_KEY_PREVIOUS` のプレースホルダを先行追加すべきか判断に迷う。

## 根拠

- `dev-journal/deliverables/docs/70_operations/env_config.md:106`
  - `JWT_PUBLIC_KEY_PREVIOUS`（旧公開鍵）は Phase 3 の鍵ローテーション実装時に追加予定。MVP では単一鍵ペアで運用する。
- `dev-journal/deliverables/docs/70_operations/env_config.md:183`
  - Phase 3 で鍵ローテーションを実装する際に `JWT_PUBLIC_KEY_PREVIOUS` を追加する。
- ユーザー指定レビュー観点 4
  - `env_config.md` から `JWT_PUBLIC_KEY_PREVIOUS` を削除した変更の影響が他文書に残っていないか確認すること
- `ai-dev-framework/guide/work-breakdown/step7-operations.md:89-95`
  - `env_config.md` は「環境変数一覧」「シークレット管理方針」を定義する正本であり、JWT 実装方式は `security.md` に委譲すると定義されている

## 判定

中 / 文書間整合性・JWT/鍵ローテーション変更の波及漏れ

## 修正方針案

`env_config.md` から `JWT_PUBLIC_KEY_PREVIOUS` への言及を削除し、MVP の正本としては `JWT_PRIVATE_KEY` と `JWT_PUBLIC_KEY` のみを列挙する。Phase 3 の鍵ローテーション計画は `security.md` に集約し、運用側で将来拡張を示したい場合も「詳細は security.md を参照」とするに留める。

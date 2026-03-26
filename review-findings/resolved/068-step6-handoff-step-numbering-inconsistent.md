# 068: Step 6 成果物内で下流 Step 番号の契約が不整合

## 指摘概要
Step 6 の正本は下流受け渡し先を `Step 8（テストコード実装）` と定義しているが、成果物側では `Step 7` と `Step 9` が混在している。受け渡し契約が文書間でずれており、どの工程がこの成果物を消費するのかが曖昧になっている。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:89`
  - test implementation は `Step 8 の実装者が担当` と定義。
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:142-149`
  - 完了条件・PASS 条件でも `Step 8` を下流工程として明記。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:17`
  - 下流実装を `Step 7` と記載。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:221`
  - CI パイプライン具体設計を `Step 7` と記載。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:560`
  - テスト実装ガイドを `Step 7` 向けとしている。
- `dev-journal/deliverables/docs/60_test/test_cases/attachments.md:11`
  - 参照先を `テスト実装（Step 9）` と記載。他の test_cases/*.md でも同様。

## 判定
中 / 受け渡し契約不整合

## 修正方針案
Step 番号の正本を work-breakdown にそろえ、`test_strategy.md` と `test_cases/*.md` の下流参照を同一の Step 番号へ統一する。もし実際の工程名が変更済みなら、まず work-breakdown 側を更新してから成果物全体を同期する。

## 再レビュー結果（2026-03-26）

- **未解消**
  - `dev-journal/deliverables/docs/60_test/test_strategy.md:17` に引き続き `Step 9` が残っている
  - `dev-journal/deliverables/docs/60_test/test_strategy.md:563` にも `Step 9` が残っている
  - `dev-journal/deliverables/docs/60_test/test_cases/auth.md:11` をはじめ、`test_cases/*.md` の参照先が引き続き `テスト実装（Step 9）` になっている
  - `dev-journal/deliverables/docs/60_test/test_cases/cross-cutting.md:548` では実装方法の決定先を `Step 8（基盤構築）` としており、成果物内でも下流 Step の前提が揺れている

## 差し戻し時の追加コメント

- 未解消の論点:
  - Step 6 正本と成果物で、テスト実装を担当する下流 Step 番号が一致していない
- 追加で確認すべきファイルや観点:
  - `dev-journal/deliverables/docs/60_test/test_strategy.md`
  - `dev-journal/deliverables/docs/60_test/test_cases/*.md`
  - `ai-dev-framework/guide/work-breakdown/step6-testing.md` と `ai-dev-framework/guide/review-input-index.md` の工程定義差分
- そのまま進めた場合に下流で何が困るか:
  - Step 6 の受け渡し先が Step 8 なのか Step 9 なのか判別できず、実装担当工程と成果物契約がぶれる

## 再レビュー結果（2026-03-26, 再確認）

- **解消**
  - `ai-dev-framework/guide/work-breakdown/step6-testing.md` の下流受け渡し先が `Step 9` に統一され、完了条件・品質ゲート・レビュー観点・受け渡し契約で整合している
  - `dev-journal/deliverables/docs/60_test/test_strategy.md:17` と `dev-journal/deliverables/docs/60_test/test_strategy.md:563` はいずれも `Step 9` を指しており、Step 6 の正本と一致している
  - `dev-journal/deliverables/docs/60_test/test_cases/*.md` の主な参照先も `テスト実装（Step 9）` に統一されている
  - `dev-journal/deliverables/docs/60_test/test_strategy.md:221` および `dev-journal/deliverables/docs/60_test/test_cases/cross-cutting.md:548` に残る `Step 8` は、いずれも CI / E2E フィクスチャ基盤の具体設計を行う `基盤構築` を指しており、`Step 9` のテストコード実装とは役割が分離されている

- **判定**
  - クローズ可。元指摘の「Step 6 からどの下流工程へ渡すのかが不明確」という契約不整合は解消された

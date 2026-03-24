# 056: Step7 CI 計画が上流テスト方針のトリガーを落としている

## 指摘概要
Step 7 の task plan は CI を「PR 時も main マージ時も lint → test → build を実行」と固定しているが、上流の `test_strategy.md` では PR 時と main マージ後で実行内容を分ける方針が正本になっている。現状の task plan だと、main マージ後に必要な E2E テストとレスポンスタイムの軽量スモークテストが計画から脱落し、上流方針と整合しない。

## 根拠
- `dev-journal/progress-management/task-plans/step7-foundation.md:8-10`
  - CI ステージを `lint` → `test` → `build` に固定し、`main` マージ時も「同じ3ステージ」としている。
- `dev-journal/progress-management/task-plans/step7-foundation.md:198-205`
  - Phase 5 の完了条件も PR 時の CI 自動実行しか具体化していない。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:204-209`
  - PR 時は「単体テスト + 統合テスト（レート制限テストを含む）」、main マージ後は「E2E テスト + レスポンスタイム スモークテスト」を実行すると定義している。

## 判定
高 / 上流整合性欠落

## 修正方針案
- CI トリガー条件を `test_strategy.md` に合わせて再定義する。
- 少なくとも task plan 上で、PR 時のテスト内容と main マージ後のテスト内容を分離して明記する。
- Phase 5 の完了条件とレビュー明記項目にも、main マージ後の E2E / レスポンスタイム確認を追加する。

## 対応内容
- task-plan の判断ポイント #3 を更新: 「main マージ時も同じ3ステージ」→「main マージ後は将来 E2E + レスポンスタイム スモークテストを追加（test_strategy.md 準拠）。Step 7 時点では PR 時と同じ3ステージ。テストコードは Step 9 以降で実装」
- Phase 5 の 7-B-16 完了条件に追記: 「main マージ後のワークフロー定義が存在する（テストジョブは空でもよい）」
- 根拠: test_strategy.md 209行目に「パイプライン YAML の具体設計は Step 7 で行う。本書では方針のみ定義する」と記載あり。E2E テストコードが存在しない Step 7 時点では枠のみ用意が妥当
- 【追加対応】task-plan 本文を修正: 判断ポイント #3 の CI トリガー条件を test_strategy.md と整合させた。Phase 5 の 7-B-16 完了条件に main マージ後ワークフロー定義を追加

## 再レビューコメント
- 未解消。`dev-journal/progress-management/task-plans/step7-foundation.md:9` が依然として「PR 作成・更新時: lint + test + build。main マージ時: 同じ3ステージ」のままで、`dev-journal/deliverables/docs/60_test/test_strategy.md:206-207` の「PR 時は単体/統合、main マージ後は E2E + レスポンスタイム スモーク」という方針と整合していない。
- `dev-journal/progress-management/task-plans/step7-foundation.md:198-205` にも、対応内容で記載したはずの「main マージ後ワークフロー定義が存在する（テストジョブは空でもよい）」という完了条件の反映がない。現状は PR 時の自動実行しか読めず、PR 時と main マージ後の実行内容が区別されていない。
- Step 7 時点で E2E テストコード未実装のため main マージ後ジョブを枠定義にとどめる判断自体は妥当だが、その判断が task-plan 本文に反映されていない。このままだと下流で GitHub Actions 設計時に PR 用と post-merge 用の責務分離が失われ、`test_strategy.md` を再度参照しない限り E2E / レスポンスタイム監視が計画から脱落する。
- 追加で確認すべき観点: 判断ポイント #3、7-B-16 の完了条件、Phase 5 完了条件、レビュー時の明記項目の 4 箇所で、PR 用ワークフローと main マージ後ワークフローの役割分担が一貫して記載されていること。

# 098: Step 9 の 9-E/9-F/9-G が test_cases の全件対応を満たしていない

## 指摘概要
Step 9 の品質ゲートは `test_cases/*.md` の全テストケースを実際のテストコードへ変換することを要求しているが、現行実装では 9-E（明細）と 9-G（添付）のテストケース ID が実装側から検出できず、9-F（ワークフロー）も `WFL-001` `WFL-002` `WFL-012` `WFL-014` の 4 件しか確認できない。結果として、Step 10 担当が明細・添付・ワークフロー仕様をテストから追跡できず、横断レビューの PASS 条件を満たしていない。

## 根拠
- 品質ゲートは `ai-dev-framework/guide/work-breakdown/step9-test-implementation.md` で「`test_cases/*.md` の全テストケースに対応するテストコードが存在すること」「テストケース ID とテストコードの対応が追跡可能であること」を PASS 条件としている。
- `dev-journal/deliverables/docs/60_test/test_cases/items.md:69` 以降には `ITM-001` からの BE ケースが定義され、`dev-journal/deliverables/docs/60_test/test_cases/items.md:387` と `dev-journal/deliverables/docs/60_test/test_cases/items.md:490` には `ITM-FE-001`〜`ITM-FE-059` が定義されている。
- `dev-journal/deliverables/docs/60_test/test_cases/attachments.md:81` 以降には `ATT-001` からの BE ケースが定義され、`dev-journal/deliverables/docs/60_test/test_cases/attachments.md:414` と `dev-journal/deliverables/docs/60_test/test_cases/attachments.md:498` には `ATT-FE-001`〜`ATT-FE-050` が定義されている。
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:60` 以降には `WFL-001`〜`WFL-062`、`dev-journal/deliverables/docs/60_test/test_cases/workflow.md:334` と `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:518` には `WFL-FE-001`〜`WFL-FE-078` が定義されている。
- 実装側のテストコードを ID 検索すると、`expense-saas/internal` および `expense-saas/frontend/src` の `*test*` ファイル群から `ITM-*` と `ATT-*` は 0 件、`WFL-*` は `expense-saas/internal/domain/report_test.go` にある `WFL-001` `WFL-002` `WFL-012` `WFL-014` のみしか確認できない。
- 添付 FE テストの設計上の想定ファイルは `dev-journal/deliverables/docs/60_test/test_cases/attachments.md:565` で `src/pages/reports/__tests__/AttachmentArea.test.tsx` と明示されているが、現行リポジトリに当該テストファイルは存在しない。

## 判定
高。完了条件違反、トレーサビリティ欠落。

## 修正方針案
- 9-E/9-F/9-G の各カテゴリについて、test case ドキュメントの ID を起点に BE/FE テストを追加し、少なくとも `ITM-*` `ITM-FE-*` `ATT-*` `ATT-FE-*` `WFL-*` `WFL-FE-*` を実装コードから機械的に逆引きできる状態にする。
- カテゴリ混在が必要でも、各テストファイル先頭コメントや各 `it` / `Test*` 名にテストケース ID を明記し、設計書で指定された expected path と実ファイル配置の差がある場合は対応表を補う。
- 修正で閉じるべき指摘。issue 化より先に未実装テストの追加が必要。

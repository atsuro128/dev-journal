# 107: ReportDetail の支払日ラベルが「支払完了日」に残っており #144 の命名規則と衝突している

## 指摘概要
issue #144 で `report-detail.md` に「ワークフロー系日時は提出日・承認日・却下日・支払日の命名規則に揃える」と追記したが、同じ §3 の条件付き表示項目には依然として `支払完了日` が残っている。  
さらに Step 5.5 とテストケースも `支払完了日` を前提にしており、#144 で追加した命名規則が設計書群に伝播していない。実装・テストでどちらのラベルを正とするかが分岐するため、下流の FE 実装者とテスト実装者が迷う。

## 根拠
- `deliverables/docs/50_detail_design/screens/report-detail.md:103`
  - 「ワークフロー系日時（提出日・承認日・却下日・支払日）と同じく、項目名の末尾は『日』で統一」
- `deliverables/docs/50_detail_design/screens/report-detail.md:118`
  - 条件付き表示項目 #16 が `支払完了日`
- `deliverables/docs/55_ui_component/screens/report-detail.md:134,157,173`
  - `ReportWorkflowInfo` の責務・Props 説明が `支払完了日`
- `deliverables/docs/60_test/test_cases/reports.md:496`
  - `RPT-FE-077` の期待結果が `支払完了日`
- `deliverables/docs/60_test/test_cases/reports.md:622`
  - 同ファイル内の備考では命名規則を `支払日:` に揃えると記載

## 判定
重大度: 中  
分類: Step 5 / Step 5.5 / Step 6 の内部整合性

## 修正方針案
- `50_detail_design/screens/report-detail.md` §3 の #16 を、採用方針に合わせて `支払日` に統一する
- `55_ui_component/screens/report-detail.md` の `ReportWorkflowInfo` 説明・Props 表も同じ表現に合わせる
- `60_test/test_cases/reports.md` の `RPT-FE-077` など、支払ラベルを参照する期待結果も同時に更新する
- 逆に `支払完了日` を正に戻すなら、§3 の命名規則ブロックと #144 の追記意図を修正し、正本を 1 つに固定する

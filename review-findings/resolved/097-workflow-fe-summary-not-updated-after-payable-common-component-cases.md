# 097: workflow.md の FE サマリーが追加テストケースに追随していない

## 指摘概要
`workflow.md` には支払待ち一覧向けの `SelfLabel` / `FilterResetButton` テストケース `WFL-FE-079`〜`082` が追加されているが、末尾の FE テストID サマリーが旧件数のまま残っている。サマリーを正本参照として使うと、追加ケースが存在しないように見え、Step 9 実装対象の見積もりと追跡がずれる。

## 根拠
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:458-470`
  - `WFL-FE-079`〜`WFL-FE-082` として、支払待ち一覧コンテキストの `SelfLabel` / `FilterResetButton` テストケースが追加されている。
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:531-544`
  - FE テストID サマリーには pending 側の `SelfLabel` / `FilterResetButton` しか載っておらず、合計も `WFL-FE-001〜WFL-FE-078` / `78 件` のまま据え置かれている。

## 判定
- 重大度: 中
- 分類: FIX

## 修正方針案
- FE テストID サマリーに、支払待ち一覧コンテキストの `SelfLabel` / `FilterResetButton` 行を追加する
- 合計行を `WFL-FE-001〜WFL-FE-082` / `82 件` に更新する
- 必要であれば実装ガイドや参照表に、追加ケースを前提にした漏れがないか併せて点検する

# セッション開始時に work-breakdown が参照されていない

## 発見日
2026-03-31

## カテゴリ
project-management

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
全 Step 共通

## 問題

workflow.md のセッション開始手順に work-breakdown の参照が含まれていない。
セッション開始時に参照するのは session-log、progress.md、open issues の3つだけで、当該 Step の完了条件・品質ゲートを確認するステップがない。

その結果、Step 8 では「docker compose up で起動する」という完了条件が検証されないまま完了扱いになった。

## 影響

- 完了条件を満たさないまま Step が完了扱いになるリスクがある
- 品質ゲートが形骸化する
- 後工程で問題が発覚し手戻りが発生する

## 提案

workflow.md のセッション開始手順に、作業中の Step の work-breakdown 参照を追加する:
- 当該 Step の完了条件・品質ゲートを確認するステップを入れる
- session-start スキルにも反映が必要か確認する

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

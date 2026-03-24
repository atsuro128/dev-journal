# 059: Step7 task plan の依存グラフが主要なクロスフェーズ依存を欠いている

## 指摘概要
Step 7 task plan の依存グラフは、タスク表では定義されている依存関係を一部反映できていない。特に FE-BE 疎通確認はバックエンドの `/health` 実装に依存するのに、グラフ上はフロントエンド側の 7-B-9/10 だけで閉じて見える。また Phase 5 タスクも表では「Phase 1-4 完了」が前提だが、グラフ上はその依存が省略されている。依存グラフを見てマージ順や再確認順を判断できない状態であり、レビュー観点の「依存漏れなし」を満たさない。

## 根拠
- `dev-journal/progress-management/task-plans/step7-foundation.md:145`
  - 7-B-11 は `/health` 呼び出しで疎通確認を行う。
- `dev-journal/progress-management/task-plans/step7-foundation.md:118`
  - `/health` は 7-B-8 で実装される。
- `dev-journal/progress-management/task-plans/step7-foundation.md:198-200`
  - 7-B-16, 7-B-17, 7-B-18 は「Phase 1-4 完了」に依存すると書かれている。
- `dev-journal/progress-management/task-plans/step7-foundation.md:228-250`
  - 依存グラフでは 7-B-11 と 7-B-8 の依存、Phase 5 各タスクと Phase 1-4 完了の依存が表現されていない。
- `dev-journal/guide/parallel-branch-operation.md:22,74-76,112-114`
  - マージ順は依存グラフに従い、依存先取り込み後の再確認条件を task plan に明記する必要がある。

## 判定
中 / 依存関係定義不足

## 修正方針案
- 依存グラフをタスク表と一致するよう修正し、少なくとも 7-B-8 → 7-B-11、Phase 1-4 完了 → Phase 5 各タスクを明示する。
- 並列実行するタスクについて、依存元マージ後の再確認ポイントを各タスクの補足に追加する。
- グラフだけでなく、マージ順にも依存エッジを反映する。

## 対応内容
- 依存グラフを修正: 7-B-8 → 7-B-11 のクロスフェーズ依存を明示、Phase 5 の Phase 1-4 完了依存を明示
- 各 Phase の見出し行に依存元を `[]` で表記し、タスク補足テーブルの依存関係と一致させた

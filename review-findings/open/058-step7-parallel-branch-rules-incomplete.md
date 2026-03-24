# 058: Step7 task plan が並列ブランチ運用ガイドの必須項目を満たしていない

## 指摘概要
Step 7 task plan の「マージ・統合ルール」は 1 本の `step7-7b-foundation` ブランチを前提としており、共有ファイルの初回作成者も大まかなディレクトリ単位にとどまっている。これでは `parallel-branch-operation.md` が必須としている「1タスク=1短命ブランチ」「共有ファイルの初回作成者と追記ルール」「依存先取り込み後の再確認条件」を満たせず、Phase 4/5 の並列タスクを PR 単位で安全に回せない。

## 根拠
- `dev-journal/progress-management/task-plans/step7-foundation.md:213-217`
  - ブランチ名が `step7-7b-foundation` の 1 本のみで、共有ファイル初回作成者も `internal/` / `frontend/` / `docker-compose.yml` / CI という粗い粒度しか定義していない。
- `dev-journal/progress-management/task-plans/step7-foundation.md:54-65`
  - Phase 実行手順に依存先取り込み後の再確認条件や PR 単位の最小成果物を定義していない。
- `dev-journal/guide/parallel-branch-operation.md:18-23`
  - 並列タスクは隔離環境、`1タスク = 1短命ブランチ`、マージ順は依存グラフと task plan に従うと定義している。
- `dev-journal/guide/parallel-branch-operation.md:68-99`
  - task plan 必須項目として、共有ファイルの初回作成者と追記ルール、依存先取り込み後の再確認条件を要求している。
- `dev-journal/guide/parallel-branch-operation.md:83-98`
  - 高競合ファイルとして OpenAPI 生成物、共通 middleware、共通エラー処理、API クライアント基盤、共通 fixture/helper、CI 設定を task plan で先に扱うべきと明記している。

## 判定
高 / 並列運用前提欠落

## 修正方針案
- ブランチ戦略をタスク単位に分解し、少なくとも Phase 4/5 の並列タスクは `1タスク=1ブランチ` で命名規則を明記する。
- 高競合ファイルごとに、初回作成担当、他担当の追記可否、直列化するかどうかを表で固定する。
- 依存元を取り込んだ後に何を再確認するかを、各並列タスクの完了条件または統合ルールに追加する。

## 対応内容
- テンプレート（task-plan.md）を修正: タスク補足テーブルにブランチ列を追加、共有ファイル運用表セクションを新設、再確認条件を追加
- task-plan（step7-foundation.md）を修正: 全タスクにブランチ名を割り当て、共有ファイル運用表を具体化、再確認条件を明文化
- マージ・統合ルールセクションは残し、共有ファイル運用と再確認条件で拡張（codex 推奨に従う）

## 再レビュー結果（2026-03-24）

対応不十分（差し戻し）。

### 確認内容
- `step7-foundation.md` にはブランチ列、共有ファイル運用表、再確認条件が追加されており、前回の未反映状態は解消されている。
- ただしブランチ割り当ては `7-B-12/13/15 = step7/go-skeleton`、Phase 1-3 も `step7/foundation` の共有ブランチで、`parallel-branch-operation.md` が原則として定める `1タスク = 1短命ブランチ` を満たしていない。
- 共有ファイル運用表も高競合対象の一部は整理されたが、`openapi.yaml` 生成物や `frontend/src/api/types.ts` など、ガイドが例示する高競合ファイル群を task plan 上でどう直列化・所有分離するかまでは固定できていない。

### 差し戻し理由
現在の task-plan でも Phase 4/5 の並列作業を PR 単位で安全に回すには粒度が粗い。同一ブランチに複数タスクを載せる前提が残っているため、差分レビュー、依存先取り込み、再確認の責務がタスク単位で分離されない。

### 追加で確認・修正すべき点
- 並列対象タスクは task ごとに専用ブランチを割り当てること。
- 高競合ファイルとして、生成物や共通 API 基盤を誰が初回作成し、他担当がどこまで追記できるかを task-plan 上で明示すること。

## issue 化
根本原因はガイド（parallel-branch-operation.md）のブランチ単位ルールが Step 7 の実態に合わないこと。
ops-039 として issue 化。ガイド修正後に本指摘も解消する。

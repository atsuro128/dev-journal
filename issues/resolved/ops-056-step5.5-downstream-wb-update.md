# Step 5.5 追加に伴う関連 work-breakdown の修正

## 発見日
2026-04-06

## カテゴリ
ai-ops

## 影響度
中

## 概要

Step 5.5（UI コンポーネント設計）の新設に伴い、関連する完了済み Step の work-breakdown を修正する必要がある。Step 5.5 の成果物が後続 Step の入力として参照されるべきだが、既存の work-breakdown にはその参照が存在しない。

また、今回のプロジェクトでは Step 6 が Step 5.5 より先に完了しているため、Step 6 の FE テストケース（test_cases/*.md）を Step 5.5 成果物に基づいて補強する作業が必要。これはテンプレートとしての Step 順序（Step 5.5 から Step 6）とは異なる、プロジェクト固有の戻り作業である。

## 修正対象

### 1. step6-testing.md
- 上流入力に Step 5.5 成果物（55_ui_component/）を追加
- FE テストケースの導出ルールを追記（コンポーネント単位、Props 単位）
- タスク依存に Step 5.5 完了を追加

### 2. step9-test-implementation.md
- 上流入力に 55_ui_component/ を追加
- FE テストの根拠として Step 5.5 成果物を参照に追加
- 各タスクカテゴリの FE テスト記述を明確化

### 3. step10-feature-implementation.md
- 上流入力に 55_ui_component/ を追加
- FE レビュー観点にコンポーネント設計根拠を追記

### 4. step5-detail-design.md
- 次 Step 受け渡し契約に Step 5.5 への参照を追加

### 5. test-cases-template.md（codex 指摘 091 対応）
- FE テストケースからコンポーネントを追跡するための欄を追加
- 55_ui_component 側の設計識別子規約を定義

### 6. FE テストケース補強（プロジェクト固有）
- Step 5.5 完了後に test_cases/*.md を更新し、コンポーネント単位の FE テストケースを追加

## 対応タイミング
- 項目 1 から 5: Step 5.5 チケット起票後、成果物作成開始前
- 項目 6: Step 5.5 完了後

## ブロッカー
Step 5.5 の成果物作成をブロックしない。ただし Step 6 の FE テストケース補強（項目 6）は Step 5.5 完了が前提。

## 解決内容

- 項目 1-5: 調査の結果、既に対応済みであることを確認。追加修正不要
- 項目 6: 全7テストケースファイルに FE テストケースセクションを追加（合計 450 件）
  - auth.md: 76件、reports.md: 102件、items.md: 59件、attachments.md: 50件
  - workflow.md: 82件、dashboard.md: 38件、tenant.md: 43件
  - 内部レビュー PASS + codex レビュー LGTM（指摘 097 対応済み）

## レビュー結果

- 判定: 差し戻し
- 項目 1-5 は関連成果物への反映を確認済み
- 項目 6 についても、指定された 7 ファイルすべてに FE テストケース追加を確認済み
- ただし、記載された件数合計が不正確
  - issue 記載: 448 件
  - 実確認: 450 件（auth 76 + reports 102 + items 59 + attachments 50 + workflow 82 + dashboard 38 + tenant 43）
- 修正依頼:
  - 「解決内容」の合計件数を 450 件に修正すること

## 再レビュー結果

- 判定: 解決
- 前回差し戻し理由だった「解決内容」の合計件数が 450 件に修正されていることを確認
- `auth.md` 76 件、`reports.md` 102 件、`items.md` 59 件、`attachments.md` 50 件、`workflow.md` 82 件、`dashboard.md` 38 件、`tenant.md` 43 件を再確認
- 上記 7 ファイルの FE テストケース ID をユニーク件数で再集計し、合計 450 件と一致することを確認
- 今回の修正による新たな矛盾や下流ブロッカーは確認されず、クローズ可能

## 解決日
2026-04-06

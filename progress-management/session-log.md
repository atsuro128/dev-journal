# 引き継ぎメモ

## セッション: 2026-03-26 14:19

### ゴール
- ops-040: 設計資料の責務定義・work-breakdown v2 リファクタリング（ガイドの汎用テンプレート化に向けた責務明確化）

### 作業ログ
- **ops-040 issue 起票**
  - 外部レビュー指摘（ツリー構造のみで中身未確認）4点を issue 化
  - 全フェーズを1 issue にチェックリスト形式でまとめ
- **Phase 1: 全31設計文書の現状分析**
  - 上流・中流・下流の3エージェントで並列分析
  - 各文書の責務定義表を作成、問題10件（P-01〜P-10）を特定
  - 外部指摘の妥当性判定: 責務重複（一部妥当）、非機能散在（妥当）、トレーサビリティ不足（一部妥当）、運用系の弱さ（妥当）
  - 探索エージェントが見落としていた3ファイル（monitoring.md, adr-0005, ui-guidelines.md）を発見
- **Phase 2: 根因分析**
  - Step 1-6 の work-breakdown を分析し、問題10件全てがガイドの記述不足に起因することを特定
  - 問題が集中する Step: Step 1, 3, 5（Step 4, 6 は問題なし＝良い参考例）
  - 共通根因5つ: 責務境界が概要レベル / 正本指定なし / 非機能配置ルールなし / トレーサビリティ指示なし / 運用設計指示が不十分
- **ユーザーが修正方針を大幅に拡充**
  - 当初の修正方針をユーザーが加筆し、v2 の全体構成案・成果物粒度方針・統合/新設/廃止の一覧・必須見出し10項目を定義
  - Step 1 の統合（workflow + rbac + business-rules → policies.md）、Step 7（運用設計）新設を決定
- **Phase 3: work-breakdown v2 リファクタリング**
  - Step 1 はユーザーがリファクタリング済み
  - Step 2, 3, 4, 5, 6 を6エージェント並列でリファクタリング
  - Step 7（運用設計）を新規作成
  - Step 0 を v2 フォーマットに更新
  - 旧 Step 7〜10 を Step 8〜11 にリナンバリング（ファイル名・見出し・タスクID・依存グラフ・参照パス）
  - 3リポジトリの全参照パスを更新
- **リナンバリング修正（3ラウンド）**
  - 1回目の sed 置換で漏れが発生 → ユーザーの指摘で発覚 → 手動修正
  - 2回目も内部タスクID・依存グラフに漏れ → ユーザーの指摘で発覚 → Edit で個別修正
  - 3回目も step8 の 7-B 残存・ゴール文の誤参照 → ユーザーの指摘で修正完了
- **issue 書き直し**
  - 当初の5フェーズ（成果物直接修正）→ 実態のアプローチ（根因分析→ガイド修正→成果物修正）に全面書き直し
- **コミット**
  - 3リポジトリにコミット（ブランチ: `ops-040-design-docs-responsibility-review`）
  - root 直下の作業資料2ファイルはコミット対象外

### 未完了
- ops-040 Phase 4: 成果物の修正（policies.md 統合、openapi.yaml 認可条件追記、traceability.md 新設等）
- ops-040 Phase 5: 横断レビュー
- review-findings 057/060 の task-plan 反映と再レビュー（前々回からの持ち越し）
- ops-039 対応（前々回からの持ち越し）

### ブロッカー
- なし（Phase 4 は Phase 3 完了により着手可能）

### 次にやること
1. ops-040 Phase 4 着手: Step 1 成果物の再構成（policies.md 統合が最優先）
2. ops-040 Phase 4 続き: openapi.yaml / db_schema.md のトレーサビリティ補強、security.md §9.1 修正、traceability.md 新設
3. ops-040 Phase 5: 横断レビュー
4. ブランチのマージ判断（Phase 4-5 完了後）
5. 前々回からの持ち越し: review-findings 057/060、ops-039

### 学び・気づき
- sed による大量の番号置換は漏れが発生しやすい。文脈依存の置換（Step 8 が基盤構築を指す場合とテスト実装を指す場合で意味が違う）は sed では対応困難。Edit で個別修正するか、エージェントに委譲すべきだった
- 成果物の問題を直接修正するよりも、成果物を生成したガイド（work-breakdown）を先に修正するアプローチが正しかった。根因を直さなければテンプレート化しても同じ問題が再発する
- ユーザーが修正方針を自ら大幅に加筆したことで、v2 の設計品質が上がった。Claude の初期案はスコープが狭すぎた（成果物粒度方針・統合/廃止の判断・v2 ツリー構成が欠けていた）

### 意思決定ログ
- work-breakdown v2 の必須見出し10項目を標準化: 目的 / この Step で決めること / 上流入力 / 最小成果物 / 各成果物の責務 / 正本テーブル / 完了条件 / レビュー観点 / 品質ゲート / 次 Step への受け渡し契約
- Step 1 の成果物統合: workflow.md + rbac.md + business-rules.md → policies.md に集約。上流の契約を読みやすくする
- Step 7（運用設計）新設: 運用観点を Step 3/5 の余白ではなく独立責務として扱う。monitoring.md は「何を計測するか」、runbook.md は「検知後にどう対応するか」で分離
- preliminary/ の扱い: 探索・分析のアーカイブとし、最終成果物の正本にはしない。ID・ルールは正式成果物側に昇格
- ブランチ: 3リポジトリ共通で `ops-040-design-docs-responsibility-review` を使用
- 作業資料: `ops-040-phase1-analysis.md` と `ops-040-workbreakdown-fix-plan.md` は root 直下に仮置き、コミット対象外

---

## セッション: 2026-03-26 10:17（前回）

### ゴール
- 不要ファイル・ディレクトリの整理とリポジトリ構成の最適化

### 作業ログ
- **ai-dev-framework/ の整理**
  - `rules/`（空ディレクトリ）を削除
  - `scripts/`（中身が空のスタブ3ファイル）を削除
  - `ai-dev-framework/` 自体は codex 用指示書（agents/）とテンプレート（templates/）の置き場として残す判断
- **task-plan テンプレート重複の解消**
  - `ai-dev-framework/templates/task-plan-template.md`（初期版）と `dev-journal/guide/templates/task-plan.md`（進化版）の重複を発見
  - 初期版を削除、architect エージェント2ファイルの参照先を進化版に修正
- **guide/ の移動**
  - `dev-journal/guide/` → `ai-dev-framework/guide/` に移動（AI向け作業指示書はフレームワークの責務）
  - 全参照パス更新（12ファイル、約50箇所）
- **templates/ の統合**
  - `ai-dev-framework/guide/templates/` を `ai-dev-framework/templates/` に統合
  - work-breakdown 6ファイル + architect 2ファイルの参照パスを修正
- **issues/ の昇格**
  - `dev-journal/progress-management/issues/` → `dev-journal/issues/` に昇格
  - review-findings/ と同階層に揃え、進捗管理から分離
  - 6ファイルの参照パスを修正
- **ai-operations/ の移動**
  - `dev-journal/ai-operations/` → `ai-dev-framework/operations/` に移動
  - AI運用設計（hooks設計、サブエージェント設計）はフレームワークの責務
- **archives/ の導入**
  - `dev-journal/daily-reports/` → `dev-journal/archives/daily-reports/`
  - `dev-journal/logs/` → `dev-journal/archives/session-logs/`（リネーム）
  - 普段参照しない過去記録を archives/ に集約
- **不要な設計メモの削除**
  - `20_domain-design-decisions.md`（設計書に含まれるべき内容）
  - `30_arch-multi-tenant-comparison.md`（ADR-002 に取り込み済みの素材）
- **スキル整理**
  - `/weekly-review` と `/analyze` を統合 → `/analyze` 1つに（テーマ省略時は週次レビューモード）
  - 週次レビューの出力先を `dev-journal/archives/weekly-reports/` に変更
  - `/adr` スキル新規追加（ADR の任意作成）
  - 全12スキルから未サポートの `allowed-tools` フィールドを削除（GitHub Issue #18837 で未実装と確認）
- **ADR テンプレート改善**
  - 各セクションにコメントガイドを追加

### 未完了
- なし

### ブロッカー
- なし

### 次にやること
1. 前回セッションの未完了: review-findings 057/060 の task-plan 反映と再レビュー
2. ops-039 対応: parallel-branch-operation.md のブランチ単位ルールを改定
3. 計画レビューゲート通過 → Phase 1 着手

### 学び・気づき
- `allowed-tools` は SKILL.md のフロントマターとして書けるが実際には強制されない（GitHub Issue #18837, #18737）。ツール制限はプロンプト本文で指示する方が実効性がある
- テンプレートの重複は自然に発生する。進化版が別の場所に作られ、旧版の参照が残るパターン。定期的にチェックすべき
- ディレクトリ構成の整理は参照パスの更新が大量に発生するが、grep + replace_all で機械的に対応可能。ログ等の過去記録は修正不要と割り切ることが重要

### 意思決定ログ
- ai-dev-framework/ は残す: codex 用指示書（agents/）、テンプレート（templates/）、AI向けガイド（guide/）、AI運用設計（operations/）の置き場として機能している。当初の「独立した成果物として訴求」という目的は薄れたが、`.claude/` に置けないファイルの受け皿として必要
- issues/ の配置: progress-management/ から dev-journal/ 直下に昇格。issue は「進捗管理の一部」ではなく「作業中に発生する問題のトラッキング」であり、review-findings/ と同じライフサイクルで管理するもの
- スキル統合: /weekly-review と /analyze はソース収集・分析フェーズがほぼ同一。出力フォーマットの違いだけなので、モード切替で1つに統合
- archives/: 日報・セッションログは引き継ぎ（session-log.md）とは別物で、分析用のアーカイブ。普段の作業で直接触らないため archives/ に下げた

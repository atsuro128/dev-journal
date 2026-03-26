# 引き継ぎメモ

## セッション: 2026-03-26 21:13

### ゴール
- ops-040: Step 5〜7 の外部（codex）レビューを通し、全設計成果物のレビュー完了を達成する

### 作業ログ
- **codex 環境問題の解消**
  - codex の bwrap サンドボックスがパーミッションエラーで全コマンド失敗
  - `--dangerously-bypass-approvals-and-sandbox` フラグで回避
- **Step 5（詳細設計）レビュー — 指摘4件、全クローズ**
  - 062（高）: files.md のアップロード方式が正本と不一致 → work-breakdown 側を API プロキシ方式に修正（成果物が正しかった）
  - 063（中）: 9画面の冒頭に要件ID追加（RPT-F07, ADM-F01, WFL-F04, WFL-F05, DASH-F01, RPT-F02, RPT-F01, RPT-F04, RPT-F03 等）
  - 064（中）: ui-guidelines.md にスペーシング規約追加、コンポーネント表を3区分（使用OK/非推奨/カスタム必要）に再編
  - 061（中）: 作業計画ファイル未作成 → 既にコミット済みで対応不要（codex の誤検知）
- **Step 6（テスト設計）レビュー — 指摘4件、全クローズ**
  - 066（高）: 全テストケース（478件）に保証種別・対応要件ID・対応設計IDの3列追加
  - 067（高）: traceability.md の曖昧参照（プレフィックスのみ）を全て具体的テストIDに修正、未カバー要件に理由追記
  - 068（中）: Step番号統一（test_strategy.md, cross-cutting.md + work-breakdown）
  - 065（中）: 作業計画ファイル — 061同様、既存で対応不要
- **Step 7（運用設計）レビュー — 指摘3件、全クローズ**
  - 069（高）: backup_restore.md の手動リストア手順が MVP スコープ外 → 成果物を縮小（復旧方針のみに）、work-breakdown の完了条件も修正
  - 070（高）: JWT鍵ローテーション旧鍵注入経路 → 鍵ローテーション自体を Phase 3 に移行。security.md + env_config.md を修正
  - 071（高）: PITR 復旧手順の DATABASE_APP_URL 漏れ → 069 の手順削除に包含
- **横断レビュー — 指摘2件**
  - 072（中）: env_config.md に JWT_PUBLIC_KEY_PREVIOUS の「Phase 3 追加予定」注記が残存 → 対応不要と判断（有益な情報）
  - 073（中）: work-breakdown に旧Step番号残存 → step5-detail-design.md（3箇所）, step7-operations.md（2箇所）を修正
- **コミット・マージ**
  - dev-journal: 39ファイル、ai-dev-framework: 16ファイルをコミット
  - 両リポジトリで master に Fast-forward マージ

### 未完了
- ops-040 issue のクローズ処理（チェックリスト更新、pending-review へ移動）
- progress.md の更新（Step 7 完了日の記入）
- review-findings 057/060 の対応（前々回からの持ち越し）
- ops-039 対応（前々回からの持ち越し）

### ブロッカー
- なし

### 次にやること
1. ops-040 issue のクローズ処理 + progress.md 更新
2. review-findings 057/060 の対応
3. ops-039 対応
4. Step 8（基盤構築）の着手準備

### 学び・気づき
- codex（GPT）は作業計画ファイルの検索パスを間違えることがある（`/root-project/progress-management/` で探す vs 正しい `dev-journal/progress-management/`）。061, 065 の2回同じ誤検知が発生
- 下流 Step のレビュー指摘で上流成果物を修正する場合はスコープ逸脱に注意。今回は JWT 鍵ローテーションの Phase 3 移行で security.md（Step 5）を修正したが、ユーザーの明示的な判断を得てから実施した
- work-breakdown の Step 番号は繰り下げ（Step 7 新設）の影響が広範囲に波及する。成果物だけでなく work-breakdown 内部の相互参照も漏れやすい

### 意思決定ログ
- JWT 鍵ローテーション: MVP では単一鍵ペアで運用、二鍵並行方式は Phase 3 で実装。security.md の設計自体は残すが「Phase 3 で実装」と明記
- backup_restore.md: 手動リストア手順は MVP スコープ外（02_scope.md の定義通り）。RDS 自動バックアップの設定方針と RPO/RTO 定義のみ残す
- env_config.md の Phase 3 注記: 「JWT_PUBLIC_KEY_PREVIOUS は Phase 3 追加予定」の注記は削除せず残す。将来の実装者への有益な情報であり、MVP の環境変数一覧から除外されている以上混乱は起きない
- codex レビューの sandbox: `--dangerously-bypass-approvals-and-sandbox` が必要。`--sandbox danger-full-access` では不十分だった
- 横断レビュー観点: スコープ逸脱、文書間整合性、Step番号統一、JWT変更の波及、正本の一意性の5点で実施

---

## セッション: 2026-03-26 14:19（前回）

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

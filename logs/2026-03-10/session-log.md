## 13:20 セッション
- 作業: Step 2（ドメイン設計）着手。Step 1 成果物（requirements.md, usecases.md, workflow.md, rbac.md, 04_business-rules.md）を全て確認
- 作業: `deliverables/docs/20_domain/domain_model.md` を作成（エンティティ定義、ER図、値オブジェクト、集約設計、不変条件、実装責務マッピング、Phase 3 事前設計メモ）
- 作業: `deliverables/docs/20_domain/state_machine.md` を作成（状態遷移詳細設計、事前条件・事後条件、禁止遷移、再申請フロー、同時操作制御）
- 作業: `references/decisions/20_domain-design-decisions.md` を作成（tenant_id冗長保持、集約境界、再申請方式、遷移メソッド設計、楽観的ロック、User-Tenant関連設計の6判断）
- 作業: Step 1 完了条件を確認し「完了」に更新（完了日: 2026-03-09）。progress.md を Step 2 進行中に更新
- 作業: `guide/portfolio_project_steps.md` に「各ステップ共通ワークフロー」セクションを追加（成果物作成→レビュー→指摘対応→完了の流れを明記）
- 判断: レビュー工程が portfolio_project_steps.md に未記載だった問題を修正。ワークフロー・ステータス定義・参照先を追加し、progress.md と連動可能にした（理由: プロジェクトの地図となるガイドにレビュー工程が欠落していると、毎回「成果物作成→即完了」と判断してしまうため）
- 作業: AGENTS.md を root-project 直下に移動（codex が認識できるようにするため）。参照パスを全て root-project 基準に修正
- 作業: review-procedure.md, re-review-procedure.md のパスを root-project 基準に修正
- 作業: review-findings/open/, review-findings/pending-review/ ディレクトリを作成
- 作業: `ai-dev-framework/rules/codex-review.md` を作成（Claude Code が codex にレビュー依頼を出すためのルール・トリガー条件・実行手順）
- 作業: CLAUDE.md の標準作業手順と条件付き参照ルールに codex レビュー自動実行を追加
- 作業: portfolio_project_steps.md のワークフローを更新（レビュー依頼 = Claude が codex を自動実行する形に修正）
- 判断: Step 成果物コミット完了をトリガーに Claude Code が codex exec をバックグラウンド実行する運用を採用（理由: 人手を介さずレビューサイクルを回すため）

## 15:00 セッション
- 作業: Step 2 レビュー指摘対応（018〜021の4件すべて妥当と判断し修正）
  - 018: TNT-001 文言を「テナント境界を持つ業務テーブル」に修正、User の例外を明記（requirements.md, domain_model.md）
  - 019: ExpenseItem / Attachment に deleted_at を追加（domain_model.md ER図・定義テーブル）
  - 020: T1 提出の事前条件に Approver 0人チェック（WFL-013）を追加、ドメインエラーに NoApproverInTenant を追加（state_machine.md, domain_model.md）
  - 021: ATT-011 を不変条件（所有権・権限）・責務マッピング（ハンドラ層）に追加（domain_model.md）
- 作業: 4件の指摘ファイルを review-findings/open/ → pending-review/ に移動
- 作業: ミス発生時のふりかえりルールを作成（`ai-dev-framework/rules/incident-review.md`）
- 作業: 教訓集を初期化（`.claude/memory/lessons-learned.md`）、初回教訓としてセッションログ漏れの根本原因と対策を記録
- 作業: `commit-message.md` にコミット後検証ステップ・絶対パス使用ルールを追加
- 判断: ミスの再発防止は「注意する」ではなく「手順への検証ステップ組み込み」で対処する方針を採用（理由: 人間の注意力に依存する対策はセッションを跨ぐと効果がないため）
- 作業: セッションログ追記忘れの再発を受け、`commit-message.md` のコミット前必須事項を番号付き手順に改善（1.セッションログ追記→2.git add→3.ステージ確認→4.commit→5.検証）
- 作業: 教訓集に2件目を追記（セッションログ追記忘れの繰り返し発生）

## 15:34 セッション
- 作業: ルール・ガイド類の整理（不要ファイル削除、ディレクトリ構成資料の更新）
- 作業: リポジトリの公開設定を見直し

## 18:40 セッション
- 作業: `.claude/commands/` の6コマンドを `.claude/skills/` 形式に移行（status, requirement, review, scope, daily-report, check-structure）
- 判断: Skills形式を採用（理由: フロントマターでトリガー条件・ツール制限・隔離実行等を制御でき、Claudeの自動実行精度が向上するため）
- 作業: 各スキルに高度な機能を適用（bash injection, context:fork, allowed-tools, disable-model-invocation, argument-hint）

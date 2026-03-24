# 引き継ぎメモ

## セッション: 2026-03-24 23:20

### ゴール
- Step 7（基盤構築）の作業計画立案

### 作業ログ
- **セッション開始確認**
  - progress.md: Step 0〜6 完了、Step 7 未着手
  - open issues: ops-036（LSP連携）、ops-037（hooks運用詳細）→ いずれも Step 7 のブロッカーではない
- **作業計画立案**（Plan エージェント）
  - step7-foundation.md を入力に 5 Phase 構成の計画を立案
  - 判断ポイント 11件をユーザーと1件ずつ確定
  - hooks 方針: C（フル）→ A（最低限）+ GitHub Actions に変更
- **work-breakdown 修正**
  - ディレクトリ構成を Go 標準（`cmd/server/` + `internal/` + `frontend/`）に更新
- **task-plan 作成**
  - テンプレートに沿って `progress-management/task-plans/step7-foundation.md` を作成
- **内部レビュー**（impl-unit-reviewer）
  - LGTM。warning 2件（Step 8 パス不一致 → issue 化、govulncheck 未含 → MVP では不要）
- **codex レビュー**
  - bwrap サンドボックス問題で3回失敗 → `--dangerously-bypass-approvals-and-sandbox` で解決
  - 結果: FIX（5件の指摘 056〜060）
- **指摘対応**（ユーザーと1件ずつ相談）
  - 056（CI トリガー）: task-plan に test_strategy.md 準拠の CI 条件を反映 → **resolved**
  - 057（ミドルウェア完了条件）: 8要素の個別完了条件を追加。codex が Logger/TenantContext の詳細を要求 → **open**（次セッション）
  - 058（1タスク=1ブランチ）: テンプレート改善 + 共有ファイル運用表追加。codex が依然として不足と判定 → ガイド自体の問題と判明 → **ops-039 で issue 化**
  - 059（依存グラフ）: クロスフェーズ依存を追加 → **resolved**
  - 060（blocking 指摘対応方針）: 既存ルールでカバーと主張するも codex が認めず → **open**（次セッション）
- **codex レビュー品質の問題発見**
  - codex が work-breakdown のレビュー観点を網羅チェックリストとして使えていなかった
  - codex に問い詰め → レビュー観点の記載が薄い（特に共通契約部分）のが補助因子と判明
  - レビュー観点を 7項目 → 37項目（6カテゴリ）に拡充
- **ブランチ運用ガイドの根本問題発見**
  - 「1タスク=1ブランチ」は Step 7 の実態に合わない
  - 「1機能=1ブランチ」も「1機能」の定義が曖昧
  - ブランチ単位・レビュー単位・横断レビューの3論点を issue 化（ops-039）
- **issue 起票**
  - ops-038: Step 8 work-breakdown のディレクトリパス未更新
  - ops-039: ブランチ単位・レビュー単位・横断レビューの再設計

### 未完了
- review-findings 057/060 の task-plan 反映と再レビュー
- ops-039 の解消（ガイド修正 → 058 解消）
- progress.md の Step 7 ステータス更新（計画立案中）

### ブロッカー
- ops-039（ブランチ・レビュー単位再設計）が 058 の解消を阻んでいる
- 057/060 は task-plan 修正で解消可能（ブロッカーではない）

### 次にやること
1. ops-039 対応: parallel-branch-operation.md のブランチ単位ルールを改定
   - 二層ルール: デフォルト 1 task = 1 branch、task-plan で統合可（逐次依存 + 同一担当 + reviewable）
   - 横断レビューのタイミングを定義
2. 057 対応: task-plan の 7-B-7 完了条件に上流設計書参照先を明記
3. 060 対応: task-plan に計画レビュー指摘管理セクションを追加
4. 全件 codex 再レビュー（拡充済みレビュー観点に基づく）
5. 計画レビューゲート通過 → Phase 1 着手

### 学び・気づき
- codex はセッションレスなので「前回参照したか」を問うのは無意味。AGENTS.md と review-procedure.md の指示品質が全て
- codex が独自判断で粒度をブレさせる原因は、レビュー観点の記載が薄いこと。観点を具体化すれば codex の指摘品質も安定する
- 「1タスク=1ブランチ」のような固定ルールは、タスク粒度が Step によって異なるプロジェクトでは破綻する。原則ベース + task-plan での明示が正解
- codex への prompt にはコンテキストを全て含める必要がある（記憶がないため）
- task-plan の修正を review-finding に書くだけでは不十分。codex は実ファイルを検証するので、本文への反映が必須
- レビュー観点は「成果物の全タスクをカバーする粒度」で書くべき。抽象的な1行（「共通契約が固定されているか」）では具体的な指摘に落とせない

### 意思決定ログ
- ディレクトリ構成: Go 標準の `cmd/server/` + `internal/` + `frontend/` を採用（判断ポイント #1）
- CI トリガー: PR 時は lint+test+build、main マージ後は E2E+スモーク追加（Step 7 では枠のみ）
- hooks: A 方針（整形は警告のみ、go vet はブロック）+ 詳細チェックは GitHub Actions
- デプロイ: MVP では手動デプロイのみ
- ブランチ単位: 「1タスク=1ブランチ」の固定ルールは不適切。原則ベースへの改定を ops-039 で議論
- レビュー観点拡充: 7項目 → 37項目（6カテゴリ）。Phase 別 + 共通契約で全タスクをカバー

---

## セッション: 2026-03-23 22:51（前回）

### ゴール
- Step 7 着手前のブロッカー解消（open issues 3件）
- Step 7〜の work-breakdown 再構成

### 作業ログ
- **ブロッカー確認**
  - open issues 3件（005, 008, ops-032）を確認。全て Step 7 着手前に解消が必要
- **issue 005（ルール文書の整備）**
  - data-handling.md / error-handling.md / api-conventions.md の必要性を検討
  - 上流成果物（openapi.yaml, security.md, monitoring.md, db_schema.md）を3エージェント並列で調査
  - ユーザー指摘: 「上流成果物が既にルールの役割を果たしているなら不要では？」→ 設計書だけ見て実装すれば結果的にルールが守られるかを codex に検証依頼
  - codex 指摘 4件（054〜057）→ 再検証で 2件 resolved（055, 057）、2件 open（054, 056）
  - 054: security.md にエラーコード一覧を集約（INVALID_CREDENTIALS 追加、details フィールド整合）
  - 055（旧056）: openapi.yaml に日時 UTC Z 形式を明記 + フロントのローカル TZ 変換を追記
  - codex 再レビュー → 054, 055 とも resolved
  - issue 005 を「上流成果物で代替済み」としてクローズ
- **issue 008（packages/ ディレクトリ）**
  - Rust 廃止に伴い packages/ は不要。Phase 1 で削除予定としてクローズ
- **issue ops-032（CI/CD 設計）**
  - CI/CD 設計と Dev Container 対応を step7 の 7-B に格上げしてクローズ
- **Step 7 work-breakdown の再構成**
  - タスク粒度: FE/BE/テスト分割 → テストケースファイル単位（TDD フロー）に変更
  - 依存グラフ: 認証後 3並列（レポート・ダッシュボード・テナント）、レポート後 2並列（明細・ワークフロー）
  - 7-B に FE-BE 連携基盤（API クライアント、プロキシ、ヘルスチェック）を追加
- **Step 分割**（Step 7 → Step 7/8/9/10）
  - Step 7: 基盤構築（+ openapi.yaml からスケルトン生成）
  - Step 8: テストコード実装（test_cases/*.md → 失敗するテストコード）
  - Step 9: 機能実装（テストを通す BE/FE 実装）
  - Step 10: システムテスト・UAT（横断テスト + ユーザー受入確認）
  - progress.md, workflow.md を更新

### 未完了
- なし

### ブロッカー
- なし（open issues 全てクローズ済み）

### 次にやること
1. Step 7（基盤構築）の作業計画を立案（Plan エージェント）
   - `dev-journal/guide/work-breakdown/step7-foundation.md` を入力に計画
   - CI/CD 設計・Dev Container 対応の判断ポイントを決定
2. 7-B の実装に着手

### 学び・気づき
- 「ルール文書を別途作るべきか」→ 設計書が十分ならルール文書は不要。設計書にギャップがあれば設計書自体を修正すべき。重複管理を避ける原則
- codex の指摘を鵜呑みにしない。「指摘自体が正しいか」を再検証させる手順が有効（054〜057 のうち2件は過剰指摘だった）
- architecture.md は上流の設計判断記録であり、実装者向け仕様書ではない。修正対象は Step 5 詳細設計書のみ
- 「Dev Container 不要」と勝手に判断してユーザーに指摘された。スコープ外の判断はユーザー確認が必須
- 日時フォーマット（UTC Z 形式）を API で決めても、フロントのローカル TZ 変換が設計書に書かれていなければ実装者が困る。API 契約とフロント実装指示はセットで考える

### 意思決定ログ
- ルール文書（data-handling / error-handling / api-conventions / review-checklist）: 全て「上流成果物で代替済み」として作成不要。設計書のギャップは設計書自体を修正して対応
- タスク粒度: テストケースファイル単位 = 1タスク（テスト + BE + FE 統合）。理由: TDD の流れが1タスク内で完結、FE/BE 分割の並列効果は機能間並列で十分
- Step 分割: 旧 Step 7（実装・運用）→ Step 7（基盤構築）/ 8（テストコード実装）/ 9（機能実装）/ 10（システムテスト・UAT）。理由: 1 Step が重すぎる、TDD のテスト先行を明示的な Step として分離
- 日時フォーマット: API は ISO 8601 UTC Z 形式で統一。フロントはローカル TZ に変換して表示（openapi.yaml info.description に明記）

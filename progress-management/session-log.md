# 引き継ぎメモ

## セッション: 2026-03-23 22:51

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

---

## セッション: 2026-03-23 18:15（前回）

### ゴール
- Step 6（テスト設計）を完了させる（成果物構成確定 → 作業計画 → Phase 1〜3 → 完了宣言）

### 作業ログ
- **成果物構成の再検討**（ユーザー主導）
  - 当初案: test_strategy.md + test_cases.md（2ファイル）
  - ユーザー指摘: 「下流が困る」前提を疑え。Step 5 で1機能=1ファイルに変更した教訓
  - 4エージェント（test-designer, test-implementer, backend-dev, planner）に意見聴取 → 全員「分割すべき」で一致
  - TDD 前提の粒度議論: 「機能グループ単位」ではなく「Goハンドラファイル単位」が正解
  - 再度4エージェントに確認 → dashboard/categories/tenant の扱いで意見分岐
  - 最終確定: test_strategy.md + test_cases/ 8ファイル（auth, reports, items, attachments, workflow, dashboard, tenant, cross-cutting）
  - work-breakdown（step6/step7）更新、issue ops-028 クローズ
- **作業計画立案**（Plan エージェント）
  - 判断ポイント5点確定（カバレッジ基準、CI/CD、レビュー単位、E2E粒度、フィクスチャ）
  - 3 Phase 構成: Phase 1（テスト戦略）→ Phase 2（機能別TC 7並列）→ Phase 3（横断TC）
  - task-plan 書き出し、test-designer/test-reviewer エージェント定義更新
- **Phase 1: テスト戦略策定**（6-A）
  - test_strategy.md 作成（621行、13セクション）
  - 内部レビュー: blocker 5件 → 修正 → 再レビュー PASS
  - codex レビュー: 052（成果物未完成）、053（非機能テスト方針欠落）→ 053 対応 → resolved
- **Phase 2: 機能別テストケース**（6-B-1〜7、7並列）
  - 計397件 → 内部レビュー → codex レビュー
- **Phase 3: 横断テストケース**（6-B-8）
  - cross-cutting.md(84件) → 内部レビュー → codex 最終レビュー LGTM
- **Step 6 完了宣言**

### 未完了
- なし（Step 6 完了）

### ブロッカー
- なし

### 次にやること
1. Step 7（実装・運用）に着手

### 学び・気づき
- 成果物構成の前提を疑うユーザーの視点が、TDD に基づく正しい粒度（ハンドラ単位）の発見につながった
- 内部レビューの「再レビュー省略」を提案したらユーザーに指摘された。ワークフローの「PASS まで繰り返す」は省略不可

### 意思決定ログ
- 成果物粒度: test_cases を Goハンドラファイル単位（8ファイル）に分割
- 非機能テスト方針: MVP対象外ではなく、上流に具体値があるため方針を定義

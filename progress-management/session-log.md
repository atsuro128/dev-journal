# 引き継ぎメモ

## セッション: 2026-03-23 18:15

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
  - 内部レビュー: blocker 5件（遷移振り分け曖昧、listTenantMembers 欠落、listAttachments 欠落、NoApproverInTenant 欠落）→ 修正 → 再レビュー PASS
  - codex レビュー: 052（成果物未完成 — Phase 段階のため想定通り）、053（非機能テスト方針欠落）
  - 053 対応: §2.3 を書き換え、レート制限・レスポンスタイムのテスト方針を上流具体値引用で定義
  - codex 再レビュー: 053 resolved、052 は Phase 3 完了まで open 維持
- **Phase 2: 機能別テストケース**（6-B-1〜7、7並列）
  - auth.md(80件), reports.md(90件), items.md(75件), attachments.md(54件), workflow.md(62件), dashboard.md(25件), tenant.md(11件) = 計397件
  - 内部レビュー: blocker 2件（WFL-013 不完全、attachments rejected 欠落）+ warning 5件 → 修正 → 再レビュー PASS
  - codex レビュー: 052 差戻し継続（cross-cutting.md 未作成のため想定通り）
- **Phase 3: 横断テストケース**（6-B-8）
  - cross-cutting.md(84件): テナント分離(16件), RBAC(34件), E2E(21件), 非機能(13件)
  - 内部レビュー: blocker 2件（テナントBフィクスチャ不足、RBAC Approver添付閲覧不正確）+ warning 2件 → 修正 → 再レビュー PASS
  - codex 最終レビュー: 052 resolved、新規指摘なし
- **Step 6 完了宣言**
  - progress.md を完了（2026-03-23）に更新

### 未完了
- なし（Step 6 完了）

### ブロッカー
- なし

### 次にやること
1. Step 7（実装・運用）に着手
   - まず `dev-journal/guide/work-breakdown/step7-implementation.md` を確認
   - open issues を確認（005, 008, ops-032）
2. Step 7 の作業計画を立案（Plan エージェント）
   - Phase 1: 基盤構築（7-B）から開始

### 学び・気づき
- 成果物構成の前提を疑うユーザーの視点が、TDD に基づく正しい粒度（ハンドラ単位）の発見につながった。「そのファイル数で本当にいいのか」は常に問うべき
- 複数エージェントに同じ問いを投げて意見を突き合わせる手法が有効。全員一致の点は信頼でき、意見が割れた点に判断を集中できる
- codex は Phase 段階のコミットでも「Step 完了条件未達」として差し戻す。Phase 分割で作業している場合、052 のような「想定通りの差戻し」は open のまま進めて最終 Phase で解消する運用が正しい
- 内部レビューの「再レビュー省略」を提案したらユーザーに指摘された。ワークフローの「PASS まで繰り返す」は省略不可。手間の考慮は判断基準にならない

### 意思決定ログ
- 成果物粒度: test_cases を Goハンドラファイル単位（8ファイル）に分割。理由: TDD ではテストコードの構造（*_handler_test.go）と設計書が1:1対応すべき
- dashboard/categories/tenant: dashboard+categories で1ファイル、tenant で1ファイル。理由: 認可境界（全ロール参照 vs Admin専用）で分ける
- DSH-018（ダッシュボードテナント分離テスト）: dashboard.md に残す。cross-cutting.md の CRS-016 が正本、DSH-018 は集計値正確性テスト。理由: 「テナント越境=404」と「集計値汚染チェック」は性質が異なる
- 非機能テスト方針: MVP対象外ではなく、上流に具体値があるため方針を定義。レート制限は統合テスト（PR時）、レスポンスタイムは軽量スモーク（mainマージ後）
- CRS-015 vs TNT-008: 両方実装する。CRS-015=テナント越境テスト正本、TNT-008=フィルタリング正確性テスト

---

## セッション: 2026-03-23 15:25（前回）

### ゴール
- Step 5 完了を目指す（review-finding 048 対応 → Phase 4 最終レビュー → 完了宣言）

### 作業ログ
- **review-finding 048 対応**（ui_flow.md 全体図の遷移欠落）
  - ui_flow.md の全体画面遷移図に `DASH001 -> ADM001`, `DASH001 -> ADM002` エッジ追加
  - review-finding 048 を resolved に移動
- **Phase 4 最終レビュー実施**（unit x4 + cross x1 = 5エージェント並列）
  - Auth 4画面: LGTM
  - Dashboard + Workflow 3画面: LGTM（info 1件 — Phase 3 対応で十分）
  - Admin 2画面: warning 2件（ソートキー不一致、期間フィルタ曖昧）
  - Report 4画面: blocker 1件（ソートキー updated_at vs created_at）
  - 横断レビュー: LGTM（warning 2件 — エラーコード略記、submitter 用語 → PASS）
- **Phase 4 指摘修正**
  - RPT-001: シーケンス図 `ORDER BY updated_at` → `created_at` に修正
  - ADM-001: ソートキー `updated_at` → `submitted_at DESC NULLS LAST` に修正 + 期間フィルタのセマンティクス明確化
  - 再レビュー: LGTM
- **codex レビュー → review-finding 051 対応**（PERMISSION_DENIED → FORBIDDEN 統一）
  - 方針 C 採用: MVP では FORBIDDEN に統一
  - 9ファイル修正 → codex レビュー LGTM
- **Step 5 完了宣言**

### 未完了
- なし（Step 5 完了）

### ブロッカー
- なし

### 次にやること
1. Step 6（テスト設計）に着手

### 学び・気づき
- review-finding 048 のレビューを内部レビューに回してしまったが、codex からの指摘は codex に再レビューを委譲すべき
- PERMISSION_DENIED の導入は上流に定義がない概念の独断導入だった

### 意思決定ログ
- PERMISSION_DENIED 廃止: 方針 C。MVP の外部契約は上流 RBC-004 準拠で FORBIDDEN に統一
- ソートキー統一: RPT-001 は created_at、ADM-001 は submitted_at DESC NULLS LAST

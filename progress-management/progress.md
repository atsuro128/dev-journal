# プロジェクト進捗管理

## マイルストーン

| # | マイルストーン | 状態 | 完了日 | ガイド |
|---|--------------|------|--------|--------|
| 0 | 事前準備（プロジェクトの土台づくり） | 完了 | 2026-03-04 | `ai-dev-framework/guide/work-breakdown/step0-preparation/` |
| 1 | 要件定義（業務理解 → ユースケース化） | 完了 | 2026-03-09 | `ai-dev-framework/guide/work-breakdown/step1-requirements/` |
| 2 | ドメイン設計（データとルールの核） | 完了 | 2026-03-14 | `ai-dev-framework/guide/work-breakdown/step2-domain/` |
| 3 | アーキテクチャ設計（技術選定・構成決定） | 完了 | 2026-03-16 | `ai-dev-framework/guide/work-breakdown/step3-architecture/` |
| 4 | 基本設計（画面一覧・画面遷移） | 完了 | 2026-03-22 | `ai-dev-framework/guide/work-breakdown/step4-basic-design/` |
| 5 | 詳細設計（API・DB・認可・セキュリティ） | 完了 | 2026-03-23 | `ai-dev-framework/guide/work-breakdown/step5-detail-design/` |
| 5.5 | UI コンポーネント設計 | 完了 | 2026-04-06 | `ai-dev-framework/guide/work-breakdown/step5.5-ui-component/` |
| 6 | テスト設計 | 完了 | 2026-04-12 | `ai-dev-framework/guide/work-breakdown/step6-testing/` |
| 7 | 運用設計 | 完了 | 2026-03-26 | `ai-dev-framework/guide/work-breakdown/step7-operations/` |
| 8 | 基盤構築 | 完了 | 2026-03-31（初回完了）/ 2026-04-12（8-11 追加・再完了） | `ai-dev-framework/guide/work-breakdown/step8-foundation/` |
| 9 | テストコード実装 | 完了 | 2026-04-08 | `ai-dev-framework/guide/work-breakdown/step9-test-implementation/` |
| 10 | 機能実装 | 完了 | 2026-04-11 | `ai-dev-framework/guide/work-breakdown/step10-feature-implementation/` |
| 11 | システムテスト・UAT | 進行中 | - | `ai-dev-framework/guide/work-breakdown/step11-system-test/` |

## タスク状態定義

| 状態 | 意味 | 遷移条件 |
|------|------|----------|
| 未着手 | 未着手 | — |
| 作業中 | サブエージェントが作業中 | 依存先が全て `完了` |
| レビュー待ち | 成果物完成、レビュー未実施 | サブエージェント作業完了 |
| 修正中 | レビュー指摘を対応中 | レビューで FIX 判定 |
| 完了 | レビュー LGTM 済み | 品質ゲート PASS |

## 完了 Step のチケット一覧

Step 5.5 / 6（追加）/ 8 / 9 / 10 のチケット一覧はアーカイブ済み（`archives/progress/steps.md` 参照）。

## Step 11: システムテスト・UAT — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 11-A | ローカル動作確認 | ユーザー + 指揮役 | Step 10 全完了、Step 8-11 完了 | 進行中（5 PR マージ済み: #121/#122/#123/#124/#125。SMK 再検証進捗: #157/#162 PASS、#158 SMK-103/104/105 PASS、残: SMK-101 再検証 + #164 DevTools 視覚再確認 → docker compose リビルド後にユーザー実施） | `tickets/step11/11-A-local-verification.md` |
| 11-B | 横断テスト（Go） | test-implementer | 11-A | 未着手 | `tickets/step11/11-B-cross-cutting-test.md` |
| 11-C | E2E テスト（Playwright） | test-implementer | 11-A | 未着手 | `tickets/step11/11-C-e2e-test.md` |
| 11-D | 横断レビュー | reviewer (codex) | 11-B, 11-C | 未着手 | `tickets/step11/11-D-cross-review.md` |
| 11-E | デプロイ・スモークテスト | platform-builder | 11-D | 未着手 | `tickets/step11/11-E-deploy.md` |
| 11-F | UAT | ユーザー | 11-E | 未着手 | `tickets/step11/11-F-uat.md` |

## 課題・ブロッカー

### 解決済み（2026-04-12 〜 2026-04-16）

アーカイブ済み（`archives/progress/issues.md` 参照）。

### 残存 issue（Step 11-A 関連）
| ID | タイトル | 起票日 | 状態 |
|----|---------|-------|------|
| 133 | ログ・エラー出力の言語ポリシー整理（FE/BE 共通、日本語メッセージの棚卸しと削減） | 2026-04-21 | 別セッションで対応予定 |
| 145 | ReportWorkflowInfo セクション見出し追加（post-MVP） | 2026-04-24 | 起票のみ（post-MVP） |
| 146 | 大規模クロステナント環境での性能テスト計画（post-MVP） | 2026-04-25 | 起票のみ（post-MVP） |
| 151 | パスワードリセットのメール送信が未実装（要件 AUTH-F06 乖離、post-MVP） | 2026-04-28 | 起票のみ（post-MVP） |
| 160 | テナント全レポート一覧スマホ幅で横スクロール / 列省略改善（再対応） | 2026-05-01 | PR #122 + PR #124 マージ済み（案 F + F': AppDataGrid Box + AppLayout main の minWidth:0 で外側 main の膨張を抑制、内側 overflow auto を発火）、SMK-101 再検証待ち |
| 162 | 却下ダイアログ初期表示で error / disabled 発火 regression（再 open） | 2026-05-02 | PR #121 マージ済み（autoFocus 削除）、SMK-011 #4 / SMK-096 #2 PASS 確認済み、resolved 移動候補 |
| 164 | Admin ダッシュボード PC 幅でカード幅が他ロールより短い（再対応） | 2026-04-30 | PR #125 マージ済み（TenantStatusCards Grid を md:4 で 3+2 の 2 行レイアウトに変更、他ロール 1/3 幅と統一）、DevTools 視覚再確認待ち |
| 165 | マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える | 2026-05-02 | open（#160 真因調査中に発見、Stack direction レスポンシブ化で対応予定） |
| 166 | Step 11-A SMK-104/105 検証用フィクスチャ不足（テナント B Approver / テナント A 第二 Approver） | 2026-05-02 | PR #123 マージ済み（seed.go フィクスチャ追加 + 補完 UPDATE + regression テスト）、SMK-104/105 PASS 確認済み、resolved 移動候補（BE: full test 実行待ち） |

### pending-review issue（第3バッチ + #157/#158 マージ済み、SMK 再検証待ち）

第1バッチ 6 件（#152/#153/#155/#156/#159/#161）は SMK 再検証 PASS で `resolved/` 移動済み（commit `f7fbf16`）。

第2バッチ #157 はマージ済み・SMK 検証は全画面共通の通常確認に統合（個別 SMK なし）。

第3バッチ MVP 対応（マージ済み、SMK 再検証 PASS、resolved 移動候補）:

| ID | タイトル | PR | コミット | 状態 |
|----|---------|-----|---------|------|
| 158 | Approver 処理済みレポート一覧（SCR-WFL-003 新規） | #111 | 722568e ほか | SMK-103/104/105 PASS 確認済み、resolved 移動候補 |

※ #162 / #164 は再対応で再 open 化（PR #121 / PR #125 マージ済み）、PR マージ後の状態は上記「残存 issue (Step 11-A 関連)」表に統合。

第2バッチ（マージ済み、DevTools 視覚 PASS、resolved 移動候補）:

| ID | タイトル | PR | コミット | 状態 |
|----|---------|-----|---------|------|
| 157 | ダッシュボードの月別合計テーブルと直近レポート一覧の境界（セクション見出し追加） | #110 | 6e0687a (dev-journal) | DevTools 視覚 PASS、resolved 移動候補 |

### 残存 issue（運用・基盤系）
| ID | タイトル |
|----|---------|
| ops-055 | work-breakdown テンプレート不整合 |
| 060 | devcontainer egress allowlist の厳格化と根拠の欠如 |
| 061 | devcontainer マウントとシークレット露出の最小化 |
| ops-062 | ワークフロースキルの粒度 |
| 064 | MCP ジョブログが proxy allowlist でブロック |
| ops-080 | Post-MVP スコープ管理方法 |
| 081 | Post-MVP テストカバレッジ項目 |
| 084 | Post-MVP HttpOnly Cookie 移行 |
| 104 | UI 層の表示・振る舞いカバレッジ監査（ロール別表示制御 + レスポンシブ） |
| 122 | Post-MVP セッション期限切れ時のリダイレクト UX 改善（サイレント → インラインアラート + return_to） |

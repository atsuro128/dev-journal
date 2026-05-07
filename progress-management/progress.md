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
| 11-A | ローカル動作確認 | ユーザー + 指揮役 | Step 10 全完了、Step 8-11 完了 | 完了（2026-05-05: 全 62 SMK PASS（FAIL 0）、副次 #170 起票） | `tickets/step11/11-A-local-verification.md` |
| 11-B | 横断テスト（Go） | test-implementer | 11-A | 完了（2026-05-06: PR #139 マージ済み、cross_cutting_test.go + rate_limit_test.go 1363 行追加、副次 #172 起票・解消） | `tickets/step11/11-B-cross-cutting-test.md` |
| 11-C | E2E テスト（Playwright） | test-implementer | 11-A | 完了（2026-05-07: PR #138 マージ済み、E2E 10/10 PASS。CRS-066 真因 = Docker クロックドリフト 18 秒巻戻りで JWT iat 未来扱い 401 → JWT leeway 60s 追加 PR #141 で解消、副次 #173 起票・解消） | `tickets/step11/11-C-e2e-test.md` |
| 11-D | 横断レビュー | reviewer (codex) | 11-B, 11-C | 完了（2026-05-07: codex 11-D 横断レビュー CONDITIONAL PASS。初回 FAIL のブロッカー B-01 (issue #173 対応漏れ middleware 経路) + M-03 (rate limit テストケース乖離) + M-04 (npm run e2e config 未指定) を全て解消し再レビュー PASS。CONDITIONAL の 4 条件は 11-E 進入条件として step11-d-review-report.md に明記） | `tickets/step11/11-D-cross-review.md` |
| 11-E | デプロイ・スモークテスト | platform-builder | 11-D | 未着手 | `tickets/step11/11-E-deploy.md` |
| 11-F | UAT | ユーザー | 11-E | 未着手 | `tickets/step11/11-F-uat.md` |

## 課題・ブロッカー

### 解決済み（2026-04-12 〜 2026-04-16）

アーカイブ済み（`archives/progress/issues.md` 参照）。

### 残存 issue（Step 11 関連）
| ID | タイトル | 起票日 | 状態 |
|----|---------|-------|------|
| 133 | ログ・エラー出力の言語ポリシー整理（FE/BE 共通、日本語メッセージの棚卸しと削減） | 2026-04-21 | 別セッションで対応予定 |
| 145 | ReportWorkflowInfo セクション見出し追加（post-MVP） | 2026-04-24 | 起票のみ（post-MVP） |
| 146 | 大規模クロステナント環境での性能テスト計画（post-MVP） | 2026-04-25 | 起票のみ（post-MVP） |
| 151 | パスワードリセットのメール送信が未実装（要件 AUTH-F06 乖離、post-MVP） | 2026-04-28 | 起票のみ（post-MVP） |
| 165 | マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える | 2026-05-02 | resolved（PR #135 + #136 + #137 マージ済み 2026-05-06、レイアウト統一 + 改行位置調整 + デグレ修正） |
| 167 | DataGrid 列の自動幅 minWidth と手動リサイズ minWidth の分離 | 2026-05-03 | 起票のみ（post-MVP） |
| 170 | 「保存して続けて追加」ボタン押下で明細が 2 件登録される（データ整合性 blocker） | 2026-05-05 | resolved（PR #132 マージ済み 2026-05-05） |
| 171 | 認証フォームのメールアドレス欄に autocomplete 属性が欠落しており、ブラウザ・パスワードマネージャーがサジェスト候補を出さない | 2026-05-05 | resolved（PR #133 + PR #134 マージ済み 2026-05-05、Edge での手動確認 PASS） |
| 172 | ログインエンドポイントのレート制限値を環境変数で上書き可能にする（11-C E2E テストブロッカー） | 2026-05-06 | resolved（PR #140 マージ済み 2026-05-06、本番値 5/20 維持・dev/E2E 100/200 緩和） |
| 173 | JWT 検証にクロックスキュー許容（leeway）を追加（CRS-066 真因 = Docker クロックドリフト） | 2026-05-07 | resolved（PR #141 マージ済み 2026-05-07 で domain.JWTVerifier に leeway 適用、PR #143 マージ済み 2026-05-07 で pkg/jwt.Verifier (middleware 経路) にも leeway 適用、両系統を 11-D 再レビューで確認） |
| 174 | Step 11-C E2E テスト品質改善（CRS-074 アサーション強化 + カテゴリセレクタ data-testid 化） | 2026-05-07 | 起票のみ（post-MVP） |

### pending-review issue

全件 resolved 移動済み（archives/progress/issues.md 参照）

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

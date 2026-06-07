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
| 11 | システムテスト・UAT | 完了 | 2026-05-28 | `ai-dev-framework/guide/work-breakdown/step11-system-test/` |

## タスク状態定義

| 状態 | 意味 | 遷移条件 |
|------|------|----------|
| 未着手 | 未着手 | — |
| 作業中 | サブエージェントが作業中 | 依存先が全て `完了` |
| レビュー待ち | 成果物完成、レビュー未実施 | サブエージェント作業完了 |
| 修正中 | レビュー指摘を対応中 | レビューで FIX 判定 |
| 完了 | レビュー LGTM 済み | 品質ゲート PASS |

## 完了 Step のチケット一覧

Step 5.5 / 6（追加）/ 8 / 9 / 10 / 11 のチケット一覧はアーカイブ済み（`archives/progress/steps.md` 参照）。

## 課題・ブロッカー

### 解決済み（2026-04-12 〜 2026-04-16）

アーカイブ済み（`archives/progress/issues.md` 参照）。

### 残存 issue（post-MVP / open）

resolved になった Step 11 関連 issue（#165, #170〜173, #175, #181, #183〜188, #190）はアーカイブ済み（`archives/progress/issues.md` 参照）。

| ID | タイトル | 起票日 | 状態 |
|----|---------|-------|------|
| 133 | ログ・エラー出力の言語ポリシー整理（FE/BE 共通、日本語メッセージの棚卸しと削減） | 2026-04-21 | 別セッションで対応予定 |
| 145 | ReportWorkflowInfo セクション見出し追加（post-MVP） | 2026-04-24 | 起票のみ（post-MVP） |
| 146 | 大規模クロステナント環境での性能テスト計画（post-MVP） | 2026-04-25 | 起票のみ（post-MVP） |
| 151 | パスワードリセットのメール送信が未実装（要件 AUTH-F06 乖離、post-MVP） | 2026-04-28 | 起票のみ（post-MVP） |
| 167 | DataGrid 列の自動幅 minWidth と手動リサイズ minWidth の分離 | 2026-05-03 | 起票のみ（post-MVP。issue #195 で UAT-044 のスマホ幅実機問題として再確認、対応時は統合検討） |
| 174 | Step 11-C E2E テスト品質改善（CRS-074 アサーション強化 + カテゴリセレクタ data-testid 化） | 2026-05-07 | 起票のみ（post-MVP） |
| 176 | frontend dev 依存の既知 CVE 棚卸し（esbuild + postcss、post-MVP） | 2026-05-13 | 起票のみ（post-MVP） |
| 177 | frontend バンドルの chunk size 500 kB 超過警告（code splitting 未適用、post-MVP） | 2026-05-13 | 起票のみ（post-MVP） |
| 178 | Go バージョン定義の環境間不整合（devcontainer / production build / test-be、post-MVP） | 2026-05-13 | 起票のみ（post-MVP） |
| 179 | devcontainer egress allowlist の github.com 全許可によるツール install リスク（issue #060 派生、post-MVP） | 2026-05-13 | 起票のみ（post-MVP） |
| 180 | backend.tf の dynamodb_table が Terraform 1.10+ で deprecated（use_lockfile に移行） | 2026-05-18 | 起票のみ（post-MVP） |
| 182 | devcontainer egress allowlist に registry.terraform.io が含まれず terraform validate 実行不可（issue #179 系統） | 2026-05-19 | 起票のみ（post-MVP） |
| 189 | terraform/security_groups.tf: SG ingress description に日本語混入で AWS validation 違反（PR #151 漏れ） | 2026-05-26 | 起票のみ（post-MVP。緊急 fix → PR #154 マージ済み。再発防止 lint ルールは post-MVP 検討） |
| 191 | 全レポート画面: 対象期間フィルタはあるがテーブル列に対象期間カラムが無く UX 上の違和感（post-MVP） | 2026-05-27 | 起票のみ（post-MVP。UAT-018 で発覚、業務継続可能） |
| 192 | draft レポートの Admin / Accounting 閲覧可否の見直し検討（post-MVP） | 2026-05-27 | 起票のみ（post-MVP。UAT-020 で発覚、申請者の心理的安全性 + Approver との一貫性の観点で見直し検討） |
| 193 | 一覧↔詳細往復で 100 req/min 到達 + F5 で生 JSON + 回復遅さ（post-MVP） | 2026-05-27 | 起票のみ（post-MVP。UAT-032 自由探索で発覚） |
| 194 | 明細金額フィールドの UX 改善（全角 IME 残置 + エラー文言 + 上限要相談、post-MVP） | 2026-05-28 | 起票のみ（post-MVP。UAT-042 で発覚） |
| 195 | スマホ幅でレポート詳細の明細一覧テーブルの列幅が縦書き化（post-MVP） | 2026-05-28 | 起票のみ（post-MVP。UAT-044 で発覚、issue #167 と統合検討） |
| 196 | マイレポート画面で 0 件時のテーブル空状態表示が見切れる（post-MVP） | 2026-05-28 | 起票のみ（post-MVP。UAT-044 で発覚） |
| 197 | AWS公開デモの ALB 除去による lean 化（コスト最適化 + 深夜自動 stop/start 運用） | 2026-06-04 | **実装・本番 apply 完了（2026-06-05・lean 構成で 200 稼働中）**。EC2 replace でアプリ消失→SSM 再デプロイで復旧。深夜 stop/start は JST 00:00 停止/08:30 起動。詳細は session-log |

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

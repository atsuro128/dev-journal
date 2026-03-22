# Step 5 詳細設計 — 作業計画

## 判断ポイント（確定済み）

| # | 論点 | 決定 |
|---|------|------|
| 1 | 月別サマリー期間 | 3ヶ月固定 |
| 2 | 直近レポート一覧件数 | 5件 |
| 3 | 明細の編集方式 | スライドパネル |
| 5 | S3 キー命名規則 | `{tenant_id}/{report_id}/{attachment_id}` |
| 6 | アップロード方式 | API 経由プロキシ（MVP） |
| 追加 | カテゴリの DB 実装 | マスタテーブル + FK（Phase 3 カスタムカテゴリに備える） |

## Phase 1（9タスク並列）

画面詳細仕様（basic-designer x5）:

| タスク | 成果物 | 入力（主要） |
|--------|--------|-------------|
| T1-1 | `screens/auth.md` — 認証系4画面 | screens.md, usecases.md (UC-SYS01/02/05), requirements.md (AUTH-*, SEC-*) |
| T1-2 | `screens/dashboard.md` — ダッシュボード | screens.md, usecases.md (UC-SYS04), requirements.md (DASH-*) |
| T1-3 | `screens/report.md` — 経費レポート系4画面（最大・最重要） | screens.md, usecases.md (UC-M01〜09), workflow.md, state_machine.md, domain_model.md |
| T1-4 | `screens/workflow.md` — ワークフロー系2画面 | screens.md, usecases.md (UC-A01〜03, UC-AC01〜02), workflow.md |
| T1-5 | `screens/admin.md` — 管理系2画面 | screens.md, usecases.md (UC-AD01/02, UC-AC03) |

横断設計（db-designer x1, detail-designer x3）:

| タスク | 成果物 | 入力（主要） |
|--------|--------|-------------|
| T1-6 | `db_schema.md` — DB スキーマ設計 | domain_model.md, state_machine.md, ADR-0002/0003 |
| T1-7 | `files.md` — 添付ファイル設計 | requirements.md (ATT-*), domain_model.md, security-policy.md |
| T1-8 | `security.md` — セキュリティ設計 | requirements.md (SEC-*), architecture.md, ADR-0001 |
| T1-9 | `monitoring.md` — 監視・ログ設計 | ADR-0005, architecture.md |

成果物配置先: `dev-journal/deliverables/docs/`（画面詳細は `40_basic_design/screens/`、横断設計は `50_detail_design/`）

## Phase 2（Phase 1 完了後）

| タスク | 成果物 | 入力 |
|--------|--------|------|
| T2-1 | `openapi.yaml` — OpenAPI 定義 | Phase 1 全成果物 + architecture.md, screens.md, rbac.md, domain_model.md |

## Phase 3（Phase 2 完了後）

| タスク | 成果物 | 入力 |
|--------|--------|------|
| T3-1 | `authz.md` — 認可設計 + `ui_flow.md` 最終版 | 全成果物 + rbac.md |

## Phase 4（レビュー）

| タスク | エージェント | 対象 |
|--------|------------|------|
| T4-1 | design-unit-reviewer x5（並列） | 機能単位（認証/ダッシュボード/経費レポート/ワークフロー/管理） |
| T4-2 | design-cross-reviewer x1 | 全成果物横断 |

## 依存グラフ

```
Phase 1（並列）          Phase 2      Phase 3       Phase 4
T1-1〜5: screens/*.md ─┐
T1-6: db_schema.md ────┼→ T2-1: ──→ T3-1: ──→ T4-1: unit x5
T1-7: files.md ────────┤  openapi    authz +      ↓
T1-8: security.md ─────┤  .yaml      ui_flow   T4-2: cross x1
T1-9: monitoring.md ───┘
```

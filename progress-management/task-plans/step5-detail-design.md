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

## Phase 構成

### Phase 1（17タスク並列）

画面詳細仕様（basic-designer x13）:

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| T1-01 | `screens/auth-signup.md` — サインアップ (SCR-AUTH-001) | screens.md, usecases.md (UC-SYS01), requirements.md (AUTH-*, SEC-*) | basic-designer | 完了 |
| T1-02 | `screens/auth-login.md` — ログイン (SCR-AUTH-002) | screens.md, usecases.md (UC-SYS02), requirements.md (AUTH-*, SEC-*) | basic-designer | 完了 |
| T1-03 | `screens/auth-password-reset-request.md` — パスワードリセット要求 (SCR-AUTH-003) | screens.md, usecases.md (UC-SYS05), requirements.md (SEC-*) | basic-designer | 完了 |
| T1-04 | `screens/auth-password-reset.md` — パスワードリセット実行 (SCR-AUTH-004) | screens.md, usecases.md (UC-SYS05), requirements.md (SEC-*) | basic-designer | 完了 |
| T1-05 | `screens/dashboard.md` — ダッシュボード (SCR-DASH-001) | screens.md, usecases.md (UC-SYS04), requirements.md (DASH-*) | basic-designer | 完了 |
| T1-06 | `screens/report-list.md` — レポート一覧 (SCR-RPT-001) | screens.md, usecases.md (UC-M01), requirements.md (RPT-*) | basic-designer | 完了 |
| T1-07 | `screens/report-create.md` — レポート作成 (SCR-RPT-002) | screens.md, usecases.md (UC-M02, UC-M09), workflow.md, state_machine.md | basic-designer | 完了 |
| T1-08 | `screens/report-edit.md` — レポート編集 (SCR-RPT-003) | screens.md, usecases.md (UC-M03), domain_model.md | basic-designer | 完了 |
| T1-09 | `screens/report-detail.md` — レポート詳細 (SCR-RPT-004) | screens.md, usecases.md (UC-M01〜09), workflow.md, state_machine.md, domain_model.md | basic-designer | 完了 |
| T1-10 | `screens/workflow-pending.md` — 承認待ち一覧 (SCR-WFL-001) | screens.md, usecases.md (UC-A01〜03), workflow.md | basic-designer | 完了 |
| T1-11 | `screens/workflow-payable.md` — 支払待ち一覧 (SCR-WFL-002) | screens.md, usecases.md (UC-AC01〜02), workflow.md | basic-designer | 完了 |
| T1-12 | `screens/admin-all-reports.md` — テナント全レポート一覧 (SCR-ADM-001) | screens.md, usecases.md (UC-AD01, UC-AC03) | basic-designer | 完了 |
| T1-13 | `screens/admin-tenant.md` — テナント情報 (SCR-ADM-002) | screens.md, usecases.md (UC-AD02) | basic-designer | 完了 |

横断設計（db-designer x1, detail-designer x3）:

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| T1-14 | `db_schema.md` — DB スキーマ設計 | domain_model.md, state_machine.md, ADR-0002/0003 | db-designer | 完了 |
| T1-15 | `files.md` — 添付ファイル設計 | requirements.md (ATT-*), domain_model.md, security-policy.md | detail-designer | 完了 |
| T1-16 | `security.md` — セキュリティ設計 | requirements.md (SEC-*), architecture.md, ADR-0001 | detail-designer | 完了 |
| T1-17 | `monitoring.md` — 監視・ログ設計 | ADR-0005, architecture.md | detail-designer | 完了 |

レビュー:

| タスク | 内容 | エージェント | 状態 |
|--------|------|------------|------|
| T1-R1 | 内部レビュー | design-cross-reviewer | 完了 |
| T1-R2 | codex レビュー | codex | 完了 |

成果物配置先: `dev-journal/deliverables/docs/`（画面詳細は `40_basic_design/screens/`、横断設計は `50_detail_design/`）

### Phase 2（Phase 1 完了後）

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| T2-1 | `openapi.yaml` — OpenAPI 定義 | Phase 1 全成果物 + architecture.md, screens.md, rbac.md, domain_model.md | api-designer | 完了 |
| T2-R1 | 内部レビュー | — | design-cross-reviewer | 完了 |
| T2-R2 | codex レビュー | — | codex | 完了 |

成果物配置先: `dev-journal/deliverables/docs/50_detail_design/`

### Phase 3（Phase 2 完了後）

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| T3-1 | `authz.md` — 認可設計 + `ui_flow.md` 最終版 | 全成果物 + rbac.md | detail-designer | 未着手 |
| T3-R1 | 内部レビュー | — | design-cross-reviewer | 未着手 |
| T3-R2 | codex レビュー | — | codex | 未着手 |

成果物配置先: `dev-journal/deliverables/docs/50_detail_design/`

### 各 Phase の進め方

**実行前:**
1. 並列タスクがある場合、共通ルール・判断ポイントの適用箇所・タスクごとの要点を整理する
2. 整理した内容をサブエージェントのプロンプトに含めて委譲する

**実行後:**
1. **内部レビュー**（design-cross-reviewer）: Phase 成果物の整合性チェック
   - blocker があれば修正 → 再レビュー（LGTM まで繰り返す）
2. **ユーザーにコミットを提案**
3. **codex レビュー**（/codex-review）: コミット後に実行
   - 指摘があれば対応 → 再レビュー（LGTM まで繰り返す）
4. 次の Phase に進む

### Phase 4（最終レビュー）

全成果物が揃った後に実施:

| タスク | エージェント | 対象 | 状態 |
|--------|------------|------|------|
| T4-1 | design-unit-reviewer x13（並列） | 画面単位（auth-signup/login/password-reset-request/password-reset, dashboard, report-list/create/edit/detail, workflow-pending/payable, admin-all-reports/tenant） | 未着手 |
| T4-2 | design-cross-reviewer x1 | 全成果物横断 | 未着手 |
| T4-R1 | codex レビュー | 全成果物 | 未着手 |

## 依存グラフ

```
Phase 1（並列）               レビュー   Phase 2    レビュー   Phase 3     Phase 4
T1-01〜13: screens/*.md ─┐    ──→  T2-1:    ──→  T3-1:    ──→ T4-1: unit x5
T1-14: db_schema.md ─────┤  コミット openapi  コミット authz +      ↓
T1-15: files.md ──────────┤           .yaml            ui_flow   T4-2: cross x1
T1-16: security.md ───────┤                                      コミット
T1-17: monitoring.md ─────┘
```

## 品質基準

- **上流整合性**: 入力として参照した上流成果物と矛盾がないか
- **タスク間整合性**: 並列タスクの成果物間で用語・ID・参照関係が統一されているか
- **MVP スコープ**: `deliverables/docs/02_scope.md` の範囲内か
- **用語集準拠**: `dev-journal/references/glossary.md` の用語を使用しているか
- **完了条件**: work-breakdown に定義された完了条件を満たしているか

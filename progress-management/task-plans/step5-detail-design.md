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

### Phase 0（実行準備）

| タスク | 成果物 | 入力（主要） | エージェント |
|--------|--------|-------------|------------|
| T0-1 | Phase 1 実行指示書（本ファイルの「実行指示書」セクション） | work-breakdown, 上流成果物, 判断ポイント | Plan |

### Phase 1（9タスク並列）

画面詳細仕様（basic-designer x5）:

| タスク | 成果物 | 入力（主要） | エージェント |
|--------|--------|-------------|------------|
| T1-1 | `screens/auth.md` — 認証系4画面 | screens.md, usecases.md (UC-SYS01/02/05), requirements.md (AUTH-*, SEC-*) | basic-designer |
| T1-2 | `screens/dashboard.md` — ダッシュボード | screens.md, usecases.md (UC-SYS04), requirements.md (DASH-*) | basic-designer |
| T1-3 | `screens/report.md` — 経費レポート系4画面（最大・最重要） | screens.md, usecases.md (UC-M01〜09), workflow.md, state_machine.md, domain_model.md | basic-designer |
| T1-4 | `screens/workflow.md` — ワークフロー系2画面 | screens.md, usecases.md (UC-A01〜03, UC-AC01〜02), workflow.md | basic-designer |
| T1-5 | `screens/admin.md` — 管理系2画面 | screens.md, usecases.md (UC-AD01/02, UC-AC03) | basic-designer |

横断設計（db-designer x1, detail-designer x3）:

| タスク | 成果物 | 入力（主要） | エージェント |
|--------|--------|-------------|------------|
| T1-6 | `db_schema.md` — DB スキーマ設計 | domain_model.md, state_machine.md, ADR-0002/0003 | db-designer |
| T1-7 | `files.md` — 添付ファイル設計 | requirements.md (ATT-*), domain_model.md, security-policy.md | detail-designer |
| T1-8 | `security.md` — セキュリティ設計 | requirements.md (SEC-*), architecture.md, ADR-0001 | detail-designer |
| T1-9 | `monitoring.md` — 監視・ログ設計 | ADR-0005, architecture.md | detail-designer |

成果物配置先: `dev-journal/deliverables/docs/`（画面詳細は `40_basic_design/screens/`、横断設計は `50_detail_design/`）

### Phase 2（Phase 1 完了後）

| タスク | 成果物 | 入力（主要） | エージェント |
|--------|--------|-------------|------------|
| T2-1 | `openapi.yaml` — OpenAPI 定義 | Phase 1 全成果物 + architecture.md, screens.md, rbac.md, domain_model.md | api-designer |

成果物配置先: `dev-journal/deliverables/docs/50_detail_design/`

### Phase 3（Phase 2 完了後）

| タスク | 成果物 | 入力（主要） | エージェント |
|--------|--------|-------------|------------|
| T3-1 | `authz.md` — 認可設計 + `ui_flow.md` 最終版 | 全成果物 + rbac.md | detail-designer |

成果物配置先: `dev-journal/deliverables/docs/50_detail_design/`

### Phase 4（レビュー）

| タスク | エージェント | 対象 |
|--------|------------|------|
| T4-1 | design-unit-reviewer x5（並列） | 機能単位（認証/ダッシュボード/経費レポート/ワークフロー/管理） |
| T4-2 | design-cross-reviewer x1 | 全成果物横断 |

## 依存グラフ

```
Phase 0        Phase 1（並列）          Phase 2      Phase 3       Phase 4
T0-1: ──→ T1-1〜5: screens/*.md ─┐
実行指示書    T1-6: db_schema.md ────┼→ T2-1: ──→ T3-1: ──→ T4-1: unit x5
              T1-7: files.md ────────┤  openapi    authz +      ↓
              T1-8: security.md ─────┤  .yaml      ui_flow   T4-2: cross x1
              T1-9: monitoring.md ───┘
```

## 実行指示書

### Phase 1 実行指示書

#### 共通ルール

- **用語**: 用語集（`dev-journal/references/glossary.md`）に従う。主要な用語: 申請（submit）、却下（reject）、再申請（resubmit）、支払済みにする（mark as paid）。テーブル名・カラム名も用語集準拠

- **判断ポイントの適用**: 確定済み判断ポイント6件を該当タスクで必ず反映する
  - 月別サマリー3ヶ月固定 → T1-2
  - 直近レポート5件 → T1-2
  - 明細編集スライドパネル → T1-3
  - S3キー `{tenant_id}/{report_id}/{attachment_id}` → T1-7
  - アップロードAPI経由プロキシ → T1-7
  - カテゴリマスタテーブル+FK → T1-6

- **テナント分離**: 全テーブル `tenant_id` 必須（`users` は `tenant_memberships` 経由）。`expense_items` と `attachments` は RLS 効率のため `tenant_id` 冗長保持。テナント境界越えは 404（403 ではない）

- **ロール**: Member, Approver, Admin, Accounting の4ロール。Approver/Accounting は Member 操作も可能。Admin は自テナントのレポート閲覧のみ（編集・承認・却下不可）。自己承認禁止、自己支払処理禁止

- **状態遷移**: 5状態（draft, submitted, approved, rejected, paid）。draft のみ編集可。終端状態: rejected, paid。`state_machine.md` 準拠

- **ID体系**: 画面ID `SCR-{category}-{seq}`、要件ID `AUTH-F01` 等、ビジネスルールID `SEC-001` 等、ユースケースID `UC-M01` 等

- **関連 issue**: issue 036（S3キー命名規則）— `security-policy.md` §3, §5 の `file_id` を `attachment_id` に読み替え。T1-7 で統一を適用

- **成果物配置先**: 画面詳細は `dev-journal/deliverables/docs/40_basic_design/screens/`、横断設計は `dev-journal/deliverables/docs/50_detail_design/`

#### タスクごとの要点

##### T1-1: screens/auth.md（basic-designer）

- **読むべき入力と注目セクション**:
  - `40_basic_design/screens.md` §3.1（認証系4画面）、§4.2（ヘッダー）、§4.4（エラー表示）
  - `10_requirements/usecases.md` UC-SYS01（サインアップ）、UC-SYS02（ログイン）、UC-SYS05（パスワードリセット）
  - `10_requirements/requirements.md` AUTH-F01〜F06、SEC-001〜SEC-011
  - `30_arch/architecture.md` §5.1（認証エンドポイント）、§3.3（認証フロー）
  - `10_requirements/rbac.md` §3.1（認証権限）
- **含めるべき主要項目**: 各画面のレイアウト、入力項目、バリデーションルール、エラーメッセージ、成功時の遷移先
- **固有の注意点**:
  - パスワード最低8文字（SEC-010）
  - ログインエラーでユーザー存在を漏洩しない（SEC-011）
  - パスワードリセットは常に成功メッセージ返却（SEC-011）
  - サインアップはテナント+Adminユーザーを同時作成
  - パスワードリセットトークン有効期限1時間（SEC-006）
- **他タスクとの接点**: T1-8（セキュリティ設計）の認証セキュリティ仕様

##### T1-2: screens/dashboard.md（basic-designer）

- **読むべき入力と注目セクション**:
  - `40_basic_design/screens.md` §3.2（ダッシュボード）、ロール別表示内容
  - `10_requirements/usecases.md` UC-SYS04
  - `10_requirements/requirements.md` DASH-001〜DASH-005
- **含めるべき主要項目**: ロール別の表示エリア、データ項目、他画面へのリンク
- **判断ポイントの適用箇所**:
  - 月別サマリー期間 = **3ヶ月固定**
  - 直近レポート一覧 = **5件**
- **固有の注意点**:
  - Approver: Member情報 + 承認待ち件数 + 月別サマリー
  - Accounting: Member情報 + 支払待ち件数 + 月別サマリー
  - Admin: テナント全体のステータス別件数 + メンバー数 + 月別サマリー
- **他タスクとの接点**: T1-6（ダッシュボード集計クエリの設計）

##### T1-3: screens/report.md（basic-designer）— 最大・最重要

- **読むべき入力と注目セクション**:
  - `40_basic_design/screens.md` §3.3、§4.5〜4.9（共通UIパターン）
  - `10_requirements/usecases.md` UC-M01〜UC-M09
  - `10_requirements/requirements.md` RPT-F01〜F07、ITM-F01〜F03、ATT-F01〜F04
  - `10_requirements/workflow.md` 全体
  - `10_requirements/state_machine.md` 全体（操作マトリクス）
  - `20_domain/domain_model.md` §3.4〜3.6（ExpenseReport, ExpenseItem, Attachment）
  - `40_basic_design/ui_flow.md` Member フロー、再申請フロー
- **含めるべき主要項目**:
  - レポート一覧（SCR-RPT-001）: フィルタ、ページネーション、ステータスバッジ
  - レポート作成（SCR-RPT-002）: フォーム項目、バリデーション
  - レポート編集（SCR-RPT-003）: 作成と同構造＋プリフィル
  - レポート詳細（SCR-RPT-004）: 中心画面 — 明細一覧、添付、状態遷移、ロール別アクションボタン
- **判断ポイントの適用箇所**:
  - 明細編集は**スライドパネル**（モーダルや別ページではない）
  - カテゴリは**マスタテーブルからのドロップダウン**（6固定カテゴリ）
- **固有の注意点**:
  - 再申請フロー: タイトル/期間/明細をコピー、添付はコピー**しない**
  - 申請/削除/承認/却下/支払にはそれぞれ確認ダイアログ（`screens.md` §4.6 参照）
  - 楽観的ロック（updated_at チェック）
- **他タスクとの接点**: T1-6（DB スキーマ）、T1-7（添付アップロード/ダウンロードフロー）、T1-4（承認/却下アクション）

##### T1-4: screens/workflow.md（basic-designer）

- **読むべき入力と注目セクション**:
  - `40_basic_design/screens.md` §3.4
  - `10_requirements/usecases.md` UC-A01〜UC-A03、UC-AC01、UC-AC02
  - `10_requirements/workflow.md` 全体
  - `10_requirements/state_machine.md` T2/T3/T4 遷移
  - `10_requirements/rbac.md` §3.3（承認フロー権限）
- **含めるべき主要項目**:
  - 承認待ち一覧（SCR-WFL-001）: Approver 向け、submitted 状態のレポート
  - 支払待ち一覧（SCR-WFL-002）: Accounting 向け、approved 状態のレポート
  - いずれもレポート詳細画面へのリンク
- **固有の注意点**:
  - 承認/却下/支払のアクション自体は SCR-RPT-004（レポート詳細）で行う。この画面はリスト表示のみ
  - 自己承認禁止の表示ルール
  - 自己支払処理禁止の表示ルール
  - 却下理由は必須、承認コメントは任意
- **他タスクとの接点**: T1-3（レポート詳細画面のアクション）

##### T1-5: screens/admin.md（basic-designer）

- **読むべき入力と注目セクション**:
  - `40_basic_design/screens.md` §3.5
  - `10_requirements/usecases.md` UC-AD01、UC-AD02、UC-AC03
  - `10_requirements/rbac.md` §3.2、§3.5（Admin/Accounting 権限）
- **含めるべき主要項目**:
  - テナントレポート一覧（SCR-ADM-001）: フィルタ（ステータス・期間・申請者）、ページネーション
  - テナント情報（SCR-ADM-002）: 会社名（MVP では読み取り専用）
- **固有の注意点**:
  - SCR-ADM-001 は Admin **と** Accounting がアクセス可能
  - Admin はレポート閲覧のみ、編集/承認/却下は不可
  - テナント情報編集は Phase 3（MVP スコープ外）
- **他タスクとの接点**: T1-3（管理一覧からレポート詳細へのナビゲーション）

##### T1-6: db_schema.md（db-designer）

- **読むべき入力と注目セクション**:
  - `20_domain/domain_model.md` 全体（エンティティ定義、リレーション）
  - `10_requirements/state_machine.md` ステータス値
  - `30_arch/adr/0002-multi-tenant.md`（shared DB + tenant_id）
  - `30_arch/adr/0003-rls-tenant-isolation.md`（RLS ポリシー設計）
  - `.claude/rules/architecture.md`（アーキテクチャ制約）
  - `dev-journal/references/glossary.md`（テーブル/カラム命名）
- **含めるべき主要項目**:
  - テーブル定義（DDL レベル）: tenants, users, tenant_memberships, expense_reports, expense_items, attachments, categories
  - RLS ポリシー（テーブルごと）
  - インデックス戦略（tenant_id を複合インデックスのプレフィックスに）
  - マイグレーション戦略（golang-migrate）
  - 型マッピング（UUID, timestamps, enums）
  - 論理削除（deleted_at）、楽観的ロック（updated_at）
- **判断ポイントの適用箇所**:
  - カテゴリ: **マスタテーブル `categories`** + `expense_items.category_id` FK（enum ではない）
  - シード: 6固定カテゴリ（交通費, 宿泊費, 飲食費, 消耗品費, 通信費, その他）
  - Phase 3 考慮: カスタムカテゴリ対応のため categories テーブルに tenant_id（nullable または別テーブル）の設計ノートを含める
- **固有の注意点**:
  - Phase 3 用の audit_logs テーブルは INSERT ONLY の設計ノートのみ（MVP では未実装）
- **他タスクとの接点**: T1-3（カテゴリドロップダウンのデータソース）、T1-7（attachments テーブル、s3_key カラム）、T1-8（RLS、DBロール）

##### T1-7: files.md（detail-designer）

- **読むべき入力と注目セクション**:
  - `10_requirements/requirements.md` ATT-F01〜F04、ATT-002/003/010-014/020
  - `20_domain/domain_model.md` §3.6（Attachment エンティティ）
  - `.claude/rules/security-policy.md` §3、§5（**`file_id` を `attachment_id` に読み替え — issue 036**）
  - `30_arch/architecture.md` §2（S3 インフラ）、§5.1（添付エンドポイント）
  - `30_arch/adr/0004-infra.md`（S3 設定）
- **含めるべき主要項目**:
  - S3 バケット構造（単一バケット、キーフォーマット）
  - アップロードフロー（API プロキシ）
  - ダウンロードフロー（署名付き GET URL、15分有効）
  - MIME タイプ検証（JPEG, PNG, PDF）+ マジックバイトチェック
  - ファイルサイズ上限（5MB）
  - 署名付き URL 発行前の認可チェック
  - 論理削除の挙動（DB 論理削除、S3 物理削除戦略）
- **判断ポイントの適用箇所**:
  - S3 キー: **`{tenant_id}/{report_id}/{attachment_id}`**（issue 036 で統一）
  - アップロード方式: **API 経由プロキシ**（署名付きアップロード URL ではない）
- **固有の注意点**:
  - クライアント提供のファイル名を S3 キーに使用しない
  - 署名付き URL 有効期限: 15分（ATT-012）
- **他タスクとの接点**: T1-6（attachments テーブルスキーマ）、T1-8（ファイルセキュリティ、バケットポリシー）、T1-3（レポート詳細の添付UI）

##### T1-8: security.md（detail-designer）

- **読むべき入力と注目セクション**:
  - `10_requirements/requirements.md` §4.2（セキュリティ要件）、SEC-* ルール
  - `30_arch/architecture.md` §6（セキュリティアーキテクチャ）
  - `30_arch/adr/0001-tech-stack.md`（JWT ライブラリ、Argon2id ライブラリ）
  - `.claude/rules/security-policy.md` 全体
  - `10_requirements/rbac.md` §5（RBAC + テナント分離フロー）
- **含めるべき主要項目**:
  - JWT 実装詳細（RS256 鍵管理、クレーム構造、ローテーション）
  - レート制限の具体値と実装方式
  - CORS ポリシー
  - セキュリティヘッダー一覧と値
  - 入力バリデーション戦略（バックエンドを信頼境界とする）
  - パスワードハッシュパラメータ（Argon2id パラメータ）
  - エラーレスポンスのサニタイズ（内部情報を含めない）
- **固有の注意点**:
  - `security-policy.md` §3, §5 の `file_id` は `attachment_id` に読み替え
  - レート制限: 100 req/min/user（認証済み）、20 req/min/IP（未認証）、5 req/min/IP（ログイン）、10 req/min/user（ファイルアップロード）
  - テナント境界越えは 404（403 ではない）
- **他タスクとの接点**: T1-6（RLS ポリシー、DBロール）、T1-7（ファイルアップロードセキュリティ）、T1-1（認証画面セキュリティ）

##### T1-9: monitoring.md（detail-designer）

- **読むべき入力と注目セクション**:
  - `30_arch/adr/0005-monitoring-logging.md` 全体（主要参照）
  - `30_arch/architecture.md` §7（監視・ログポリシー）
  - `30_arch/adr/0004-infra.md`（CloudWatch 統合）
- **含めるべき主要項目**:
  - 構造化ログのフィールド定義
  - ログレベルポリシー（例付き）
  - メトリクス定義と収集方法
  - アラート閾値と通知先
  - ヘルスチェックエンドポイント仕様
  - ダッシュボード設計（CloudWatch Logs Insights クエリ）
  - セキュリティログ要件（ログに含めてはいけないもの）
  - ログ保持ポリシー
- **固有の注意点**:
  - slog（Go 標準ライブラリ）+ JSON 出力
  - stdout → CloudWatch Logs（ECS 自動転送）
  - メトリクスは CloudWatch メトリクスフィルタ（カスタム SDK ではない）
  - Logs Insights でアドホックダッシュボード、メトリクスフィルタでアラーム
- **他タスクとの接点**: T1-8（セキュリティログ要件）、T1-6（DB 接続監視）

## 品質基準

- **上流整合性**: 入力として参照した上流成果物と矛盾がないか
- **タスク間整合性**: 並列タスクの成果物間で用語・ID・参照関係が統一されているか
- **MVP スコープ**: `deliverables/docs/02_scope.md` の範囲内か
- **用語集準拠**: `dev-journal/references/glossary.md` の用語を使用しているか
- **完了条件**: work-breakdown に定義された完了条件を満たしているか

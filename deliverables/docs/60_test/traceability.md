# トレーサビリティ

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 要件・設計・テストの対応関係を一元的に示す |
| 正本情報 | 要件ID → 設計反映先 → テスト反映先 |
| 扱わない内容 | 個別設計の本文 |
| 主な参照元 | `10_requirements/*`, `20_domain/*`, `30_arch/*`, `50_detail_design/*`, `60_test/test_cases/*.md` |
| 主な参照先 | レビュー、テスト実装、差分影響分析 |

---

## 1. 機能要件トレーサビリティ

### 1.1 認証・ユーザー管理（AUTH-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| AUTH-F01 | サインアップ（テナント + Admin ユーザー同時作成） | `openapi.yaml#signup`, `db_schema.md#tenants`, `db_schema.md#users`, `db_schema.md#tenant_memberships` | `test_cases/auth.md#AUTH-024〜033` | `test_cases/auth.md#AUTH-FE-025〜042` | |
| AUTH-F02 | ログイン（JWT 発行） | `openapi.yaml#login`, `security.md#2.1` | `test_cases/auth.md#AUTH-034〜041` | `test_cases/auth.md#AUTH-FE-008〜024` | |
| AUTH-F03 | トークンリフレッシュ | `openapi.yaml#refreshToken`, `db_schema.md#refresh_tokens` | `test_cases/auth.md#AUTH-042〜050` | - | |
| AUTH-F04 | ログアウト（リフレッシュトークン無効化） | `openapi.yaml#logout`, `db_schema.md#refresh_tokens` | `test_cases/auth.md#AUTH-051〜056` | - | |
| AUTH-F05 | 認証情報取得（現在ユーザー情報） | `openapi.yaml#getMe` | `test_cases/auth.md#AUTH-057〜064` | `test_cases/dashboard.md#DSH-FE-037〜DSH-FE-038` | |
| AUTH-F06 | パスワードリセット | `openapi.yaml#requestPasswordReset`, `openapi.yaml#executePasswordReset`, `db_schema.md#password_reset_tokens`, `security.md#2.3` | `test_cases/auth.md#AUTH-065〜076` | `test_cases/auth.md#AUTH-FE-043〜076` | |

### 1.2 RBAC（RBAC-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| RBAC-F01 | ロール検証ミドルウェア（全 API でロール検証） | `authz.md#3`, `openapi.yaml#x-authz` | `test_cases/cross-cutting.md#CRS-021〜054` | `test_cases/tenant.md#TNT-FE-002〜005, TNT-FE-018〜019`<br>`test_cases/workflow.md#WFL-FE-005, WFL-FE-034, WFL-FE-057` | |
| RBAC-F02 | リソース所有者チェック | `authz.md#4`, `openapi.yaml#x-authz.ownership_required` | `test_cases/cross-cutting.md#CRS-021〜054`, `test_cases/reports.md#RPT-036, RPT-038, RPT-043〜044, RPT-054` | `test_cases/reports.md#RPT-FE-053, RPT-FE-083` | |

### 1.3 テナント分離（TNT-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| TNT-F01 | tenant_id 自動付与 | `db_schema.md#全業務テーブル`, `authz.md#5` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-F02 | tenant_id フィルタ（全クエリ） | `authz.md#5.1`, `db_schema.md#RLS` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-F03 | RLS ポリシー（DB 層テナント分離） | `db_schema.md#RLS設定` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |

### 1.4 経費レポート CRUD（RPT-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| RPT-F01 | レポート作成（draft で作成） | `openapi.yaml#createReport`, `db_schema.md#expense_reports` | `test_cases/reports.md#RPT-008〜018` | `test_cases/reports.md#RPT-FE-005, RPT-FE-009〜010, RPT-FE-024〜049, RPT-FE-095` | |
| RPT-F02 | 自分のレポート一覧取得 | `openapi.yaml#listMyReports` | `test_cases/reports.md#RPT-001〜007` | `test_cases/reports.md#RPT-FE-001〜023` | |
| RPT-F03 | レポート詳細取得 | `openapi.yaml#getReport` | `test_cases/reports.md#RPT-027〜034` | `test_cases/reports.md#RPT-FE-004, RPT-FE-018, RPT-FE-059〜060, RPT-FE-064〜086, RPT-FE-088, RPT-FE-090〜091` | |
| RPT-F04 | レポート編集（draft のみ） | `openapi.yaml#updateReport` | `test_cases/reports.md#RPT-035〜041` | `test_cases/reports.md#RPT-FE-050〜058, RPT-FE-061〜063, RPT-FE-092` | |
| RPT-F05 | レポート削除（draft のみ、論理削除） | `openapi.yaml#deleteReport` | `test_cases/reports.md#RPT-045〜052` | `test_cases/reports.md#RPT-FE-068, RPT-FE-094, RPT-FE-100〜102` | |
| RPT-F06 | レポート提出（draft → submitted） | `openapi.yaml#submitReport` | `test_cases/reports.md#RPT-053〜065` | `test_cases/reports.md#RPT-FE-067, RPT-FE-069, RPT-FE-087〜093, RPT-FE-096〜099` | |
| RPT-F07 | テナント全レポート一覧取得（Admin + Accounting） | `openapi.yaml#listAllReports` | `test_cases/reports.md#RPT-020〜026` | `test_cases/tenant.md#TNT-FE-016〜023` | |

### 1.5 経費明細 CRUD（ITM-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| ITM-F01 | 明細追加（draft レポートに追加） | `openapi.yaml#createItem`, `db_schema.md#expense_items` | `test_cases/items.md#ITM-001〜099` | `test_cases/items.md#ITM-FE-001〜050` | |
| ITM-F02 | 明細編集（draft レポートの明細編集） | `openapi.yaml#updateItem` | `test_cases/items.md#ITM-101〜199` | `test_cases/items.md#ITM-FE-015, ITM-FE-017, ITM-FE-021, ITM-FE-024, ITM-FE-038, ITM-FE-041, ITM-FE-051〜053` | |
| ITM-F03 | 明細削除（draft レポートの明細削除） | `openapi.yaml#deleteItem` | `test_cases/items.md#ITM-201〜299` | `test_cases/items.md#ITM-FE-016, ITM-FE-017, ITM-FE-054〜056` | |

### 1.6 承認フロー（WFL-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| WFL-F01 | 承認（submitted → approved） | `openapi.yaml#approveReport`, `db_schema.md#expense_reports.approved_by` | `test_cases/workflow.md#WFL-010〜025` | `test_cases/workflow.md#WFL-FE-055, WFL-FE-059〜060, WFL-FE-063, WFL-FE-066, WFL-FE-069〜072` | |
| WFL-F02 | 却下（submitted → rejected） | `openapi.yaml#rejectReport`, `db_schema.md#expense_reports.rejected_by` | `test_cases/workflow.md#WFL-026〜041` | `test_cases/workflow.md#WFL-FE-055, WFL-FE-064, WFL-FE-067, WFL-FE-073〜075` | |
| WFL-F03 | 支払完了（approved → paid） | `openapi.yaml#markReportAsPaid`, `db_schema.md#expense_reports.paid_by` | `test_cases/workflow.md#WFL-051〜063` | `test_cases/workflow.md#WFL-FE-056, WFL-FE-058, WFL-FE-061〜062, WFL-FE-065, WFL-FE-068, WFL-FE-076〜078` | |
| WFL-F04 | 承認待ち一覧（Approver 用） | `openapi.yaml#listPendingReports` | `test_cases/workflow.md#WFL-001〜009` | `test_cases/workflow.md#WFL-FE-001〜029` | |
| WFL-F05 | 支払待ち一覧（Accounting 用） | `openapi.yaml#listPayableReports` | `test_cases/workflow.md#WFL-042〜050` | `test_cases/workflow.md#WFL-FE-030〜054, WFL-FE-079〜082` | |

### 1.7 添付ファイル（ATT-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| ATT-F01 | ファイルアップロード（S3 + DB メタデータ） | `openapi.yaml#uploadAttachment`, `files.md`, `db_schema.md#attachments` | `test_cases/attachments.md#ATT-001〜019` | `test_cases/attachments.md#ATT-FE-001〜002, ATT-FE-004〜005, ATT-FE-016〜020, ATT-FE-025, ATT-FE-027〜028, ATT-FE-036〜037, ATT-FE-045` | |
| ATT-F02 | ファイル一覧取得 | `openapi.yaml#listAttachments` | `test_cases/attachments.md#ATT-020〜029` | `test_cases/attachments.md#ATT-FE-001, ATT-FE-005, ATT-FE-007〜009, ATT-FE-029〜032` | |
| ATT-F03 | ファイルダウンロード（署名付き URL 発行） | `openapi.yaml#getAttachmentDownload`, `files.md#署名付きURL` | `test_cases/attachments.md#ATT-030〜041` | `test_cases/attachments.md#ATT-FE-010, ATT-FE-033〜035, ATT-FE-049` | |
| ATT-F04 | ファイル削除（論理削除） | `openapi.yaml#deleteAttachment` | `test_cases/attachments.md#ATT-042〜054` | `test_cases/attachments.md#ATT-FE-006, ATT-FE-013〜015, ATT-FE-041〜044, ATT-FE-047〜048` | |

### 1.8 ダッシュボード（DASH-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| DASH-F01 | ダッシュボード取得（ロール別集計） | `openapi.yaml#getDashboard` | `test_cases/dashboard.md#DSH-001〜018` | `test_cases/dashboard.md#DSH-FE-001〜038` | |

### 1.9 テナント管理（ADM-F*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| ADM-F01 | テナント情報取得 | `openapi.yaml#getTenant`, `db_schema.md#tenants` | `test_cases/tenant.md#TNT-001〜010` | `test_cases/tenant.md#TNT-FE-001〜043` | |

---

## 2. ポリシートレーサビリティ

### 2.1 RBAC ルール（RBC-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| RBC-001 | 全 API でミドルウェアによるロール検証 | `authz.md#3`, `openapi.yaml#x-authz` | `test_cases/cross-cutting.md#CRS-021〜054` | `test_cases/tenant.md#TNT-FE-002〜005, TNT-FE-018〜019`<br>`test_cases/workflow.md#WFL-FE-005, WFL-FE-034, WFL-FE-057` | |
| RBC-002 | 1 ユーザーは 1 テナントにつき 1 ロールのみ | `db_schema.md#tenant_memberships#UNIQUE` | `test_cases/tenant.md#TNT-006〜008` | - | |
| RBC-003 | リソースアクセスは「ロール」と「所有権」の両方で制御 | `authz.md#4` | `test_cases/cross-cutting.md#CRS-021〜054` | `test_cases/reports.md#RPT-FE-053, RPT-FE-083` | |
| RBC-004 | 同一テナント内の権限不足時は 403 Forbidden | `authz.md#3.4`, `openapi.yaml` | `test_cases/cross-cutting.md#CRS-021〜054` | `test_cases/tenant.md#TNT-FE-005, TNT-FE-020`<br>`test_cases/workflow.md#WFL-FE-005, WFL-FE-034` | |
| RBC-010 | 申請系操作は所有者のレポートに限定 | `authz.md#4.1`, `openapi.yaml#x-authz.ownership_required` | `test_cases/reports.md#RPT-054`, `test_cases/cross-cutting.md#CRS-051〜052` | `test_cases/reports.md#RPT-FE-053, RPT-FE-081` | |
| RBC-011 | Approver は同テナント submitted レポートを承認/却下可能 | `authz.md#4.2`, `openapi.yaml#approveReport`, `openapi.yaml#rejectReport` | `test_cases/workflow.md#WFL-010〜012, WFL-026`, `test_cases/cross-cutting.md#CRS-021〜054` | `test_cases/workflow.md#WFL-FE-055, WFL-FE-063〜064` | |
| RBC-012 | Accounting の自己支払処理禁止 | `authz.md#4.3`, `openapi.yaml#markReportAsPaid` | `test_cases/workflow.md#WFL-056`, `test_cases/cross-cutting.md#CRS-050` | `test_cases/workflow.md#WFL-FE-058, WFL-FE-079〜080` | |
| RBC-013 | Admin は同テナント全レポートを閲覧可能（編集不可） | `authz.md#4.4`, `openapi.yaml#listAllReports`, `openapi.yaml#getReport` | `test_cases/reports.md#RPT-020〜021, RPT-032`, `test_cases/cross-cutting.md#CRS-023, CRS-053` | `test_cases/tenant.md#TNT-FE-016` | |
| RBC-014 | Admin は自分のレポートのみ申請者操作可能 | `authz.md#4.4` | `test_cases/reports.md#RPT-013, RPT-038, RPT-044`, `test_cases/cross-cutting.md#CRS-051〜052` | - | |
| RBC-015 | Approver は全件承認型（MVP） | `authz.md#4.2`, `openapi.yaml#listPendingReports` | `test_cases/workflow.md#WFL-001〜005` | `test_cases/workflow.md#WFL-FE-001〜004` | |
| RBC-016 | Approver の自己承認・自己却下禁止 | `authz.md#4.2`, `openapi.yaml#approveReport`, `openapi.yaml#rejectReport` | `test_cases/workflow.md#WFL-018, WFL-035`, `test_cases/cross-cutting.md#CRS-048〜049` | `test_cases/workflow.md#WFL-FE-057`<br>`test_cases/reports.md#RPT-FE-083` | |

### 2.2 ワークフロー・状態遷移ルール（WFL-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| WFL-001 | 状態遷移はドメイン層で一元管理 | `20_domain/state_machine.md`, `db_schema.md#expense_reports.status` | `test_cases/reports.md#RPT-065〜090` | - | |
| WFL-002 | 許可される遷移のみ実行可能 | `20_domain/state_machine.md#許可遷移`, `db_schema.md#CHECK制約` | `test_cases/reports.md#RPT-073〜082`, `test_cases/workflow.md#WFL-013〜016, WFL-031〜034, WFL-052〜055` | - | |
| WFL-003 | 状態遷移を実行できるロールは遷移ごとに定義 | `20_domain/state_machine.md`, `authz.md` | `test_cases/workflow.md#WFL-019〜022, WFL-036〜039, WFL-057〜060` | `test_cases/workflow.md#WFL-FE-055〜062` | |
| WFL-004 | 終端状態（rejected, paid）からの遷移は不可 | `20_domain/state_machine.md#X9`, `20_domain/state_machine.md#X10` | `test_cases/reports.md#RPT-081〜082`, `test_cases/workflow.md#WFL-015〜016, WFL-033〜034, WFL-054〜055` | `test_cases/reports.md#RPT-FE-086, RPT-FE-091`<br>`test_cases/workflow.md#WFL-FE-062` | |
| WFL-010 | draft → submitted 遷移（所有者、明細1件以上） | `openapi.yaml#submitReport`, `20_domain/state_machine.md#T1` | `test_cases/reports.md#RPT-053〜065` | `test_cases/reports.md#RPT-FE-067, RPT-FE-088〜089, RPT-FE-093, RPT-FE-097〜099`<br>`test_cases/cross-cutting.md#CRS-059, CRS-072` | |
| WFL-011 | submitted → approved 遷移（Approver、自己承認禁止） | `openapi.yaml#approveReport`, `20_domain/state_machine.md#T2` | `test_cases/workflow.md#WFL-010〜012, WFL-018`, `test_cases/reports.md#RPT-083〜085` | `test_cases/workflow.md#WFL-FE-055, WFL-FE-063, WFL-FE-069〜072`<br>`test_cases/cross-cutting.md#CRS-062` | |
| WFL-012 | submitted → rejected 遷移（却下理由必須） | `openapi.yaml#rejectReport`, `20_domain/state_machine.md#T3` | `test_cases/workflow.md#WFL-026〜030`, `test_cases/reports.md#RPT-086〜088` | `test_cases/workflow.md#WFL-FE-055, WFL-FE-064, WFL-FE-073〜075`<br>`test_cases/cross-cutting.md#CRS-068` | |
| WFL-013 | approved → paid 遷移（Accounting、自己処理禁止） | `openapi.yaml#markReportAsPaid`, `20_domain/state_machine.md#T4` | `test_cases/workflow.md#WFL-051, WFL-056`, `test_cases/reports.md#RPT-089〜090` | `test_cases/workflow.md#WFL-FE-056, WFL-FE-065, WFL-FE-076〜078`<br>`test_cases/cross-cutting.md#CRS-065` | |
| WFL-014 | 提出時に同テナントに Approver が 1 人以上必要 | `openapi.yaml#submitReport` | `test_cases/reports.md#RPT-057, RPT-071` | - | |
| WFL-015 | draft → (削除) 遷移（所有者、論理削除） | `openapi.yaml#deleteReport`, `20_domain/state_machine.md#T5` | `test_cases/reports.md#RPT-045〜052` | `test_cases/reports.md#RPT-FE-068, RPT-FE-094, RPT-FE-100〜102` | |

### 2.3 テナント分離ルール（TNT-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| TNT-001 | 業務テーブルに tenant_id カラム必須 | `db_schema.md#全業務テーブル` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-002 | 全クエリに WHERE tenant_id = ? 必須 | `authz.md#5.1` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-003 | tenant_id 付与・検証はリポジトリ層で強制 | `authz.md#5.2` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-004 | PostgreSQL RLS でアプリ層を二重化 | `db_schema.md#RLS設定` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-005 | テナント間データ参照は一切不可 | `authz.md#5`, `db_schema.md#RLS設定` | `test_cases/cross-cutting.md#CRS-001〜020` | - | |
| TNT-006 | テナント境界越えアクセスは 404 を返却 | `authz.md#5.3` | `test_cases/cross-cutting.md#CRS-001〜016` | - | |

### 2.4 認証・セキュリティルール（SEC-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| SEC-001 | 認証方式はメール + パスワード | `openapi.yaml#login`, `security.md#2` | `test_cases/auth.md#AUTH-034〜041` | `test_cases/auth.md#AUTH-FE-008〜024` | |
| SEC-002 | パスワードハッシュは Argon2id | `security.md#2.2`, `db_schema.md#users.password_hash` | `test_cases/auth.md#AUTH-001〜007` | - | |
| SEC-003 | アクセストークン 15 分、リフレッシュトークン 7 日 | `security.md#2.1`, `db_schema.md#refresh_tokens` | `test_cases/auth.md#AUTH-010`, `test_cases/auth.md#AUTH-013` | - | |
| SEC-004 | JWT 署名アルゴリズム: RS256 | `security.md#2.1` | `test_cases/auth.md#AUTH-017`, `test_cases/auth.md#AUTH-078` | - | |
| SEC-005 | ログアウト時にリフレッシュトークンを無効化 | `openapi.yaml#logout`, `db_schema.md#refresh_tokens` | `test_cases/auth.md#AUTH-051〜056` | - | |
| SEC-006 | パスワードリセットトークンは 1 回使用で無効化 | `openapi.yaml#executePasswordReset`, `db_schema.md#password_reset_tokens` | `test_cases/auth.md#AUTH-073` | - | |
| SEC-010 | パスワード最小長: 8 文字以上 | `openapi.yaml#signup` | `test_cases/auth.md#AUTH-029` | `test_cases/auth.md#AUTH-FE-036〜037, AUTH-FE-067` | |
| SEC-011 | 認証失敗時のレスポンスでユーザーの存在を漏らさない | `openapi.yaml#login`, `security.md#2.4` | `test_cases/auth.md#AUTH-036`, `test_cases/auth.md#AUTH-066` | `test_cases/auth.md#AUTH-FE-010, AUTH-FE-019, AUTH-FE-022, AUTH-FE-043, AUTH-FE-054` | |

### 2.5 添付ファイルルール（ATT-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| ATT-002 | 許可ファイル形式: JPEG, PNG, PDF | `openapi.yaml#uploadAttachment`, `files.md#3`, `db_schema.md#attachments#CHECK` | `test_cases/attachments.md#ATT-001〜003, ATT-007〜008` | `test_cases/attachments.md#ATT-FE-018〜022, ATT-FE-025〜026` | |
| ATT-003 | 1 ファイルサイズ上限: 5MB | `openapi.yaml#uploadAttachment`, `db_schema.md#attachments#CHECK` | `test_cases/attachments.md#ATT-004, ATT-006` | `test_cases/attachments.md#ATT-FE-023〜024, ATT-FE-039, ATT-FE-050` | |
| ATT-010 | ファイルダウンロードは署名付き URL 経由 | `openapi.yaml#getAttachmentDownload`, `files.md#署名付きURL` | `test_cases/attachments.md#ATT-030〜034` | `test_cases/attachments.md#ATT-FE-033, ATT-FE-049` | |
| ATT-011 | 署名付き URL 発行前に認可チェック必須 | `openapi.yaml#getAttachmentDownload`, `authz.md#6.5` | `test_cases/attachments.md#ATT-035〜038` | - | |
| ATT-012 | 署名付き URL の有効期限: 15 分 | `files.md#署名付きURL`, `openapi.yaml#getAttachmentDownload` | `test_cases/attachments.md#ATT-034` | `test_cases/attachments.md#ATT-FE-034` | |
| ATT-013 | アップロード時に MIME タイプを検証 | `openapi.yaml#uploadAttachment`, `files.md#3.2` | `test_cases/attachments.md#ATT-007〜010` | `test_cases/attachments.md#ATT-FE-021〜022, ATT-FE-026, ATT-FE-038, ATT-FE-046` | |
| ATT-014 | S3 パスにテナント ID を含む | `files.md#3.3`, `db_schema.md#attachments.s3_key` | `test_cases/attachments.md#ATT-001` | - | S3 パスの検証は ATT-001 のレコード挿入時に暗黙的に確認 |
| ATT-020 | 添付の追加・削除は draft 状態のみ | `openapi.yaml#uploadAttachment`, `openapi.yaml#deleteAttachment` | `test_cases/attachments.md#ATT-013〜016, ATT-048〜051` | `test_cases/attachments.md#ATT-FE-003〜004, ATT-FE-011〜012, ATT-FE-040, ATT-FE-044` | |

### 2.6 経費レポートルール（RPT-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| RPT-001 | レポートにはタイトルが必須 | `openapi.yaml#createReport`, `db_schema.md#expense_reports.title` | `test_cases/reports.md#RPT-009` | `test_cases/reports.md#RPT-FE-029` | |
| RPT-002 | レポートには対象期間（開始日・終了日）が必須 | `openapi.yaml#createReport`, `db_schema.md#expense_reports` | `test_cases/reports.md#RPT-008, RPT-010` | `test_cases/reports.md#RPT-FE-030〜033` | |
| RPT-003 | 対象期間の開始日は終了日以前 | `openapi.yaml#createReport`, `db_schema.md#expense_reports#CHECK` | `test_cases/reports.md#RPT-010` | `test_cases/reports.md#RPT-FE-031` | |
| RPT-006 | 合計金額は明細の金額合計から自動計算 | `db_schema.md#expense_reports.total_amount` | `test_cases/items.md#ITM-002` | - | |
| RPT-011 | レポート編集は draft 状態のみ | `openapi.yaml#updateReport` | `test_cases/reports.md#RPT-037` | `test_cases/reports.md#RPT-FE-054` | |
| RPT-012 | レポート編集は所有者のみ | `openapi.yaml#updateReport`, `authz.md#4.1` | `test_cases/reports.md#RPT-036, RPT-038` | `test_cases/reports.md#RPT-FE-052〜053` | |
| RPT-013 | レポート削除は draft 状態のみ | `openapi.yaml#deleteReport` | `test_cases/reports.md#RPT-045, RPT-048〜052` | - | |
| RPT-014 | レポート提出には明細が 1 件以上必要 | `openapi.yaml#submitReport` | `test_cases/reports.md#RPT-056, RPT-066` | `test_cases/reports.md#RPT-FE-089` | |
| RPT-015 | 却下後の再申請は新規レポートとして作成 | `openapi.yaml#createReport` | `test_cases/reports.md#RPT-014〜019` | `test_cases/reports.md#RPT-FE-026, RPT-FE-036, RPT-FE-047, RPT-FE-084, RPT-FE-090, RPT-FE-095` | |
| RPT-016 | 再申請レポートは元レポートへの参照を保持 | `db_schema.md#expense_reports.reference_report_id` | `test_cases/reports.md#RPT-014, RPT-016` | `test_cases/reports.md#RPT-FE-079〜080` | |

### 2.7 経費明細ルール（ITM-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| ITM-001 | 明細には日付が必須 | `openapi.yaml#createItem`, `db_schema.md#expense_items.expense_date` | `test_cases/items.md#ITM-015〜016` | `test_cases/items.md#ITM-FE-028` | |
| ITM-002 | 明細には金額が必須（正の整数値） | `openapi.yaml#createItem`, `db_schema.md#expense_items.amount#CHECK` | `test_cases/items.md#ITM-006, ITM-011〜014, ITM-111〜112` | `test_cases/items.md#ITM-FE-011, ITM-FE-029〜032` | |
| ITM-003 | 明細にはカテゴリが必須 | `openapi.yaml#createItem`, `db_schema.md#expense_items.category_id` | `test_cases/items.md#ITM-017〜019` | `test_cases/items.md#ITM-FE-033` | |
| ITM-004 | 明細には摘要が必須 | `openapi.yaml#createItem`, `db_schema.md#expense_items.description` | `test_cases/items.md#ITM-020〜023` | `test_cases/items.md#ITM-FE-034〜036` | |
| ITM-005 | カテゴリは固定 6 種類 | `openapi.yaml#listCategories`, `db_schema.md#categories` | `test_cases/dashboard.md#DSH-019〜020` | `test_cases/items.md#ITM-FE-047, ITM-FE-057〜059` | |
| ITM-010 | 明細の追加・編集・削除は draft 状態のみ | `openapi.yaml#createItem`, `openapi.yaml#updateItem`, `openapi.yaml#deleteItem` | `test_cases/items.md#ITM-061〜065, ITM-161〜165, ITM-241〜245` | `test_cases/items.md#ITM-FE-004, ITM-FE-008, ITM-FE-012, ITM-FE-013` | |

### 2.8 ダッシュボードルール（DASH-*）

| ルールID | ルール概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|---------|-----------|-----------|---------------|---------------|------|
| DASH-001 | Member には自分の draft / submitted / rejected 件数を表示 | `openapi.yaml#getDashboard` | `test_cases/dashboard.md#DSH-003〜005` | `test_cases/dashboard.md#DSH-FE-001〜002, DSH-FE-016〜018, DSH-FE-027〜029` | |
| DASH-002 | Approver には Member 情報 + 承認待ち件数を追加表示 | `openapi.yaml#getDashboard` | `test_cases/dashboard.md#DSH-006〜009` | `test_cases/dashboard.md#DSH-FE-003` | |
| DASH-003 | Accounting には Member 情報 + 支払待ち件数を追加表示 | `openapi.yaml#getDashboard` | `test_cases/dashboard.md#DSH-010〜013` | `test_cases/dashboard.md#DSH-FE-004` | |
| DASH-004 | Admin にはテナント全体の件数 + メンバー数を表示 | `openapi.yaml#getDashboard` | `test_cases/dashboard.md#DSH-014〜017` | `test_cases/dashboard.md#DSH-FE-005, DSH-FE-015, DSH-FE-019〜021` | |
| DASH-005 | Approver / Accounting / Admin にテナント全体の月別支出合計を表示 | `openapi.yaml#getDashboard` | `test_cases/dashboard.md#DSH-009`, `test_cases/dashboard.md#DSH-012`, `test_cases/dashboard.md#DSH-016` | `test_cases/dashboard.md#DSH-FE-022〜026` | |

---

## 3. 非機能要件トレーサビリティ

### 3.1 パフォーマンス（NFR-PERF-*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| NFR-PERF-001 | API レスポンスタイム p95 500ms 以下 | `architecture.md#8.1`, ADR-0001, ADR-0005 | `test_cases/cross-cutting.md#CRS-083〜087` | - | |
| NFR-PERF-002 | ファイルアップロード 5秒以下（5MB） | `architecture.md#8.1`, `files.md`, ADR-0004 | `test_cases/cross-cutting.md#CRS-088` | - | |
| NFR-PERF-003 | 同時接続ユーザー数 100 | `architecture.md#8.1`, ADR-0002, ADR-0004 | - | - | 本格負荷テストは MVP スコープ外。CRS-083〜088 の軽量スモークテストで代替 |
| NFR-PERF-004 | オフセットベースページネーション（デフォルト20件） | `architecture.md#5.3`, `openapi.yaml` | `test_cases/reports.md#RPT-001〜003`, `test_cases/workflow.md#WFL-005, WFL-046` | - | |

### 3.2 セキュリティ（NFR-SEC-*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| NFR-SEC-001 | JWT RS256 認証 | `security.md#2.1`, `openapi.yaml#securitySchemes` | `test_cases/auth.md#AUTH-034〜041` | - | ルールID: SEC-003, SEC-004 |
| NFR-SEC-002 | パスワード Argon2id ハッシュ、8文字以上 | `security.md#2.2` | `test_cases/auth.md#AUTH-024〜033` | - | ルールID: SEC-002, SEC-010 |
| NFR-SEC-003 | テナント分離（アプリ層 + RLS 二重保証） | `db_schema.md#RLS設定`, `authz.md#9` | `test_cases/cross-cutting.md#CRS-001〜020` | - | ルールID: TNT-001〜005 |
| NFR-SEC-004 | RBAC 全APIでミドルウェア検証 | `authz.md#3`, `architecture.md#3.2` | `test_cases/cross-cutting.md#CRS-021〜075` | `test_cases/cross-cutting.md#CRS-055〜075` | ルールID: RBC-001 |
| NFR-SEC-005 | レート制限（認証済み: 100 req/min/user、未認証: 20 req/min/IP） | `security.md#4.1`, `openapi.yaml#x-ratelimit` | `test_cases/cross-cutting.md#CRS-076〜088` | - | ルールID: SEC-012 |
| NFR-SEC-006 | CORS（許可オリジンを明示指定） | `security.md#4.2` | `test_cases/cross-cutting.md#CRS-076〜088` | - | ルールID: SEC-013 |
| NFR-SEC-007 | セキュリティヘッダー（HSTS, X-Content-Type-Options, X-Frame-Options） | `security.md#4.3` | `test_cases/cross-cutting.md#CRS-076〜088` | - | ルールID: SEC-014 |
| NFR-SEC-008 | ファイルアップロード MIMEタイプ検証、5MB制限 | `files.md#3`, `openapi.yaml#uploadAttachment` | `test_cases/attachments.md#ATT-004, ATT-006〜010` | `test_cases/attachments.md#ATT-FE-018〜026, ATT-FE-038〜039` | ルールID: ATT-013, ATT-003 |

### 3.3 可用性（NFR-AVAIL-*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| NFR-AVAIL-001 | 稼働率 99.5% | `architecture.md#8.3`, ADR-0004 | - | - | 運用監視で確認。自動テスト対象外（インフラ SLA の計測は本番環境でのみ可能） |
| NFR-AVAIL-002 | ヘルスチェックエンドポイント `/health` | `openapi.yaml#getHealth`, `architecture.md#8.3`, ADR-0005 | - | - | Step 9 以降で実装時にヘルスチェックテストを追加 |
| NFR-AVAIL-003 | DB バックアップ RDS 自動バックアップ 7日間保持 | `architecture.md#8.3`, ADR-0004 | - | - | 運用確認項目 |

### 3.4 データ（NFR-DATA-*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| NFR-DATA-001 | 論理削除（deleted_at タイムスタンプ方式） | `db_schema.md#全業務テーブル` | `test_cases/reports.md#RPT-045〜052` | - | ルールID: DAT-002 |
| NFR-DATA-002 | 全レコードに created_at, updated_at | `db_schema.md#全業務テーブル` | `test_cases/reports.md#RPT-008, RPT-035`, `test_cases/items.md#ITM-001, ITM-101` | - | ルールID: DAT-004。各 CRUD の正常系レスポンスで created_at/updated_at の存在を暗黙的に検証 |
| NFR-DATA-003 | 提出以降のレポートは物理削除不可 | `db_schema.md#expense_reports.deleted_at` | `test_cases/reports.md#RPT-045, RPT-049〜052` | - | ルールID: DAT-001 |
| NFR-DATA-004 | MVP監査証跡（改ざんしにくいデータ構造） | `db_schema.md#expense_reports` | `test_cases/reports.md#RPT-053, RPT-065` | - | submitted_at/approved_at/rejected_at/paid_at の記録を検証 |
| NFR-DATA-005 | 日本円（JPY）のみ、整数値 | `db_schema.md#expense_items.amount` | `test_cases/items.md#ITM-006, ITM-011〜014` | `test_cases/items.md#ITM-FE-029〜032` | ルールID: ITM-002 |
| NFR-DATA-006 | 文字コード UTF-8 | `db_schema.md#CREATE DATABASE` | - | - | 自動テスト対象外（DB 設定で保証。マイグレーション時に確認） |
| NFR-DATA-007 | タイムゾーン UTC保存、表示時JST変換 | `architecture.md#4.2` | - | - | 自動テスト対象外（アプリケーション実装規約で保証。Step 9 以降でタイムスタンプ形式の検証を含める） |

### 3.5 ユーザビリティ（NFR-UX-*）

| 要件ID | 要件概要 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| NFR-UX-001 | レスポンシブデザイン（PC・タブレット・スマートフォン対応） | `architecture.md#8.5`, ADR-0001（MUI 選定） | `test_cases/cross-cutting.md#CRS-055〜075` | `test_cases/cross-cutting.md#CRS-055〜075` | E2E テストで主要フローの画面表示を検証 |
| NFR-UX-002 | UI 日本語固定 | `architecture.md#8.5` | `test_cases/cross-cutting.md#CRS-055〜075` | `test_cases/cross-cutting.md#CRS-055〜075` | E2E テストで日本語表示を暗黙的に検証 |
| NFR-UX-003 | 操作結果の即時フィードバック | `architecture.md#8.5` | `test_cases/cross-cutting.md#CRS-055〜075` | `test_cases/cross-cutting.md#CRS-055〜075` | E2E テストで操作後のフィードバック表示を検証 |
| NFR-UX-004 | クライアントサイドリアルタイムバリデーション | `architecture.md#8.5` | - | `test_cases/auth.md#AUTH-FE-016〜018, AUTH-FE-032〜037, AUTH-FE-050〜051, AUTH-FE-066〜068`<br>`test_cases/reports.md#RPT-FE-029〜033`<br>`test_cases/items.md#ITM-FE-028〜036` | |
| NFR-UX-005 | API 通信中のローディング表示 | `architecture.md#8.5` | - | `test_cases/dashboard.md#DSH-FE-006`<br>`test_cases/reports.md#RPT-FE-006, RPT-FE-034, RPT-FE-057, RPT-FE-065`<br>`test_cases/workflow.md#WFL-FE-006, WFL-FE-020, WFL-FE-035, WFL-FE-049, WFL-FE-066〜068`<br>`test_cases/attachments.md#ATT-FE-017, ATT-FE-028` | |

---

## 4. 禁止遷移トレーサビリティ（X1〜X10）

| 遷移ID | 禁止遷移 | 設計反映先 | BE テスト反映先 | FE テスト反映先 | 備考 |
|--------|---------|-----------|---------------|---------------|------|
| X1 | draft → approved（承認スキップ禁止） | `20_domain/state_machine.md#X1` | `test_cases/reports.md#RPT-061, RPT-073` | - | |
| X2 | draft → rejected（未提出却下禁止） | `20_domain/state_machine.md#X2` | `test_cases/reports.md#RPT-062, RPT-074` | - | |
| X3 | draft → paid（承認スキップ禁止） | `20_domain/state_machine.md#X3` | `test_cases/reports.md#RPT-063, RPT-075` | - | |
| X4 | submitted → draft（提出取消 MVP 対象外） | `20_domain/state_machine.md#X4` | `test_cases/reports.md#RPT-055` | - | |
| X5 | submitted → paid（承認スキップ禁止） | `20_domain/state_machine.md#X5` | `test_cases/reports.md#RPT-077`, `test_cases/workflow.md#WFL-052` | `test_cases/workflow.md#WFL-FE-061` | |
| X6 | approved → draft（承認済みを下書きに戻せない） | `20_domain/state_machine.md#X6` | `test_cases/reports.md#RPT-078` | - | |
| X7 | approved → submitted（逆遷移禁止） | `20_domain/state_machine.md#X7` | `test_cases/reports.md#RPT-079` | - | |
| X8 | approved → rejected（承認後の却下不可） | `20_domain/state_machine.md#X8` | `test_cases/reports.md#RPT-080`, `test_cases/workflow.md#WFL-031` | `test_cases/workflow.md#WFL-FE-060` | |
| X9 | rejected → 任意（終端状態からの遷移不可） | `20_domain/state_machine.md#X9` | `test_cases/reports.md#RPT-081`, `test_cases/workflow.md#WFL-015, WFL-033, WFL-055` | `test_cases/reports.md#RPT-FE-086, RPT-FE-090〜091` | |
| X10 | paid → 任意（終端状態からの遷移不可） | `20_domain/state_machine.md#X10` | `test_cases/reports.md#RPT-082`, `test_cases/workflow.md#WFL-016, WFL-034, WFL-054` | `test_cases/reports.md#RPT-FE-086`<br>`test_cases/workflow.md#WFL-FE-062` | |

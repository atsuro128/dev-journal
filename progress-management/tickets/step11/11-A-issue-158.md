# 11-A-issue-158: Approver の処理済みレポート一覧画面（SCR-WFL-003）追加

- 担当: 指揮役配下の各実装エージェント（platform-builder → backend-developer + frontend-developer 並列 → test-implementer）
- 依存: なし（#157 とは独立して並列実行可能）
- ブランチ: `step11/158-approver-processed-history`（expense-saas）
- 出力先:
  - 設計書: `dev-journal/deliverables/docs/40_basic_design/screens.md`, `50_detail_design/authz.md`, `50_detail_design/openapi.yaml`, `50_detail_design/screens/workflow-processed.md`（新規）
  - 実装: `expense-saas/` BE / FE 配下
  - SMK: `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`（新章 §4.12「処理済みレポート一覧」として SMK-103〜105 を §4.11 の直後に追加）
- 元資料: `dev-journal/issues/open/158-approver-past-approval-list-ui-missing.md`
- テンプレート: `ai-dev-framework/templates/ticket-template.md`

## 入力

| 資料 | パス | 参照箇所 | 用途 |
|------|------|----------|------|
| issue 本体 | `dev-journal/issues/open/158-approver-past-approval-list-ui-missing.md` | 全文（特に「設計書の規定」「現状」「完了条件」） | 課題の出発点と DoD のマッピング |
| 画面一覧 | `dev-journal/deliverables/docs/40_basic_design/screens.md` | §3.4 ワークフロー系（SCR-WFL-001 承認待ち / SCR-WFL-002 支払待ち）、§3.6 サマリ、§4.3 サイドナビ、§4.7 空状態、§4.9 ページネーション、§5 画面と API 対応 | 既存 SCR-WFL の構造把握。空き番号 SCR-WFL-003 を新採番として使用（既存 SCR-WFL-001/002 への参照は一切変更しない） |
| 認可設計 | `dev-journal/deliverables/docs/50_detail_design/authz.md` | §10.1 ロール別閲覧範囲、§10.2 一覧 vs 詳細の可視範囲、§10.3 Visibility Scope の組み立て、§10.4 判定表 | 一覧 API 用の visibility scope を明文化する追記対象 |
| OpenAPI | `dev-journal/deliverables/docs/50_detail_design/openapi.yaml` | L1421-1471（`/api/workflow/pending` 定義）、`PendingReport` schema、`Pagination` schema | `/api/workflow/processed` 追加の雛形と `ProcessedReport` schema 設計 |
| 既存画面仕様 | `dev-journal/deliverables/docs/50_detail_design/screens/workflow-pending.md` | 全体構造（§1〜§17） | `workflow-processed.md` を構造踏襲で作成するベース |
| 参考画面仕様 | `dev-journal/deliverables/docs/50_detail_design/screens/admin-all-reports.md` | §5 表示項目（処理結果・ステータス表示）、フィルタ仕様 | カラム表現・ステータス色分けの参考 |
| 既存 BE | `expense-saas/internal/handler/workflow.go`（`ListPendingReports`）、`internal/service/workflow_service.go`（`ListPendingReports`、`PendingReport` DTO）、`internal/repository/postgres/sqlcgen/expense_reports.sql.go` | 全体 | 構造踏襲（ハンドラ→サービス→リポジトリ→sqlc）の雛形 |
| 既存 FE | `expense-saas/frontend/src/pages/workflow/ApprovalListPage.tsx`、`frontend/src/components/layout/AppLayout.tsx`、`frontend/src/hooks/useReports.ts`（`usePendingReports`） | 全体 | Page・hooks・サイドメニューの実装パターン踏襲 |
| SMK | `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` | §4.11「画面内ナビゲーションリンク（SMK-099〜100）」の末尾位置、§5 チェック数サマリ | 新章 §4.12「処理済みレポート一覧（SMK-103〜105）」を §4.11 の直後に追加する |

## 責務

### 含めること

1. **設計書改訂**
   - `screens.md`：**SCR-WFL-003（処理済みレポート一覧）** を空き番号として新採番追加。既存 SCR-WFL-001（承認待ち）/ SCR-WFL-002（支払待ち）への参照は **一切変更しない**。§3.4 ワークフロー系の表に行追加、§3.6 サマリ、§4.3 サイドナビ（Approver の「処理済み」を「承認待ち」直下）、§4.7 空状態に追記、§5 画面と API 対応に `GET /api/workflow/processed` 追加、§4.9 ページネーション対象画面に SCR-WFL-003 追加。サイドメニューの並びは「画面ID 順 ≠ 表示順」とし業務文脈で並べる。
   - `authz.md` §10.2 Approver の表に `GET /api/workflow/processed` 行を追加（可視範囲: 自分が `approved_by` または `rejected_by` に記録されているテナント内レポート全件）。§10.3 一覧取得の擬似コードに「処理済み一覧 → `{ tenant_id: actor.tenant_id, approved_by: actor.user_id OR rejected_by: actor.user_id }`」を追記。
   - `openapi.yaml` に `GET /api/workflow/processed` を追加（`/api/workflow/pending` の隣）。`x-authz.roles: [approver]`、新 schema `ProcessedReport`（`PendingReport` + `decision`（approved/rejected）+ `decided_at` + `current_status`）を定義。
   - `screens/workflow-processed.md` を新規作成（`workflow-pending.md` の構造を踏襲）。
2. **BE 実装**：`/api/workflow/processed` ハンドラ・サービス・sqlc クエリ・認可テスト。
3. **FE 実装**：新 Page `ProcessedReportsPage.tsx`、ルーティング `/approvals/processed`、`AppLayout.tsx` の Approver サイドメニューに「処理済み」追加、`usePendingReports` を雛形にした `useProcessedReports` hook 追加、Vitest 単体テスト。
4. **SMK 追加**：新章 §4.12「処理済みレポート一覧（SMK-103〜105）」を `smoke_check.md` §4.11 の直後に新設追加。§5 チェック数サマリも更新する。
5. **DoD マッピング**：issue 完了条件 4 項目 + 「他 Approver の処理分が表示されない」の確認を完了条件に組み込む。

### 含めないこと

- Admin / Accounting への画面開放（本対応は Approver 限定。将来検討は本チケット外）。
- 既存 `/reports/all`（SCR-ADM-001）の改修や Approver 開放（issue 案1 は不採用）。
- 既存 `/api/workflow/pending` の挙動変更（一切変更しない）。
- `/api/reports/{id}` 詳細認可ロジックの変更（既に `approved_by` / `rejected_by` で許可済み、issue 本文も「API は仕様通り」と整理済み）。
- 既存 SCR-WFL-001（承認待ち）/ SCR-WFL-002（支払待ち）の番号変更・機能変更・ファイル変更（一切変更しない。空き番号 SCR-WFL-003 を新採番するため既存参照のリナンバリングは発生しない）。
- ダッシュボードへの「最近承認したレポート」セクション追加（issue 案3 は不採用）。

## 完了条件

### issue #158 完了条件マッピング

| issue 完了条件 | 本チケットでの実現 |
|---|---|
| Approver で過去の承認/却下レポートが UI 一覧表示される | `/approvals/processed` を新設、サイドメニューに「処理済み」追加 |
| API 側で正しい visibility scope が適用 | `/api/workflow/processed` に `tenant_id = actor.tenant_id AND (approved_by = actor.user_id OR rejected_by = actor.user_id)` を固定適用、テナント分離テスト追加 |
| 設計書（authz.md / screens.md）に scope 規定 | screens.md §3.4 に SCR-WFL-003（新採番）追加、authz.md §10.2 / §10.3 に処理済み一覧用 scope 追記 |
| 既存テスト継続 PASS + 新規テスト追加 | BE: ハンドラ単体・統合（テナント分離・RBAC・scope）。FE: Vitest 単体（レンダリング・空状態・並び替え・他ロール 403） |

### 追加 DoD

- 「他 Approver が処理したレポートが自分の `/approvals/processed` に表示されないこと」を BE 統合テストおよび SMK-105 で確認する。
- `screens.md` のサイドメニュー §4.3 で Approver の「処理済み」が「承認待ち」の直下に配置される。
- 既存 SCR-WFL-001 / SCR-WFL-002 を参照する設計書（screens.md, common-components.md, state-management.md, ui_flow.md, workflow-pending.md, workflow-payable.md, dashboard.md, report-detail.md, test_cases/workflow.md 等）/ 実装コードのコメント / テストコードに含まれる **既存の SCR-WFL-001 / SCR-WFL-002 への参照行は一切変更しない**（grep で全出現を確認したうえで本方針を確定済み、約 50 箇所の既存参照を温存）。`screens.md`, `common-components.md`, `state-management.md` には新規行（SCR-WFL-003 関連）の **追記のみ** 行う。
- `openapi.yaml` の lint（既存 CI）が PASS する。

---

## 任意セクション: 実装チケット

### 影響範囲

#### ドキュメント

| ファイル | 変更内容 |
|---|---|
| `dev-journal/deliverables/docs/40_basic_design/screens.md` | §3.4 に SCR-WFL-003（処理済み）行追加（既存 SCR-WFL-001/002 行は無変更）、§3.6 サマリ更新、§4.3 サイドナビに Approver「処理済み」追加、§4.7 空状態に「処理済みレポートはありません。」追加、§5 画面と API 対応に `GET /api/workflow/processed` 追加、§4.9 ページネーション対象画面に SCR-WFL-003（処理済み）追加 |
| `dev-journal/deliverables/docs/50_detail_design/authz.md` | §10.2 Approver の API 表に `GET /api/workflow/processed` 行追加。§10.3 擬似コードに処理済み一覧 scope を追記 |
| `dev-journal/deliverables/docs/50_detail_design/openapi.yaml` | `paths` に `/api/workflow/processed` 追加、`components.schemas.ProcessedReport` を新規定義（`PendingReport` のフィールドに `decision`, `decided_at`, `current_status` を追加） |
| `dev-journal/deliverables/docs/50_detail_design/screens/workflow-processed.md` | 新規作成（`workflow-pending.md` 構造踏襲） |
| `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` | 新章 §4.12「処理済みレポート一覧（SMK-103〜105）」を §4.11 の直後に追加（§4.9 はキャッシュ整合性、§4.11 末尾は SMK-100 のため）。§5 チェック数サマリも更新 |
| `dev-journal/deliverables/docs/55_ui_component/common-components.md` | 各共通コンポーネント（AppDataGrid / AppPagination / PageSizeSelector / AppPaginationFooter / EmptyState / PageSkeleton 等）の「使用箇所」リストに SCR-WFL-003 を**追記のみ**（既存 SCR-WFL-001/002 を含む既存記述は一切変更しない） |
| `dev-journal/deliverables/docs/55_ui_component/state-management.md` | Hook 対応表（L125-126 周辺）に `useProcessedReports` / `GET /api/workflow/processed` / `SCR-WFL-003` を**追記のみ**（既存 `usePendingReports` / `usePayableReports` 行は一切変更しない） |

#### BE

| ファイル | 変更内容 |
|---|---|
| `expense-saas/internal/handler/workflow.go` | `ListProcessedReports` ハンドラ追加（`ListPendingReports` を雛形）、ルーター登録（`/api/workflow/processed`、Approver RBAC ミドルウェア適用） |
| `expense-saas/internal/handler/workflow_handler_test.go` | `ListProcessedReports` の単体テスト追加（200 / 401 / 403） |
| `expense-saas/internal/service/workflow_service.go` | `ListProcessedReports` サービスメソッド追加、`ProcessedReport` DTO 追加（`decision`, `decided_at`, `current_status` を含む） |
| `expense-saas/internal/service/workflow_service_test.go`（既存があれば） | サービス層テスト追加 |
| `expense-saas/internal/repository/postgres/sqlcgen/queries/workflow.sql` 等（sqlc クエリ定義） | `ListProcessedReports` クエリ追加: `WHERE er.tenant_id = $1 AND (er.approved_by = $2 OR er.rejected_by = $2) AND er.deleted_at IS NULL ORDER BY COALESCE(er.approved_at, er.rejected_at) DESC, er.report_id DESC LIMIT $3 OFFSET $4`、および COUNT クエリ |
| `expense-saas/internal/repository/postgres/sqlcgen/*.sql.go` | sqlc 再生成 |
| 統合テスト | テナント分離・RBAC・他 Approver 処理分非表示の確認 |

#### FE

| ファイル | 変更内容 |
|---|---|
| `expense-saas/frontend/src/pages/workflow/ProcessedReportsPage.tsx`（新規） | `ApprovalListPage.tsx` を雛形に作成 |
| `expense-saas/frontend/src/pages/workflow/__tests__/ProcessedReportsPage.test.tsx`（新規） | レンダリング・空状態・処理結果表示・並び替え・403 リダイレクト |
| `expense-saas/frontend/src/hooks/useReports.ts` | `useProcessedReports` hook 追加（`usePendingReports` 雛形） |
| `expense-saas/frontend/src/api/`（クライアント生成箇所） | `listProcessedReports` クライアント関数追加 |
| `expense-saas/frontend/src/App.tsx` または ルーティング定義 | `/approvals/processed` ルート追加（Approver ロールガード） |
| `expense-saas/frontend/src/components/layout/AppLayout.tsx` | Approver サイドメニューに「処理済み」項目追加（「承認待ち」直下） |

### 実装方針

#### 設計判断（確定済み、実装の前提）

1. **新エンドポイント**: `GET /api/workflow/processed`（`/api/workflow/pending` は無変更）
2. **認可スコープ**: `tenant_id = ? AND (approved_by = actor.user_id OR rejected_by = actor.user_id)`、ロール: Approver のみ
3. **新規画面 URL**: `/approvals/processed`、画面ID: **SCR-WFL-003（処理済みレポート一覧）**（空き番号活用、既存 SCR-WFL-001/002 への参照は変更しない）
4. **表示カラム**: タイトル / 申請者 / 金額 / 処理結果（承認/却下） / 処理日（approved_at or rejected_at） / 現在ステータス（approved/paid/rejected）
5. **ソート**: 処理日時 DESC（同値時は report_id DESC で安定化）
6. **サイドメニュー**: Approver 用「処理済み」を「承認待ち」の直下に追加

#### `ProcessedReport` schema（OpenAPI）

```yaml
ProcessedReport:
  type: object
  required: [id, title, total_amount, submitter, decision, decided_at, current_status]
  properties:
    id: { type: string, format: uuid }
    title: { type: string }
    total_amount: { type: integer }
    submitter:
      type: object
      properties:
        id: { type: string, format: uuid }
        name: { type: string }
    decision:
      type: string
      enum: [approved, rejected]
      description: 自分の処理結果。approved_by が actor の場合 approved、rejected_by が actor の場合 rejected
    decided_at:
      type: string
      format: date-time
      description: 自分が処理した日時（approved_at または rejected_at）。decision に対応する側を返す
    current_status:
      type: string
      enum: [approved, rejected, paid]
      description: 現在のレポートステータス。承認後に paid に進んだか rejected のままかを示す
```

#### `workflow-processed.md`（新規画面仕様）の主要点

- 基本情報: 画面ID `SCR-WFL-003`、URL `/approvals/processed`、対応 API `GET /api/workflow/processed`、対応ロール `Approver`、不可ロールはダッシュボードへリダイレクト。
- アクションの責務分担: 本画面は **一覧表示と詳細画面（SCR-RPT-004）への遷移のみ**。再承認・取消等の操作は提供しない（MVP スコープ）。
- レイアウト: `workflow-pending.md` と同等。`AppDataGrid` + `AppPaginationFooter`。
- 表示カラム: ①申請者名 ②タイトル（リンク） ③合計金額 ④処理結果（承認/却下バッジ。承認=緑、却下=赤、§4.8 ステータスバッジ準拠の色） ⑤処理日（YYYY/MM/DD、デフォルト降順） ⑥現在ステータス（approved/paid/rejected をバッジ表示） ⑦遷移アイコン。
- フィルタ: MVP では実装しない（issue・指示書ともにフィルタ要件なし）。空状態のみ実装。
- ページネーション: `screens.md` §4.9 準拠（オフセット、デフォルト 20、選択肢 [10,20,50,100]）。
- 空状態: 「処理済みのレポートはありません。」（`screens.md` §4.7 に追記）。
- エラー表示: 401 → ログイン、403 → ダッシュボードリダイレクト、500 → トースト（`workflow-pending.md` と同一）。
- 処理シーケンス: ハンドラ→サービス→リポジトリ→DB のシーケンス図（`workflow-pending.md` の流用、WHERE 句のみ差し替え）。
- API リクエスト/レスポンス: `ProcessedReport` schema に従う。`is_own_report` は不要（自分の処理分のみ返るため）。
- テナント分離: `tenant_id` フィルタ + RLS の二重保証。
- 自己承認禁止との整合: 既に `/api/workflow/{id}/approve|reject` 側で禁止済みのため、`processed` 一覧には「自分が approved_by/rejected_by」となるレポートしか入らない（実装上問題なし）。

#### サービス層 DTO 構築方針

`approved_by = actor.user_id` の行は `decision = "approved"`, `decided_at = approved_at`、`rejected_by = actor.user_id` の行は `decision = "rejected"`, `decided_at = rejected_at`。両方が actor のケースは仕様上発生しない（rejected 後の再申請は別レポート）。`current_status` はレコードの `status` をそのまま返す。

### 作業手順

1. **設計書改訂（platform-builder 担当）**
   1. `screens.md`: §3.4 の表に SCR-WFL-003（処理済みレポート一覧）行を新規追加（既存 SCR-WFL-001/002 行は無変更）、§3.6 サマリ、§4.3 サイドナビ、§4.7 空状態、§5 画面と API 対応、§4.9 対象画面の整合更新。
   2. `authz.md` §10.2 / §10.3 に `/api/workflow/processed` の scope 追記。**§10.1 ロール別閲覧範囲表は改訂しない**（Approver 行に既に「自分が承認/却下したレポート」が記載されており、`/api/workflow/processed` の可視範囲はこの既存表記が内包する。設計の整合は既に取れている）。
   3. `openapi.yaml` に `/api/workflow/processed` パスと `ProcessedReport` schema 追加、yaml lint 確認。
   4. `screens/workflow-processed.md` 新規作成（本チケットの「主要点」を満たす内容）。
   5. 新章 §4.12「処理済みレポート一覧（SMK-103〜105）」を `smoke_check.md` §4.11 の直後に追加。§5 チェック数サマリの値も更新する。
   6. `common-components.md` の各共通コンポーネント「使用箇所」リストに SCR-WFL-003 を**追記のみ**（既存記述変更禁止）。
   7. `state-management.md` の Hook 対応表に `useProcessedReports` 行を**追記のみ**（既存記述変更禁止）。
   8. **以下のファイルへの変更は禁止**: `screens/workflow-payable.md`, `screens/workflow-pending.md`, `ui_flow.md`, `dashboard.md`, `report-detail.md`, `test_cases/workflow.md`, `test_cases/reports.md`, 実装コードコメント（SCR-WFL-001/002 関連の既存記述は現状維持）。
2. **BE 実装（backend-developer 担当、設計書改訂後着手）**
   1. sqlc クエリ追加 → `go generate ./...` で sqlcgen 再生成。
   2. リポジトリ層・サービス層・ハンドラ層・ルーター登録。
   3. 単体テスト・統合テスト追加（下記「テスト方針」参照）。
   4. `go vet ./...` / `go build ./...` PASS。
3. **FE 実装（frontend-developer 担当、BE と並列可）**
   1. API クライアント関数追加（OpenAPI ベースのクライアント生成があれば再生成）。
   2. `useProcessedReports` hook 追加。
   3. `ProcessedReportsPage.tsx` 実装（`ApprovalListPage.tsx` を雛形）。
   4. ルーティング・サイドメニュー追加。
   5. Vitest テスト追加。`npm run build` PASS。
4. **テスト追加（test-implementer、BE / FE 完了後）**
   1. BE: 認可テスト（テナント分離・RBAC・他 Approver 処理分非表示・自分が承認のみ／却下のみ／両方ありのケース）。
   2. FE: ProcessedReportsPage のレンダリング・空状態・処理結果表示・他ロール 403 リダイレクト。
5. **コミット計画**
   - C1: 設計書改訂（screens.md / authz.md / openapi.yaml / workflow-processed.md / smoke_check.md）
   - C2: BE 実装（sqlc + repo + service + handler + router）
   - C3: BE テスト追加
   - C4: FE 実装（hook + page + route + sidemenu）
   - C5: FE テスト追加

### テスト方針

#### BE

| 種別 | ケース | 期待動作 |
|---|---|---|
| ハンドラ単体 | Approver 200 OK | `data` と `pagination` が返る |
| ハンドラ単体 | Member / Admin / Accounting 403 | RBAC ミドルウェアで弾かれる |
| ハンドラ単体 | 未認証 401 | Auth ミドルウェアで弾かれる |
| 統合（テナント分離） | テナント A の Approver が テナント B のレポートを取得しない | 0 件で返る |
| 統合（scope） | 自分が approved_by のレポートのみ含まれる | `decision=approved`, `decided_at=approved_at` |
| 統合（scope） | 自分が rejected_by のレポートのみ含まれる | `decision=rejected`, `decided_at=rejected_at` |
| 統合（他 Approver 非表示） | 同テナントの別 Approver が処理したレポートは含まれない | 0 件 |
| 統合（current_status） | approved → paid に進んだレポートが `current_status=paid` で返る | OK |
| 統合（ソート） | 処理日時 DESC で並ぶ | OK |
| 統合（ページネーション） | `per_page=10`, `page=2` で正しく分割 | OK |

#### FE

| 種別 | ケース | 期待動作 |
|---|---|---|
| Vitest | レンダリング | 6 カラム表示 |
| Vitest | 空状態 | 「処理済みのレポートはありません。」表示 |
| Vitest | 処理結果バッジ | approved=緑、rejected=赤 |
| Vitest | 現在ステータスバッジ | paid 等が正しく表示 |
| Vitest | Approver 以外のロール | ダッシュボードへリダイレクト |
| Vitest | 行クリック | `/reports/:id` に遷移 |
| Vitest | per_page セレクタ | 既存 `AppPaginationFooter` テスト同様の挙動 |

> ローカルでのフルスイート実行はしない（CI で確認）。

#### SMK 追加項目（`smoke_check.md` §4.11 の直後に新章 §4.12「処理済みレポート一覧」として追加）

| ID | 項目 | 画面 | 前提データ | 実行ロール | 出発点 | 手順 | 期待結果 |
|---|---|---|---|---|---|---|---|
SMK-103〜105 は新章 §4.12「処理済みレポート一覧」として `smoke_check.md` の §4.11 の直後に追加する。

| SMK-103 | 処理済み一覧の基本表示 | 処理済みレポート一覧（SCR-WFL-003） | 自分が承認したレポート 1 件・却下したレポート 1 件 | Approver | サイドメニュー「処理済み」 | 1. 「処理済み」を押下 | 2 件が処理日 DESC で表示。承認バッジ（緑）・却下バッジ（赤）、現在ステータスバッジ表示 |
| SMK-104 | テナント分離 | 処理済みレポート一覧 | テナント A の Approver、テナント B にも自分の user_id 風データなし | Approver（テナント A） | サイドメニュー「処理済み」 | 1. 一覧を開く | テナント A の自分処理分のみ表示、テナント B は混入しない |
| SMK-105 | 他 Approver 処理分の非表示 | 処理済みレポート一覧 | 同テナントの別 Approver が承認したレポートあり | Approver（自分は未処理） | サイドメニュー「処理済み」 | 1. 一覧を開く | 別 Approver の処理分は表示されない（自分が approved_by / rejected_by のものだけ） |

---

## 任意セクション: 後続への引き継ぎ

- Admin / Accounting への画面開放可否は **本対応外**。本チケット完了後、ユーザー判断 issue として別途起票候補（Accounting は支払業務文脈で「自分が支払処理した分」を見たい可能性、Admin は監査文脈で全 Approver の処理履歴を見たい可能性。要件再確認が必要）。
- 「自分が承認後に paid に進んだ件数」サマリのダッシュボード表示は Phase 3 検討。
- 採番方針: 空き番号 SCR-WFL-003 を新採番。既存 SCR-WFL-001/002 への参照（約 50 箇所、設計書 11 ファイル + テストケース + 実装コードコメント）は一切変更しない。grep で全出現を事前確認済み（指揮役による）。
- 業務文脈の連続性は **サイドメニューの表示順**（承認待ち → 処理済み → 支払待ち 等）で実現し、画面 ID 番号と業務順の一致は要求しない。
- `useProcessedReports` の配置先 `expense-saas/frontend/src/hooks/useReports.ts` は `usePendingReports` の同居場所として確認済み（指揮役）。実装着手時に念のため `usePendingReports` の export 位置を確認すること。

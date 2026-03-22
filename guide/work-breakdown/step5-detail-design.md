# Step 5: 詳細設計（API / DB / 認可 / 添付 / セキュリティ）

## 目的

Step 4 で確定した画面一覧を元に、機能別の画面詳細仕様・API・DB・認可・セキュリティ・監視を設計し、実装できるレベルまで仕様を確定する。

## 前提

Step 4 の完了が前提。画面一覧（screens.md）で機能数・画面数が確定していること。

## 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md`（Step 4） |
| 画面遷移図 | `deliverables/docs/40_basic_design/ui_flow.md`（Step 4） |
| ドメインモデル | `deliverables/docs/20_domain/domain_model.md` |
| 状態遷移 | `deliverables/docs/20_domain/state_machine.md` |
| 要件定義 | `deliverables/docs/10_requirements/requirements.md` |
| ユースケース | `deliverables/docs/10_requirements/usecases.md` |
| RBAC | `deliverables/docs/10_requirements/rbac.md` |
| ワークフロー | `deliverables/docs/10_requirements/workflow.md` |
| アーキテクチャ | `deliverables/docs/30_arch/architecture.md` |
| ADR | `deliverables/docs/30_arch/adr/` |

## 成果物

| 成果物 | パス |
|--------|------|
| 機能別画面詳細仕様 | `deliverables/docs/50_detail_design/screens/*.md` |
| OpenAPI 定義 | `deliverables/docs/50_detail_design/openapi.yaml` |
| DB スキーマ設計 | `deliverables/docs/50_detail_design/db_schema.md` |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` |
| UI デザインガイドライン | `deliverables/docs/50_detail_design/ui-guidelines.md` |

### 画面一覧と画面仕様の分離

画面一覧（screens.md）は Step 4 で作成済みの全体俯瞰。各画面の詳細仕様（入力項目・バリデーション・エラー表示等）は機能別ファイル（screens/*.md）に分離する。抽象度が異なるため同一ファイルに混在させない。

### 成果物粒度原則

1画面（1機能）= 1ファイル。下流工程（テスト設計・実装）が画面単位でパイプラインを回せる粒度にする。
画面一覧（screens.md）のカテゴリは俯瞰用のグルーピングであり、成果物の粒度とは異なる。

## 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step5-detail-design.md`
- 計画が確定してから成果物作成に入ること

## 完了条件

- API の入出力が決まっている（OpenAPI で機械可読）
- 主要テーブル / 関係・インデックス方針が決まっている
- 認可チェックの責務と実装場所が決まっている
- セキュリティ要件の実装方針が明記されている
- 各画面の主要操作に処理シーケンス図がある（ボタン押下→API→サービス→ドメイン→リポジトリ→DB の縦断フロー）
- 複雑なバリデーション・権限チェックにフローチャートがある（分岐条件とエラーパスの可視化）

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。Phase ごとに整理する。

### 全 Phase 共通

#### 1. 上流整合性
- 各成果物が参照している上流のルールID（WFL-xxx, RBC-xxx, TNT-xxx 等）の定義元が実際に存在するか
- Step 1 の要件定義・Step 2 のドメイン設計・Step 3 のアーキテクチャ設計と矛盾する記述がないか
- テナント分離（全業務テーブルに `tenant_id`、RLS ポリシー、テナント境界越え=404）が反映されているか

#### 2. 内部整合性（Step 5 成果物間）
- `openapi.yaml` のエンドポイント定義と `screens/*.md` の API 呼び出し先が一致しているか
- `db_schema.md` のテーブル定義と `openapi.yaml` のリクエスト/レスポンスフィールドが整合しているか
- `authz.md` の認可ルールと `openapi.yaml` の各エンドポイントのロール制限が一致しているか
- `security.md` のトークン設計と `db_schema.md` のトークンテーブル定義が一致しているか
- `files.md` の添付操作フローと `db_schema.md` の RLS ポリシー設計が整合しているか

#### 3. 用語準拠
- 全成果物の用語が `glossary.md` に準拠しているか

#### 4. スコープ準拠
- MVP スコープ外の機能が含まれていないか

### Phase 1: 画面詳細 + DB スキーマ + セキュリティ

#### 5. 画面詳細仕様
- 各画面仕様が `screens.md` の画面定義（画面ID・URL・対応UC）と整合しているか
- 入力項目・バリデーションルール・エラーメッセージが定義されているか
- ロール別の表示差異が定義されているか
- 対応するユースケースの全フロー（正常系・代替系・例外系）が画面仕様でカバーされているか
- 主要操作の処理シーケンス図（ボタン→API→内部処理→DB）が含まれているか
- 複雑な分岐（バリデーション・権限チェック）のフローチャートが含まれているか

#### 6. DB スキーマ
- 全業務テーブルに `tenant_id` が付与されているか（User は例外として明記されているか）
- RLS ポリシーが全対象テーブル・全操作（SELECT, INSERT, UPDATE, DELETE）に定義されているか
- 論理削除対象テーブルの全てに `deleted_at` があるか（子テーブルを含む）
- インデックスが `tenant_id` を先頭に配置しているか
- 一意制約が NULL を含むケースで正しく機能するか（部分ユニークインデックスの使用）
- 楽観的ロック（`updated_at`）の方針が記載されているか
- `state_machine.md` の状態値と CHECK 制約が一致しているか
- RLS バイパスが必要なケースが「認証系+マイグレーション」に限定されているか（業務 API で使わないか）

#### 7. セキュリティ
- JWT 実装詳細（RS256 鍵管理、クレーム構造、有効期限）が定義されているか
- レート制限の具体値と実装方式が定義されているか
- CORS ポリシーが環境別に定義されているか
- パスワードハッシュパラメータ（Argon2id）が定義されているか
- DB ロール分離（`expense_owner` vs `expense_app`）の用途境界が明確か

### Phase 2: OpenAPI

#### 8. OpenAPI
- 全 MVP 機能のエンドポイントが定義されているか
- 各エンドポイントに許可ロール・所有者条件・状態条件が明示されているか
- エラーレスポンス（400, 401, 403, 404, 409, 422）の使い分けが統一されているか
- 上流の RBAC 定義（`rbac.md`）と OpenAPI の許可ロールが一致しているか

### Phase 3: 認可設計

#### 9. 認可
- 認可チェックの責務（ミドルウェア/サービス層/リポジトリ層）が明確か
- 所有者チェック・ロールチェック・テナントチェックの実行順序が定義されているか

### Phase 4: 最終レビュー（横断）

#### 10. 添付ファイル
- 署名付きURL発行フローのセキュリティ（認可チェック、有効期限）が定義されているか
- S3 バケット構成（キーフォーマット、暗号化、パブリックアクセスブロック）が定義されているか
- MIME バリデーション・サイズ制限が `requirements.md` と一致しているか

#### 11. ui_flow.md との整合性
- `ui_flow.md` の遷移先が `screens/*.md` の全画面をカバーしているか
- `screens/*.md` 内で定義された画面遷移（ボタン押下後の遷移先等）が `ui_flow.md` と矛盾しないか
- Step 5 の詳細化で追加・変更された遷移パスが `ui_flow.md` に反映されているか

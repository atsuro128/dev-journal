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
| 機能別画面詳細仕様 | `deliverables/docs/40_basic_design/screens/*.md` |
| OpenAPI 定義 | `deliverables/docs/50_detail_design/openapi.yaml` |
| DB スキーマ設計 | `deliverables/docs/50_detail_design/db_schema.md` |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` |
| 画面遷移図（最終版） | `deliverables/docs/40_basic_design/ui_flow.md` |

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
- 画面遷移図が全機能の詳細化結果を反映し最終版になっている

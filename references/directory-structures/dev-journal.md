# dev-journal ディレクトリ構成

開発プロセスの記録。進捗管理・日報・セッションログ・設計成果物・参照資料を管理する。

```
dev-journal/
├── ai-operations/                    # AI運用設計
│   ├── overview.md                   # AI運用方針（全体）
│   ├── hooks-design.md               # Hooks設計資料
│   ├── subagent-design.md            # サブエージェント設計
│   └── subagent-workflow.md          # サブエージェントワークフロー
├── daily-reports/                    # 作業日報（日付別）
│   └── YYYY-MM-DD.md
├── deliverables/                     # 設計成果物
│   ├── docs-structure.md             # 成果物構成定義
│   └── docs/
│       ├── 00_goals.md
│       ├── 01_glossary.md
│       ├── 02_scope.md
│       ├── 10_requirements/          # 要件定義
│       │   ├── requirements.md
│       │   ├── usecases.md
│       │   ├── workflow.md
│       │   ├── rbac.md
│       │   └── preliminary/          # 事前調査
│       │       ├── 01_business-overview.md
│       │       ├── 02_actor-analysis.md
│       │       ├── 03_business-flow.md
│       │       └── 04_business-rules.md
│       ├── 20_domain/               # ドメイン設計
│       │   ├── domain_model.md
│       │   └── state_machine.md
│       ├── 30_arch/                  # アーキテクチャ設計
│       │   ├── architecture.md
│       │   ├── diagrams.md
│       │   └── adr/                  # ADR（アーキテクチャ決定記録）
│       │       ├── 0001-tech-stack.md
│       │       ├── 0002-multi-tenant.md
│       │       ├── 0003-rls-tenant-isolation.md
│       │       ├── 0004-infra.md
│       │       └── 0005-monitoring-logging.md
│       ├── 40_basic_design/          # 基本設計
│       │   ├── screens.md
│       │   └── ui_flow.md
│       ├── 50_detail_design/         # 詳細設計
│       │   ├── authz.md
│       │   ├── db_schema.md
│       │   ├── files.md
│       │   ├── monitoring.md
│       │   ├── openapi.yaml
│       │   ├── security.md
│       │   ├── ui-guidelines.md
│       │   └── screens/              # 画面詳細仕様（12件）
│       │       ├── admin-all-reports.md
│       │       ├── admin-tenant.md
│       │       ├── auth-login.md
│       │       ├── auth-password-reset-request.md
│       │       ├── auth-password-reset.md
│       │       ├── auth-signup.md
│       │       ├── dashboard.md
│       │       ├── report-create.md
│       │       ├── report-detail.md
│       │       ├── report-edit.md
│       │       ├── report-list.md
│       │       ├── workflow-payable.md
│       │       └── workflow-pending.md
│       └── 60_test/                  # テスト設計
│           ├── test_strategy.md
│           └── test_cases/           # テストケース（8件）
│               ├── attachments.md
│               ├── auth.md
│               ├── cross-cutting.md
│               ├── dashboard.md
│               ├── items.md
│               ├── reports.md
│               ├── tenant.md
│               └── workflow.md
├── guide/                            # プロジェクト進行ガイド
│   ├── implementation-guide.md
│   ├── parallel-branch-operation.md  # 並列ブランチ運用ガイド
│   ├── review-input-index.md         # レビュー入力インデックス
│   ├── templates/
│   │   └── task-plan.md              # タスク計画テンプレート
│   └── work-breakdown/              # Step 別作業分解（成果物・完了条件・レビュー観点）
│       ├── step0-preparation.md
│       ├── step1-requirements.md
│       ├── step2-domain.md
│       ├── step3-architecture.md
│       ├── step4-basic-design.md
│       ├── step5-detail-design.md
│       ├── step6-testing.md
│       ├── step7-foundation.md
│       ├── step8-test-implementation.md
│       ├── step9-feature-implementation.md
│       └── step10-system-test.md
├── logs/                             # ログ
│   ├── session-log-archive.md        # セッションログアーカイブ
│   └── YYYY-MM-DD/                   # 過去セッションログ（アーカイブ）
│       └── session-log.md
├── progress-management/              # 進捗・課題管理
│   ├── progress.md
│   ├── session-log.md                # セッションログ（直近2セッション）
│   ├── issues/                       # 課題管理
│   │   ├── dashboard.md              # 課題ダッシュボード
│   │   ├── open/                     # 未解決（4件）
│   │   └── resolved/                 # 解決済み（多数）
│   └── task-plans/                   # タスク計画（Step別）
│       ├── step5-detail-design.md
│       ├── step6-testing.md
│       └── step7-foundation.md
├── references/                       # 参照資料
│   ├── decisions/                    # ADR（意思決定記録）
│   ├── dev-commands.md
│   ├── directory-structures/         # 各リポジトリのディレクトリ構成（本フォルダ）
│   ├── glossary.md
│   ├── links.md
│   └── tech-stack-notes/             # 技術スタック補足
│       └── explanation.md
└── review-findings/                  # レビュー記録
    ├── open/                         # 未対応の指摘（3件 + .gitkeep）
    ├── pending-review/               # 対応済み・再レビュー待ち
    └── resolved/                     # 解消済みの指摘（多数）
```

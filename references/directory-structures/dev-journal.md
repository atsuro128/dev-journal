# dev-journal ディレクトリ構成

開発プロセスの記録。進捗管理・日報・セッションログ・設計成果物・参照資料を管理する。

```
dev-journal/
├── PROJECT_SUMMARY.md               # プロジェクト概要・仕様サマリ
├── ai-operations/                    # AI運用設計
│   ├── overview.md                   # AI運用方針（全体）
│   └── hooks-design.md               # Hooks設計資料
├── daily-reports/                    # 作業日報（日付別）
│   └── YYYY-MM-DD.md
├── deliverables/                     # 設計成果物
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
│       └── 20_domain/               # ドメイン設計
│           ├── domain_model.md
│           └── state_machine.md
├── guide/                            # プロジェクト進行ガイド
│   ├── implementation-guide.md
│   └── portfolio_project_steps.md
├── logs/                             # セッションログ（日付別）
│   ├── YYYY-MM-DD/
│   │   └── session-log.md
│   └── hooks/
│       └── hook-warnings.log         # Hook警告ログ（自動記録）
├── progress-management/              # 進捗・課題管理
│   ├── progress.md
│   ├── issues/                       # 未解決の課題
│   ├── resolved/                     # クローズ済み課題
│   └── step-deliverables/            # ステップ別成果物チェック
├── references/                       # 参照資料
│   ├── decisions/                    # ADR（意思決定記録）
│   ├── dev-commands.md
│   ├── directory-structures/         # 各リポジトリのディレクトリ構成（本フォルダ）
│   ├── glossary.md
│   ├── links.md
│   └── tech-stack-notes/
└── review-findings/                  # レビュー記録
    ├── open/                         # 未対応の指摘
    ├── pending-review/               # 対応済み・再レビュー待ち
    └── resolved/                     # 解消済みの指摘
```

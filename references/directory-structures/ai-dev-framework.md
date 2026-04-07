# ai-dev-framework ディレクトリ構成

AI駆動開発フレームワーク。ガイド・テンプレート・運用設計・Codex手順書を管理する。

```
ai-dev-framework/
├── agents/                      # Codex エージェント手順書
│   ├── issue-review-procedure.md
│   ├── pr-review-procedure.md
│   └── review-procedure.md
├── guide/                       # プロジェクト進行ガイド
│   ├── workflow.md              # ワークフロー手順（指揮役専用）
│   └── work-breakdown/          # Step 別作業分解
│       ├── pre-step-discovery.md
│       ├── step0-preparation.md
│       ├── step1-requirements.md
│       ├── step2-domain.md
│       ├── step3-architecture.md
│       ├── step4-basic-design.md
│       ├── step5-detail-design.md
│       ├── step6-testing.md
│       ├── step7-operations.md
│       ├── step8-foundation.md
│       ├── step9-test-implementation.md
│       ├── step10-feature-implementation.md
│       └── step11-system-test.md
├── operations/                  # AI運用設計（人間向け参照資料）
│   ├── overview.md              # AI運用方針
│   ├── branch-strategy.md       # ブランチ戦略
│   ├── hooks-design.md          # Hooks設計資料
│   ├── subagent-design.md       # サブエージェント設計
│   └── subagent-workflow.md     # サブエージェントワークフロー
└── templates/                   # ドキュメントテンプレート
    ├── ADR-template.md
    ├── README-template.md
    ├── docs-v2/                 # 成果物テンプレート集
    ├── issue-template.md
    ├── review-finding-template.md
    ├── task-template.md
    ├── ticket-template.md
    └── work-breakdown-template.md
```

## 備考

- 旧 `rules/` ディレクトリは `root-project/.claude/rules/` に移行済み（ADR-002）
- 旧 `prompts/` ディレクトリは廃止済み
- 旧 `scripts/` ディレクトリは廃止済み
- `operations/` は人間向けの設計参照資料（AI の自動読み込み対象外）

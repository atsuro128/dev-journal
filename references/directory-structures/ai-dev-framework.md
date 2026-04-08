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
│   └── work-breakdown/          # Step 別作業分解（各 Step = ディレクトリ）
│       ├── pre-step-discovery/
│       │   ├── main.md          # タスク・依存関係・プロセス
│       │   └── review.md        # 上流成果物（入力）・完了条件・品質ゲート・レビュー観点
│       ├── step0-preparation/
│       │   ├── main.md
│       │   └── review.md
│       ├── step1-requirements/   ... (同構造)
│       ├── step2-domain/
│       ├── step3-architecture/
│       ├── step4-basic-design/
│       ├── step5-detail-design/
│       ├── step5.5-ui-component/
│       ├── step6-testing/
│       ├── step7-operations/
│       ├── step8-foundation/
│       ├── step9-test-implementation/
│       ├── step10-feature-implementation/
│       └── step11-system-test/
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

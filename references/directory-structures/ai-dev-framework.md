# ai-dev-framework ディレクトリ構成

AI駆動開発フレームワーク。ルール体系・テンプレート・エージェント設計を管理する。

```
ai-dev-framework/
├── AGENTS.md                    # エージェント設計
├── agents/                      # エージェント手順書
│   ├── re-review-procedure.md
│   └── review-procedure.md
├── prompts/                     # 指示文テンプレート
│   └── README.md
├── rules/                       # 規約・ポリシー
│   ├── architecture.md
│   ├── branching.md
│   ├── coding-standards.md
│   ├── commit-message.md
│   ├── data-handling.md
│   ├── issue-management.md
│   ├── review-checklist.md
│   ├── review-findings.md
│   ├── security-policy.md
│   ├── session-log.md
│   └── testing.md
├── scripts/                     # 自動化スクリプト
│   ├── db-reset.sh
│   ├── lint.sh
│   └── setup.sh
└── templates/                   # ドキュメントテンプレート
    ├── ADR-template.md
    ├── issue-template.md
    ├── README-template.md
    └── RFC-template.md
```

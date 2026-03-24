# ai-dev-framework ディレクトリ構成

AI駆動開発フレームワーク。テンプレート・エージェント設計・スクリプトを管理する。

```
ai-dev-framework/
├── agents/                      # エージェント手順書
│   ├── issue-review-procedure.md
│   ├── re-review-procedure.md
│   └── review-procedure.md
├── scripts/                     # 自動化スクリプト
│   ├── db-reset.sh
│   ├── lint.sh
│   └── setup.sh
└── templates/                   # ドキュメントテンプレート
    ├── ADR-template.md
    ├── README-template.md
    ├── issue-template.md
    ├── review-finding-template.md
    ├── task-plan-template.md
    ├── task-template.md
    └── work-breakdown-template.md
```

## 備考

- 旧 `rules/` ディレクトリは `root-project/.claude/rules/` に移行済み（ADR-002）
- 旧 `prompts/` ディレクトリは廃止済み

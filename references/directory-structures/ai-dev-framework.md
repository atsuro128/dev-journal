# ai-dev-framework ディレクトリ構成

AI駆動開発フレームワーク。テンプレート・エージェント設計・スクリプトを管理する。

```
ai-dev-framework/
├── agents/                      # エージェント手順書
│   ├── re-review-procedure.md
│   └── review-procedure.md
├── prompts/                     # 指示文テンプレート
│   └── README.md
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

## 備考

- 旧 `rules/` ディレクトリは `root-project/.claude/rules/` に移行済み（ADR-002）

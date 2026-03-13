# root-project ディレクトリ構成

メタリポジトリ。Claude Code の実行基盤（CLAUDE.md, .claude/）を配置し、サブリポジトリを統括する。

```
root-project/
├── CLAUDE.md                    # Claude Code プロジェクト方針
├── AGENTS.md                    # codex エージェント指示書
├── .gitignore
├── .devcontainer/               # Dev Container 環境
│   ├── devcontainer.json
│   ├── Dockerfile
│   └── init-firewall.sh         # Egress ファイアウォール（Squid プロキシ）
├── .claude/                     # Claude Code 設定
│   ├── commands/                # カスタムコマンド（空）
│   ├── hooks/                   # Hooks スクリプト
│   │   ├── edit-scope-check.py
│   │   └── stop-check.py
│   ├── rules/                   # ルール（自動ロード）
│   │   ├── architecture.md
│   │   ├── coding-standards.md
│   │   ├── incident-response.md
│   │   ├── security-policy.md
│   │   ├── testing.md
│   │   └── workflow.md
│   ├── skills/                  # スキル定義
│   │   ├── analyze/
│   │   ├── check-structure/
│   │   ├── codex-review/
│   │   ├── commit/
│   │   ├── daily-report/
│   │   ├── incident-review/
│   │   ├── issue/
│   │   ├── requirement/
│   │   ├── review/
│   │   ├── review-findings/
│   │   └── session-log/
│   └── settings.json            # 権限・hooks設定
├── ai-dev-framework/            # AI駆動開発フレームワーク（独立Git）
├── expense-saas/                # プロダクト本体（独立Git）
└── dev-journal/                 # 開発プロセス記録（独立Git）
```

## 備考

- サブリポジトリ（ai-dev-framework/, expense-saas/, dev-journal/）は独立した Git リポジトリとして管理
- 各リポジトリの詳細構成は本フォルダ内の個別ファイルを参照
- `rules/` は旧 `ai-dev-framework/rules/` から `.claude/rules/` に移行済み
- `skills/` の各ディレクトリには `SKILL.md` が格納されている

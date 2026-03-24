# root-project ディレクトリ構成

メタリポジトリ。Claude Code の実行基盤（CLAUDE.md, .claude/）を配置し、サブリポジトリを統括する。

```
root-project/
├── CLAUDE.md                    # Claude Code プロジェクト方針
├── AGENTS.md                    # codex エージェント指示書
├── .gitattributes
├── .gitignore
├── .devcontainer/               # Dev Container 環境
│   ├── devcontainer.json
│   ├── Dockerfile
│   ├── SECURITY.md              # セキュリティ方針
│   ├── announce-egress-status.sh # Egress 状態通知
│   ├── init-devcontainer.sh     # Dev Container 初期化スクリプト
│   ├── init-firewall.sh         # Egress ファイアウォール（Squid プロキシ）
│   ├── proxy-allowlist.txt      # プロキシ許可リスト
│   ├── proxy-allowlist-rationale.md # 許可リストの根拠
│   ├── render-squid-config.sh   # Squid 設定レンダリング
│   └── verify-egress.sh         # Egress 検証
├── .claude/                     # Claude Code 設定
│   ├── agents/                  # サブエージェント手順書（17件）
│   │   ├── backend-developer.md
│   │   ├── basic-designer.md
│   │   ├── db-designer.md
│   │   ├── design-architect.md
│   │   ├── design-cross-reviewer.md
│   │   ├── design-unit-reviewer.md
│   │   ├── detail-designer.md
│   │   ├── frontend-developer.md
│   │   ├── impl-architect.md
│   │   ├── impl-cross-reviewer.md
│   │   ├── impl-unit-reviewer.md
│   │   ├── ops-writer.md
│   │   ├── planner.md
│   │   ├── platform-builder.md
│   │   ├── test-designer.md
│   │   ├── test-implementer.md
│   │   └── test-reviewer.md
│   ├── hooks/                   # Hooks スクリプト
│   │   └── edit-scope-check.py
│   ├── memory/                  # 永続メモリ
│   │   └── .gitkeep
│   ├── rules/                   # ルール（自動ロード）
│   │   ├── architecture.md
│   │   ├── coding-standards.md
│   │   ├── security-policy.md
│   │   ├── testing.md
│   │   └── workflow.md
│   ├── skills/                  # スキル定義
│   │   ├── analyze/
│   │   ├── audit-context/
│   │   ├── check-structure/
│   │   ├── codex-review/
│   │   ├── commit/
│   │   ├── daily-report/
│   │   ├── issue/
│   │   ├── review/
│   │   ├── review-findings/
│   │   ├── self-review/
│   │   ├── session-log/
│   │   └── weekly-review/
│   └── settings.json            # 権限・hooks設定
├── ai-dev-framework/            # AI駆動開発フレームワーク（独立Git）
├── expense-saas/                # プロダクト本体（独立Git）
└── dev-journal/                 # 開発プロセス記録（独立Git）
```

## 備考

- サブリポジトリ（ai-dev-framework/, expense-saas/, dev-journal/）は独立した Git リポジトリとして管理
- 各リポジトリの詳細構成は本フォルダ内の個別ファイルを参照
- `rules/` は旧 `ai-dev-framework/rules/` から `.claude/rules/` に移行済み
- `skills/` の各ディレクトリには `SKILL.md`（一部 `skill.md`）が格納されている
- `agents/` はサブエージェント手順書を中央集約管理

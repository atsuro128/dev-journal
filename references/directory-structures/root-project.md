# root-project ディレクトリ構成

メタリポジトリ。Claude Code の実行基盤（CLAUDE.md, .claude/）を配置し、サブリポジトリを統括する。

```
root-project/
├── CLAUDE.md                    # Claude Code プロジェクト方針
├── .claude/                     # Claude Code 設定
│   ├── commands/                # カスタムコマンド
│   ├── hooks/                   # Hooks スクリプト
│   ├── memory/                  # 自動メモリ
│   └── settings.json            # 権限・hooks設定
├── ai-dev-framework/            # AI駆動開発フレームワーク（独立Git）
├── expense-saas/                # プロダクト本体（独立Git）
└── dev-journal/                 # 開発プロセス記録（独立Git）
```

## 備考

- サブリポジトリ（ai-dev-framework/, expense-saas/, dev-journal/）は独立した Git リポジトリとして管理
- 各リポジトリの詳細構成は本フォルダ内の個別ファイルを参照

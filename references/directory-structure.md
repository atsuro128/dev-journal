# root-project ディレクトリ構成

project/ 以下のプロダクト構成は `references/project-structure.md` を参照。

```
root-project/
├─ CLAUDE.md                         # Claude Code プロジェクト方針
├─ PROJECT_SUMMARY.md                # プロジェクト概要・仕様サマリ
├─ .claude/                          # Claude Code 設定
│  ├─ commands/                      # カスタムコマンド（check-structure, daily-report, requirement, review, scope, session-log, status）
│  └─ settings.json
├─ guide/                            # プロジェクト進行ガイド
│  ├─ github.md
│  ├─ implementation-guide.md
│  └─ portfolio_project_steps.md
├─ progress-management/              # 進捗・課題管理
│  ├─ progress.md
│  ├─ progress-review.md
│  ├─ issues/                        # 未解決の課題（フラット構造・カテゴリはファイル内で管理）
│  ├─ pending-review/                # 対応完了・レビュー待ちの課題
│  ├─ resolved/                      # レビュー完了・クローズ済み課題
│  └─ step-deliverables/             # ステップ別成果物チェック
├─ rules/                            # 規約・ポリシー
│  ├─ branching.md
│  ├─ coding-standards.md
│  ├─ commit-message.md
│  ├─ data-handling.md
│  ├─ github.md
│  ├─ architecture.md
│  ├─ issue-management.md
│  ├─ review-checklist.md
│  ├─ review-findings.md
│  ├─ security-policy.md
│  ├─ session-log.md
│  └─ testing.md
├─ prompts/                          # 指示文テンプレート（手動コピペ用 / .claude/commands/ 昇格前の下書き）
│  └─ README.md
├─ templates/                        # ドキュメントテンプレート
│  ├─ ADR-template.md
│  ├─ issue-template.md
│  ├─ README-template.md
│  └─ RFC-template.md
├─ ai-operations/                    # AI運用設計
│  ├─ overview.md                    # AI運用方針（全体）
│  └─ hooks-design.md               # Hooks設計資料
├─ references/                       # 参照資料
│  ├─ decisions/
│  ├─ dev-commands.md
│  ├─ directory-structure.md         # 本ファイル
│  ├─ glossary.md
│  ├─ links.md
│  ├─ project-structure.md           # project/ の構成
│  └─ tech-stack-notes/
├─ scripts/                          # 自動化スクリプト
│  ├─ db-reset.sh
│  ├─ lint.sh
│  └─ setup.sh
├─ deliverables/                     # 成果物
│  └─ docs/
│     ├─ 00_goals.md
│     ├─ 01_glossary.md
│     ├─ 02_scope.md
│     └─ 10_requirements/           # 要件定義
│        ├─ requirements.md
│        ├─ usecases.md
│        ├─ workflow.md
│        ├─ rbac.md
│        └─ preliminary/            # 事前調査
│           ├─ 01_business-overview.md
│           ├─ 02_actor-analysis.md
│           ├─ 03_business-flow.md
│           └─ 04_business-rules.md
├─ daily-reports/                     # 作業日報（日付別）
│  └─ YYYY-MM-DD.md                 # その日の作業日報
├─ logs/                             # セッションログ（日付別）
│  └─ YYYY-MM-DD/
│     └─ session-log.md             # その日の作業ログ
└─ project/                          # プロダクトコード → references/project-structure.md 参照
```

## ディレクトリ役割

| ディレクトリ | 役割 |
|---|---|
| `CLAUDE.md` / `rules/` / `prompts/` / `.claude/commands/` | AIエージェント設定・規約 |
| `ai-operations/` | AI運用設計（hooks設計、運用方針等） |
| `references/` | 仕様・内部資料 |
| `templates/` / `scripts/` / `deliverables/` | 作業補助・成果物 |
| `progress-management/` | 進捗・課題管理 |
| `daily-reports/` | 作業日報（日付別） |
| `logs/` | セッションログ（日付別） |
| `project/` | 実プロダクト（ソースコード）|


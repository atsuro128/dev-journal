# ロールベース・カスタムサブエージェント設計

## 概要

Step 4（基本設計）以降の開発フェーズにおいて、**役割（ロール）** に対応したカスタムサブエージェント（全15体）を `.claude/agents/` に配置する。
計画者・設計者・実装者・レビュー者の役割分担で、並列化・品質ゲート・専門性の分離を実現する。

---

## エージェント一覧

### 設計フェーズ（Step 4〜5）— 6 体

| # | Agent | Role | Model | 権限 | 配置ファイル |
|---|-------|------|-------|------|------------|
| 1 | `design-architect` | 計画者・統合者 | opus | Write（計画ファイル + 統合成果物） | `.claude/agents/design-architect.md` |
| 2 | `basic-designer` | 基本設計者 | opus | Write | `.claude/agents/basic-designer.md` |
| 3 | `detail-designer` | 詳細設計者 | opus | Write | `.claude/agents/detail-designer.md` |
| 4 | `db-designer` | DB設計者 | opus | Write | `.claude/agents/db-designer.md` |
| 5 | `design-unit-reviewer` | 単体レビュー者 | opus | Read-only | `.claude/agents/design-unit-reviewer.md` |
| 6 | `design-cross-reviewer` | 横断レビュー者 | opus | Read-only | `.claude/agents/design-cross-reviewer.md` |

### 実装フェーズ（Step 6+）— 9 体

| # | Agent | Role | Model | 権限 | 配置ファイル |
|---|-------|------|-------|------|------------|
| 7 | `impl-architect` | 計画者 | opus | Write（計画ファイル）+ Bash | `.claude/agents/impl-architect.md` |
| 8 | `test-designer` | テスト設計者 | sonnet | Write | `.claude/agents/test-designer.md` |
| 9 | `test-implementer` | テスト実装者 | sonnet | Write + Bash | `.claude/agents/test-implementer.md` |
| 10 | `test-reviewer` | テストレビュー者 | sonnet | Read-only + Bash | `.claude/agents/test-reviewer.md` |
| 11 | `platform-builder` | 基盤実装者（初回のみ） | sonnet | Write + Bash | `.claude/agents/platform-builder.md` |
| 12 | `frontend-developer` | フロント実装者 | sonnet | Write + Bash | `.claude/agents/frontend-developer.md` |
| 13 | `backend-developer` | バック実装者 | sonnet | Write + Bash | `.claude/agents/backend-developer.md` |
| 14 | `impl-unit-reviewer` | 単体レビュー者 | opus | Read-only + Bash | `.claude/agents/impl-unit-reviewer.md` |
| 15 | `impl-cross-reviewer` | 横断レビュー者 | opus | Read-only + Bash | `.claude/agents/impl-cross-reviewer.md` |

---

## 設計方針

### Model 選定基準

| Model | 用途 | 対象エージェント |
|-------|------|----------------|
| **opus** | 設計判断・複数ドキュメント横断の推論・品質ゲート | 設計フェーズ全体、architect、reviewer |
| **sonnet** | 明確な仕様に基づく実装作業 | developer、test-designer、test-implementer、platform-builder |

設計フェーズは上流成果物の解釈と判断が必要なため全エージェントを opus とした。
実装フェーズは設計ドキュメントという明確な仕様が入力となるため sonnet で十分。
ただし実装フェーズでもレビュー（unit/cross）は品質判断が必要なため opus とした。

### 権限設計

- **Read-only**: レビュー者（分析のみ、成果物は変更しない）
- **Write（計画ファイル + 統合成果物）**: architect（タスク計画ファイル作成 + 最終統合）
- **Write**: 設計者・実装者（成果物を作成・編集する）
- **Bash**: 実装フェーズのエージェント（ビルド確認・テスト実行・静的解析に使用）

---

## タスク実行計画

architect がタスク実行計画を永続ファイルとして作成する。指揮役はこのファイルを読んで状況を把握し、エージェントを起動する。

```
dev-journal/progress-management/task-plans/
├── step5.md     # design-architect が作成・更新
└── step6-7.md   # impl-architect が作成・更新
```

- Step 4 は成果物が少なく task-plans 不要（work-breakdown にプロセスを記載）
- セッション間の引き継ぎ: 新セッションの指揮役が計画ファイルを読むだけで復帰可能
- 並列指揮役: 複数ターミナルの指揮役が同じ計画ファイルを参照し、タスクの重複を防ぐ
- 詳細なプロトコルは `subagent-workflow.md` を参照

---

## 連携ワークフロー

詳細は `subagent-workflow.md` を参照。

### Step 4: 基本設計

```
Lead が basic-designer に直接委譲（architect 不要）
  basic-designer → screens.md + ui_flow.md
    ↓
  design-unit-reviewer: 上流整合性チェック
    ↓
  ユーザーにコミット確認
```

### Step 5: 詳細設計

```
design-architect: タスク分解・依存関係・受け入れ基準を策定
    ↓
architect の計画に従いタスクを実行:
  基盤タスク（DB・セキュリティ・監視）
  機能別タスク（画面詳細 → API 定義）
    ↓
design-unit-reviewer × 各機能: 設計整合性チェック
    ↓
design-architect: authz.md 作成 + ui_flow.md 最終更新 + 全成果物の整合性確認
design-cross-reviewer: 全機能横断 + 上流トレーサビリティ
```

### Step 6+: 実装フェーズ

```
impl-architect: 実装タスク分解・依存関係整理・受け入れ基準
    ↓
test-designer → test_strategy.md, test_cases.md
    ↓
platform-builder → 骨組み・共通基盤・Docker・CI（初回のみ）
    ↓
各機能ごと:
  backend-developer + frontend-developer（並列）
    → test-implementer
    ↓
impl-unit-reviewer × 各機能: フロント/バック横断 + 設計トレーサビリティ
test-reviewer: テスト品質・網羅性チェック
    ↓
impl-cross-reviewer: 全機能横断 + セキュリティ + 設計整合性
```

#### platform-builder の位置づけ

初回構築が主務。ディレクトリ構造・共通ミドルウェア・Docker・CI を構築したら完了。
以降は backend-developer / frontend-developer がその基盤の上に機能を積む。
共通基盤に変更が必要になった場合のみ再起動する。

---

## 既存スキルとの関係

| 既存スキル | エージェントとの関係 |
|---|---|
| `/review` | reviewer エージェントの導入により将来的に廃止を検討。reviewer エージェントがワークフロー内の品質ゲートを自動化するため、手動レビューの必要性が低下する |
| `/commit` | レビューエージェントの結果を確認してからコミットするフローに組み込み可能 |
| `/codex-review` | Step 成果物完成後の外部レビュー。エージェントレビューは作業中の品質ゲート |
| `/issue` | レビューエージェントが blocker を検出した場合、issue 起票につなげる |
| `/plan` | design-architect / impl-architect がプラン策定の補助として機能 |

---

## レビューパターン

### パターン 3: 単体レビュー（design-unit-reviewer / impl-unit-reviewer）

1 機能を対象に、基本設計・詳細設計・DB 設計（または フロント・バック・テスト）を横断チェック + 上流トレーサビリティ検証。
各機能について並列に実行可能（対象ファイルが異なるため干渉しない）。

### パターン 4: 横断レビュー（design-cross-reviewer / impl-cross-reviewer）

全機能を横断して機能間整合性 + 成果物全体の上流トレーサビリティを検証。
全成果物が揃った後に実行する（Step 最終タスク / 実装フェーズ最終）。

---

## 呼び出し方

Claude Code の `Agent` ツールで `subagent_type` パラメータに agent 名を指定:

```
Agent(subagent_type="basic-designer", prompt="画面一覧・遷移を設計して...")
Agent(subagent_type="design-unit-reviewer", prompt="基本設計の上流整合性をレビューして...")
```

並列実行例:

```
Agent(subagent_type="db-designer", prompt="DB スキーマを設計して...")
Agent(subagent_type="detail-designer", prompt="セキュリティ設計を作成して...")
```

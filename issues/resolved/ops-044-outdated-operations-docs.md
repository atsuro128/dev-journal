# operations/ 配下の文書がエージェント統合後に未更新

## 発見日
2026-03-29

## カテゴリ
documentation

## 影響度
低

## 発見経緯
ai-dev-framework 全体の棚卸しで発見

## 関連ステップ
全体

## 問題

ai-dev-framework/operations/ 配下の3ファイルが、カスタムエージェント統合（17体→8体）後に更新されていない。

- `overview.md` — CLAUDE.md から参照なし、エージェント数が古い（15体前提）
- `subagent-design.md` — 15体の設計カタログだが実態は8体
- `subagent-workflow.md` — 15体前提のオーケストレーション手順

## 影響

- 実害は現時点でなし（これらを参照する運用フローがない）
- 新規参加者が読んだ場合に混乱する可能性

## 提案

以下のいずれかで対応：
- A: 8体構成に合わせて改訂
- B: 役割が workflow.md とエージェント定義に移行済みのため削除

### 変更対象

- `ai-dev-framework/operations/overview.md`
- `ai-dev-framework/operations/subagent-design.md`
- `ai-dev-framework/operations/subagent-workflow.md`
- `AGENTS.md` — 参照ドキュメント欄に削除済みルールファイル（`architecture.md`, `coding-standards.md`, `testing.md`, `security-policy.md`）への参照が残っている
- `.claude/agents/backend-developer.md` — 制約セクションに同じく削除済みファイルへの参照

---

## 解決内容

operations/ 3ファイルは人間向け参照資料のため残置。それ以外の古い参照を全て修正した。

### 削除済み rules ファイルへの参照除去（5ファイル）
- `AGENTS.md` — `architecture.md`, `coding-standards.md`, `testing.md`, `security-policy.md` への参照4行を除去
- `.claude/agents/frontend-developer.md` — 必須参照・制約セクションから `coding-standards.md`, `security-policy.md` を除去
- `.claude/agents/backend-developer.md` — 必須参照・制約セクションから `architecture.md`, `coding-standards.md`, `security-policy.md` を除去
- `.claude/agents/test-implementer.md` — 必須参照・制約セクションから `testing.md`, `coding-standards.md` を除去
- `.claude/agents/platform-builder.md` — 必須参照・制約セクションから4件の rules 参照を除去

### 旧エージェント名の修正（1ファイル）
- `ai-dev-framework/guide/work-breakdown/step4-basic-design.md` — `basic-designer` → `designer`、`design-unit-reviewer` → `reviewer`

### ディレクトリ構成ドキュメント更新（3ファイル）
- `dev-journal/references/directory-structures/root-project.md` — agents/ 8体、rules/ 2ファイルに更新
- `dev-journal/references/directory-structures/ai-dev-framework.md` — guide/・operations/ を追加、scripts/ 廃止を反映
- `dev-journal/references/directory-structures/dev-journal.md` — operations/ 誤記除去、削除済みファイル（implementation-guide.md, review-input-index.md）除去

## 解決日
2026-03-30

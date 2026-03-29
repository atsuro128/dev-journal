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

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

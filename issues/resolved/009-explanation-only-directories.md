# explanation.md のみのディレクトリが複数ある

## 発見日
2026-03-04

## 関連ステップ
各ステップで段階的に対処

## カテゴリ
project-management

## 問題
以下のディレクトリには `explanation.md`（ディレクトリの存在理由を示すプレースホルダ）しか配置されておらず、実質的なコンテンツが未整備:

### root-project/ 配下
- `references/decisions/` — ADR（アーキテクチャ決定記録）が未作成
- `references/tech-stack-notes/` — 技術選定メモが未作成

### project/ 配下（実装開始前のため現時点では想定通り）
- `project/.github/workflows/`
- `project/apps/web/src/`
- `project/apps/api/src/`
- `project/packages/ui/`
- `project/packages/config/`
- `project/packages/types/`
- `project/infra/`
- `project/database/`
- `project/docker/`
- `project/scripts/`
- `project/tests/`

## 影響
- `references/decisions/` が空のままだと、技術選定の経緯が追えない
- `references/tech-stack-notes/` が空のままだと、スタック選定理由が不明なまま実装に入るリスク
- `project/` 配下は実装フェーズ（Step 7以降）で自然に埋まるため緊急度は低い

## 提案
- `references/decisions/` → Step 2〜3 の仕様確定時に ADR を作成（issue-003 と関連）
- `references/tech-stack-notes/` → Step 3 実施時に技術選定メモを作成
- `project/` 配下 → 該当ステップの実装時に対応（別途管理不要）

## 解決内容

3/8 のリポジトリ再構成（ADR-001）により issue の前提が変更され、指摘の大部分が解消:

- `references/decisions/` — ADR-001, ADR-002, ADR-004 等が作成済み。explanation.md のみの状態は解消
- `references/tech-stack-notes/` — Step 3 未着手のため explanation.md のみは想定通り。Step 3 で自然に埋まる
- `project/` 配下 — `expense-saas/` にリネーム済みで旧パス自体が存在しない。実装前フェーズのため想定通り

## 解決日
2026-03-16

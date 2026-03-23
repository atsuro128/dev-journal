# 開発ルール文書の整備

## 発見日
2026-03-04

## 関連ステップ
各ステップで段階的に対処

## カテゴリ
project-management

## 問題

開発ルール文書の一部が未整備。実装フェーズに入る前に、各ルール文書の概要を定義し、必要なタイミングで肉付けする必要がある。

## ルール文書一覧

ルール配置先: `.claude/rules/`

### 整備済み

| ファイル | 概要 | 整備時期 |
|---------|------|---------|
| `workflow.md` | プロジェクト運用ルール。セッション管理、成果物作成フロー、品質ゲート判定基準、意思決定権限 | Step 0 〜 |
| `architecture.md` | アーキテクチャ制約。マルチテナント（tenant_id 必須）、ドメイン層の状態遷移一元管理、JWT RS256 認証、4ロール RBAC | Step 3 |
| `coding-standards.md` | コーディング規約。Go（gofmt/vet/lint、panic禁止、構造化エラー）、TypeScript（strict mode、any禁止） | Step 3 |
| `security-policy.md` | セキュリティポリシー。認証認可、テナント分離、レート制限、ファイルアップロード、ヘッダー、入力バリデーション、依存関係管理、ログ。実装時チェックリスト付き | Step 3 |
| `testing.md` | テスト方針。ドメイン層は単体テスト必須、リポジトリ層はテスト用DB、フロントは Vitest + Playwright、カバレッジ目標 | Step 3 |

### 対応不要（上流成果物で代替済み）

| ファイル | 代替先 |
|---------|--------|
| `review-checklist.md` | Step 7 work-breakdown のレビュー観点で定義済み |
| `data-handling.md` | db_schema.md §2.3, §9 で定義済み |
| `error-handling.md` | security.md §8, monitoring.md §2-4 で定義済み |
| `api-conventions.md` | openapi.yaml info.description + security.md で定義済み |

### 廃止

| 旧ファイル | 理由 |
|-----------|------|
| `ai-dev-framework/rules/branching.md` | `.claude/rules/` 体系に移行前の旧配置。ブランチ戦略は ops-032 で解決済み |

## 解決内容

### 2026-03-22: ルール文書一覧の整理

- 整備済み5ファイルの概要を記録
- 未整備4ファイルの概要と対応タイミングを定義
- 旧 `ai-dev-framework/rules/` 体系からの移行を反映

### 2026-03-23: 未整備4ファイルの対応不要判断

- Step 3〜6 で作成された設計成果物（openapi.yaml, security.md, monitoring.md, db_schema.md）が実質的にルール文書の役割を果たしていることを確認
- codex レビュー（054〜057）で検証した結果、設計書にギャップがあった箇所（エラーコード一覧の不整合、日時フォーマット未明記）は設計書自体を修正して対応
- 独立したルール文書の作成は不要と判断。review-checklist.md は Step 7 work-breakdown のレビュー観点で代替

## 解決日
2026-03-23

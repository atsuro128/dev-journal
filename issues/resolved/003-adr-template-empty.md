# ADR テンプレートが空

## 発見日
2026-03-04

## 関連ステップ
Step 3（アーキテクチャ設計）までに対処

## カテゴリ
architecture

## 問題
`templates/ADR-template.md` がヘッダーのみで、テンプレート本文がない。

## 影響
Step 3 で ADR を書く際に雛形がなく、フォーマットが統一されない。

## 提案
標準的な ADR 構成で整備する:
- Title / Date / Status (Proposed / Accepted / Deprecated)
- Context（背景・課題）
- Decision（選択した方針）
- Consequences（影響・トレードオフ）

## 解決内容

`ai-dev-framework/templates/ADR-template.md` に標準的な ADR テンプレートが整備済み（ステータス / 日付 / 背景 / 検討した選択肢 / 決定 / 理由 / 影響・結果）。3/8 のリポジトリ再構成時に整備されたと推定。既存 ADR（ADR-001, ADR-002, ADR-004）もこのテンプレートに従っている。

## 解決日
2026-03-16

# 却下後の再申請フローの DB スキーマが未定義

## 発見日
2026-03-04

## 関連ステップ
Step 2（ドメイン設計）で対処

## カテゴリ
domain

## 問題
PROJECT_SUMMARY.md では「却下後の再申請: 新規レポートとして作成（元レポートへの参照を保持）」と記載があるが、`expense_reports` テーブルに参照元レポートIDのカラムがない。

## 影響
仕様と DB 設計の不整合。実装時に設計判断が必要になる。

## 提案
- `expense_reports` に `original_report_id` (nullable UUID FK) を追加する
- または `rejection_reason` と合わせてコメント/参照を別テーブルで管理する

## 解決内容
PROJECT_SUMMARY.md は初期構想であり、カラムレベルのDB設計は Step 5（詳細設計）で確定する。Step 2（ドメイン設計）では「再申請時に元レポートへの参照を保持する」というドメインルールを定義すれば十分。全体フロー（portfolio_project_steps.md）に沿った正常な進行であり、現時点で対応不要。

## 解決日
2026-03-08

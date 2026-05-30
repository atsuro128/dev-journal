# dev-journal

経費精算 SaaS（[expense-saas](../expense-saas/)）の開発プロセス記録。設計成果物・進捗・課題・意思決定の履歴をすべて残す。

---

## このリポジトリについて

「動くもの」だけでなく「**どう作ったか**」を可視化することを目的とした記録用リポジトリ。実装コードは持たず、要件定義から UAT 完走までの全プロセスを以下の観点で残している。

- **設計成果物**: 要件 / ドメイン / アーキテクチャ / 詳細設計 / UI / テスト / 運用設計（[deliverables/docs/](./deliverables/docs/)）
- **進捗・チケット**: Step 単位の進捗、各チケットの状態（[progress-management/](./progress-management/)）
- **課題**: 設計中・実装中・レビューで発見した issue（[issues/](./issues/)）
- **意思決定の経緯**: セッションログ・レビュー履歴（[archives/](./archives/)）

採用担当者・エンジニアが「なぜその技術にしたか」「どう設計判断したか」「どこで詰まったか」を追跡できる状態を維持している。

## ディレクトリ構成

| ディレクトリ | 役割 |
|---|---|
| [`deliverables/docs/`](./deliverables/docs/) | 設計成果物の正本（プロジェクトゴール・用語・要件・ドメイン・アーキ・基本/詳細設計・UI・テスト・運用設計・基盤） |
| [`progress-management/`](./progress-management/) | 進捗管理（progress.md / session-log.md / チケット / タスク計画） |
| [`issues/`](./issues/) | 課題管理（open / pending-review / resolved の 3 フォルダで状態管理） |
| [`review-findings/`](./review-findings/) | codex レビュー指摘の管理（`open/` / `pending-review/` / `resolved/` の 3 フォルダで状態管理） |
| [`archives/`](./archives/) | アーカイブ（日報 / 週報 / セッションログ / レビューログ / 進捗履歴） |
| [`logs/`](./logs/) | テスト実行結果ログ |
| [`references/`](./references/) | 参照資料（外部リンク / 意思決定メモ / dev-commands / devcontainer ドキュメント） |

## 何を見ればプロセスが分かるか

| 関心 | 参照先 |
|---|---|
| プロジェクトの目的・スコープ | [`deliverables/docs/00_goals.md`](./deliverables/docs/00_goals.md) / [`02_scope.md`](./deliverables/docs/02_scope.md) |
| 用語の定義 | [`deliverables/docs/01_glossary.md`](./deliverables/docs/01_glossary.md) |
| 設計判断（ADR） | [`deliverables/docs/30_arch/adr/`](./deliverables/docs/30_arch/adr/) |
| API / DB / 認可 / セキュリティ | [`deliverables/docs/50_detail_design/`](./deliverables/docs/50_detail_design/) |
| テスト戦略 | [`deliverables/docs/60_test/`](./deliverables/docs/60_test/) |
| 運用設計 | [`deliverables/docs/70_operations/`](./deliverables/docs/70_operations/) |
| 全体の進捗 | [`progress-management/progress.md`](./progress-management/progress.md) |
| 直近のセッション引き継ぎ | [`progress-management/session-log.md`](./progress-management/session-log.md) |
| 過去のセッション履歴 | [`archives/session-logs/`](./archives/session-logs/) |
| 未対応 issue 一覧 | [`issues/open/`](./issues/open/) |

## issue 管理の状態フロー

```
open/ → pending-review/ → resolved/
（未対応）   （解決済み・レビュー待ち）  （クローズ）
```

- ファイル命名: `NNN-kebab-case.md`（プロダクト系）/ `ops-NNN-kebab-case.md`（運用系）
- 番号は 3 桁ゼロ埋めのグローバル連番（3 フォルダ横断で一意）
- 詳細な運用は [`ai-dev-framework/templates/issue-template.md`](../ai-dev-framework/templates/issue-template.md) と [`../.claude/skills/issue/`](../.claude/skills/issue/) 参照

## 関連リポジトリ

| リポジトリ | 役割 |
|---|---|
| [`../expense-saas/`](../expense-saas/) | プロダクト本体 |
| [`../ai-dev-framework/`](../ai-dev-framework/) | 本記録を生み出すための AI 駆動開発フレームワーク |
| [`../`](../) | メタリポジトリ（全体統括） |

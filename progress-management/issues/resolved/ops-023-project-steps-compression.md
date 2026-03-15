# project_steps.md の圧縮

## 発見日
2026-03-15

## カテゴリ
project-management

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
全 Step（横断）

## 問題

project_steps.md は約340行の全体地図だが、毎セッション全文を読み込んでおり、作業中の Step に無関係な情報がコンテキストのノイズになっている。work-breakdown を導入したことで、各 Step の詳細はそちらに移行する前提だが、project_steps.md 自体の圧縮が未対応。

## 影響

- セッション開始時に不要な情報を読み込む時間・トークンの浪費
- 作業に無関係な Step の記述がコンテキストを圧迫し、作業精度に悪影響

## 提案

project_steps.md を以下の構成に圧縮する:
- 全体ステップの一覧（索引としての表）
- 各ステップ共通ワークフロー（レビューフロー等の共通事項）
- 各 Step の詳細は work-breakdown に委譲し、project_steps.md からは削除またはリンクのみに

---

## 解決内容

- project_steps.md を 339行 → 63行に圧縮（索引テーブル＋共通ワークフローのみ）
- Step 0〜7 の詳細を work-breakdown/ 配下に分離（7ファイル）
- deliverables/docs 構成ガイドを deliverables/docs-structure.md に切り出し
- 最終成果物（ゴール）セクション・レビュー参照先テーブル・メモセクションを削除
- 懸念セクション運用を廃止し、全件 issue 起票に統一（workflow.md も同時に簡素化）
- progress.md のスリム化（ステップ別成果物・直近タスク削除、Step 4+5 統合）
- 派生 issue: ops-033（運用・監視設計）、ops-034（マイルストーン移行）、ops-035（機能別進捗移行）

## 解決日
2026-03-15

# 引継ぎ資料: issue 116-121 対応

作成日: 2026-04-20
作成者: セッション A（別チャネルの Claude Code セッション）
対象セッション: 新規 Claude Code セッション（セッション B）

---

## ⚠️ 最重要: 並行セッションの存在

**本セッション（セッション B）起動時、別のセッション A が以下の issue を並行対応中です**:

| issue | 状態 |
|-------|------|
| 123（SMK-026 文言整合） | reviewer PASS 済み、コミット承認待ち |
| 125（SMK-028 文言整合） | 同上（123 と束ねて 1 コミット） |
| 124（API クライアント error mapping） | PR #70 作成済み、`/test`→reviewer→codex 待ち |
| 126（レポート作成/編集 成功トースト） | PR #69 作成済み、同上 |
| 127（明細期間外警告、ConfirmDialog） | Phase 1+2 設計書変更済み・reviewer 実行中、Phase 3+4 未着手 |
| 128（SMK 残件 6 件集約） | 起票済み（セッション A で方針確定予定、実装は別セッション） |

### セッション B が触ってはいけない範囲

- **dev-journal のファイル**（セッション A が編集・コミット中）:
  - `deliverables/docs/60_test/manual_checklists/smoke_check.md`（SMK-026/028）
  - `deliverables/docs/55_ui_component/state-management.md`（§6.5.6 新設）
  - `deliverables/docs/50_detail_design/screens/report-detail.md`（§6 期間外警告）
  - `deliverables/docs/55_ui_component/screens/report-detail.md`（ItemSlidePanelProps 拡張）
  - `deliverables/docs/60_test/test_cases/items.md`（ITM-FE-099〜106 採番済み）
  - `issues/open/128-smoke-check-additional-wording-mismatches.md`

- **expense-saas のファイル**（セッション A の PR 対象）:
  - `frontend/src/api/client.ts`、`frontend/src/lib/error-messages.ts`（PR #70）
  - `frontend/src/pages/reports/ReportCreatePage.tsx` / `ReportEditPage.tsx` / `ReportDetailPage.tsx`（PR #69）
  - `frontend/src/pages/reports/ItemForm.tsx` / `ItemSlidePanel.tsx`（issue 127 Phase 4 で編集予定）

- **アクティブなブランチ**:
  - `fix/124-api-client-error-message-mapping`（セッション A）
  - `fix/126-report-create-edit-success-toast`（セッション A）
  - `fix/127-item-date-outside-period-warning`（セッション A、Phase 3+4 で作成予定）

- **起票済み issue 番号**: 128 まで使用中。新規 issue 起票時は **129 から採番**。

### セッション B の責務範囲

**issue 116 / 117 / 118 / 119 / 120 / 121** のみ。他は触らないこと。

---

## セッション B の進め方

### 1. セッション開始時

`/session-start` スキルを実行して状態確認。ただし以下に注意:

- `progress.md` の「残存 issue」欄には 116〜128 が列挙されているが、123/125/124/126/127/128 は**セッション A の責務**。
- `session-log.md` が古い可能性がある（セッション A が最後に更新するため）。本ファイルを優先参照すること。

### 2. issue 116-121 の方針確定プロセス

`/issue 対応` スキル経由で各 issue ファイルを読み込み、**ユーザーと直列で1件ずつ論点を議論** して方針を確定する。セッション A でも同様のプロセスで 123〜127 の方針を確定した（参考: 本ファイル末尾「セッション A の方針確定プロセス例」）。

### 3. 方針確定後、並列実行可能性を評価

- 対象ファイルが重複しないタスクは別エージェント・別ブランチで並列化（`.claude/memory/feedback_parallel_agents_by_scope.md`）
- worktree 起動前に `git -C expense-saas fetch origin` を実行（workflow.md §実行時）
- 対象ファイルが **セッション A の変更範囲と衝突する場合は、セッション A の完了まで待つ**

### 4. 実装・レビュー・マージ

`workflow.md` の「設計成果物フロー」または「PR フロー」に従う。

---

## issue 116-121 の概要

ファイル: `dev-journal/issues/open/{NN}-*.md` を参照

| ID | タイトル | 主な対象 |
|----|---------|---------|
| **116** | 一覧画面のスケルトン表示がページ全体を置換（設計はテーブル領域のみ） | FE 実装（SCR-RPT-001 等の一覧画面、共通 PageSkeleton コンポーネント） |
| **117** | period_start/period_end が RFC3339 date-time で返され日付入力にプリフィルされない | BE+FE（OpenAPI スキーマ or FE パース層） |
| **118** | ItemForm のリアルタイムバリデーションが動作しない（useForm の mode 未指定） | FE 実装（react-hook-form 設定） |
| **119** | ReportForm 日付フィールドで onBlur バリデーション未発火（Controller が field.onBlur を渡していない） | FE 実装（react-hook-form Controller） |
| **120** | 日付フィールドを空にすると Zod デフォルトエラー「Invalid input: expected string, received null」が露出 | FE 実装（Zod スキーマ or FE エラー表示層） |
| **121** | 422 レスポンスに details 配列が欠落（OpenAPI 契約違反） | BE 実装（エラーハンドラの details 配列組み立て） |

### 依存関係・衝突のヒント

- **118 / 119 / 120**: react-hook-form 周りで関連。同一ファイル編集の可能性あり
  - 118: `ItemForm.tsx`（セッション A も issue 127 で編集予定 → 衝突回避必要）
  - 119: `ReportForm.tsx`（セッション A は触らない）
  - 120: 日付フィールドの Zod スキーマ（共通コンポーネント `AppDatePicker` または各 Form）
- **121**: BE 実装（`expense-saas/internal/handler/*`）、FE には 124 で対応済みの `VALIDATION_ERROR` サーバー message 優先ロジックと連携
- **117**: API レスポンススキーマ改修 → BE 契約変更のインパクト大
- **116**: 共通コンポーネント `PageSkeleton` の仕様確認が先

### 推奨順序（方針確定は直列、実装は並列化可）

1. 先に**仕様理解が必要なもの**: 116（スケルトン）、117（API スキーマ）、121（BE エラー形式）
2. FE ロジック改修系: 118、119、120
3. **118 の着手前にセッション A の issue 127 マージを必ず確認**（`ItemForm.tsx` 衝突回避）

---

## セッション A の方針確定プロセス例（参考）

以下はセッション A で実施したプロセス。セッション B でも同じ流れで論点を議論してユーザー確定すること。

1. **各 issue を個別に開く** — `/issue 対応` 経由で本文を読む
2. **論点を抽出して提示** — 方針候補（案 A/B/C）、影響範囲、対応条件、トレードオフを明示
3. **ユーザーに論点ごとに判断を依頼** — 「論点1: UI パターンは A / B / C？」「論点2: 文言は…」
4. **確定内容を要約** — issue 単位で方針表（担当・ブランチ・フロー・作業内容）を作る
5. **全件確定後、並列起動計画を立てる** — 対象ファイル衝突を分析、波次（Wave 1 / Wave 2 …）を決める
6. **エージェント起動** — `run_in_background: true` + `isolation: "worktree"`（expense-saas 編集時）

### セッション A で特に議論になった観点（116-121 にも適用できる）

- **設計書の空白を埋める必要があるか**（文言が設計書に未定義ならまず設計書を追記する → 設計成果物フロー）
- **実装修正だけで済むか、テスト/設計も併せて変更が必要か**
- **テスト実装と機能実装を直列に分けてエージェントを切り替えるか**（つじつま合わせリスク防止の観点）
- **1 issue にまとめるか複数に分けるか**（対象ファイルが同じなら 1 issue / PR で対応）

---

## 制約・ルール（再掲）

- サブエージェント起動は `run_in_background: true`
- `expense-saas/` 編集エージェントは `isolation: "worktree"` + ブランチ名指定
- `isolation: "worktree"` エージェント起動の直前に `git -C expense-saas fetch origin`
- サブエージェントにローカル CI を走らせない（指揮役が `/test` スキルで実施）
- PR 本文に絵文字フッターを含めない
- コミット前にユーザー承認を得る（`/commit` スキル）
- 新規 issue 起票時は 129 以降から採番

---

## 参考ファイル

- `ai-dev-framework/guide/workflow.md` — ワークフロー手順
- `ai-dev-framework/guide/work-breakdown/step11-system-test/main.md` — Step 11 完了条件
- `dev-journal/progress-management/progress.md` — 残存 issue 一覧
- `.claude/memory/MEMORY.md` — ルール・フィードバック一覧
- `.claude/memory/feedback_parallel_agents_by_scope.md` — 並列化判断基準
- `.claude/memory/feedback_issue_upstream_check.md` — issue 着手時の上流確認ルール

---

## セッション B 着手時の確認事項

セッション B 開始前に、以下をユーザーに確認すること:

1. セッション A がまだ稼働中か、完了済みか
2. progress.md / session-log.md はセッション A 完了後の最新か
3. ブランチ `fix/124` `fix/126` `fix/127` のマージ状況
4. issue 123〜128 が `resolved/` に移動済みか
5. セッション B で扱う範囲の最終確認（116〜121 のみで OK か、追加あるか）

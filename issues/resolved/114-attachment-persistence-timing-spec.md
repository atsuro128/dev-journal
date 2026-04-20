# 添付の永続化タイミング仕様明示（即時保存方式の明確化）

## 発見日
2026-04-18

## カテゴリ
design / frontend / ux / documentation

## 影響度
低（仕様明示と UI 案内の追加のみ。挙動変更なし）

## 発見経緯
issue 108 の方針議論中に、現状の「ファイル選択時点で S3+DB に即時永続化される」挙動と、ユーザーの「フォーム保存ボタンを押すまで何も確定しない」期待値の乖離が判明。issue 108 のスコープを「フォーム編集中の操作整合性」に限定し、本 issue として独立起票。

## 関連ステップ
Step 5（詳細設計）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし

## 問題

### 現状の挙動

`useUploadAttachment` は POST 成功直後にキャッシュ無効化を実行し、ファイル選択時点で S3 と DB に永続化される（`useUploadAttachment.ts:36-42`）。フォーム保存と独立した即時保存方式。

具体的なユーザー視点での挙動:
- 明細を編集 → 添付追加 → 「キャンセル」を押す → **添付は残ったまま**
- 明細を編集 → 添付追加 → ブラウザを閉じる → **添付は残ったまま**
- 明細を編集 → 既存添付を削除 → 「キャンセル」を押す → **削除は確定したまま**

### 設計書での欠落

以下の設計書に永続化タイミングの明記なし:
- `dev-journal/deliverables/docs/50_detail_design/files.md` §3（アップロードフロー）: 「API プロキシ方式」のみ記述、永続化タイミング未記載
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §7（添付ファイル管理）: UI 仕様のみ、保存ボタンとの関係性未記載

ユーザーは「保存ボタンを押すまで何も確定しない」と期待するが、設計書側でも明示されていないため、実装と期待値の乖離を判断する根拠が無い状態。

## 対応方針（合意済み）

**仕様として「即時保存方式」を明記し、UI で案内する**（B-② 案）。

### 1. 設計書追記

#### `files.md` §3 アップロードフロー
- 「ファイル選択時点で S3 + DB に永続化される。フォーム保存ボタンとは独立」を明記
- 削除も同様に即時実行されることを明記

#### `report-detail.md` §7 添付ファイル管理
- 添付の永続化タイミングを「即時保存」と明記
- フォーム保存・キャンセルとの関係を明記（「キャンセルしても添付は残る」「既存添付の削除も即時実行され、キャンセルで復活しない」）

### 2. UI 案内表示

`AttachmentArea` コンポーネントに常時表示の案内文を追加:

```
※ 添付ファイルは選択した時点で保存されます。フォームをキャンセルしても添付は残ります。
```

文言は用語集（`dev-journal/deliverables/docs/01_glossary.md`）と整合をとる。

### 3. ToolTip / Popover の検討

案内文を常時表示する以外の選択肢:
- 削除ボタン押下時に確認ダイアログ「この添付を削除します。元に戻せません」（既存 issue 103 で実装済みのため、文言調整のみで良いか確認）
- info アイコンで Popover 表示（情報密度を抑える）

→ MVP では常時表示テキスト + 削除確認ダイアログの文言調整、で進める。Popover 化は post-MVP。

## 採用しない案

### B-① ロールバック方式（キャンセル時に DELETE API）
- 既存添付削除のロールバックも必要となり、UI 状態管理の複雑度が増す
- ネットワーク断時に S3 孤児ファイルが残るリスク
- → 「分離維持」というユーザー方針と整合しない

### B-③ 保留 → 確定 DB 改修
- DB スキーマ変更（temp テーブル or status カラム追加）が必要
- MVP 範囲外、post-MVP で再検討する場合は別 issue として起票

## 修正対象ファイル

- `dev-journal/deliverables/docs/50_detail_design/files.md` §3
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §7
- `dev-journal/deliverables/docs/01_glossary.md`（必要に応じて文言追加）
- `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`（案内文追加）

## 完了条件

- files.md §3 に「添付は API 成功時点で S3 + DB に永続化、フォーム保存と独立」が明記されている
- report-detail.md §7 に「添付の永続化タイミング」「フォームキャンセル時の挙動」「既存添付削除の挙動」が明記されている
- AttachmentArea に「※ 添付ファイルは選択した時点で保存されます。フォームをキャンセルしても添付は残ります。」相当の案内文が表示される
- 文言が用語集と整合している
- 既存テストが通過する

## 関連

- 108: フォーム編集中の操作整合性 — 本 issue の発見起点。状態管理は本 issue とは独立（添付は分離保存方針）
- 115: 新規明細での添付方法 — 永続化タイミング方針が両立する形で実装する必要あり
- 100: AttachmentUploader UI 改善 — 解決済み

## 次セッション着手手順（別セッションでワークツリー対応想定）

1. `/issue 対応 114` で本 issue を読み込む
2. 上流確認: `dev-journal/deliverables/docs/50_detail_design/files.md` §3、`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §7
3. designer エージェント（dev-journal 編集）で設計書追記 → 内部レビュー
4. ユーザー合意後、frontend-developer エージェント（worktree、ブランチ名例: `fix/114-attachment-persistence-spec`）で AttachmentArea に案内文追加
5. PR 作成 → /test → reviewer → codex → マージ
6. 設計書追記は dev-journal リポジトリで独立コミット

並列化: 本 issue（設計書 + 案内文の小規模変更）は issue 108（並行操作 + 破棄ダイアログ）と**ファイル重複あり**（AttachmentArea）。先に 114 を片付けて 108 を進めるか、108 と同一 PR にまとめるかは別セッション開始時に判断する。

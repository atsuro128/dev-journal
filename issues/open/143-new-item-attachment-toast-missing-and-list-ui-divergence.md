# 新規明細追加モードの添付トースト不在 + リスト UI 構造が編集モードと不一致

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design

## 影響度
中（UX 一貫性の問題。機能的には動作するがフィードバック欠如でユーザー混乱の可能性）

## 発見経緯
Step 11-A SMK-052（トースト文言の自然さ）手順 6（添付アップロード）・手順 7（添付削除）にて、Member アカウントで明細スライドパネルの「新規追加モード」を開き、ファイルを添付・削除する操作時に、編集モードと UX が乖離している点を発見。

## 関連ステップ
Step 10-G（添付ファイル機能）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 関連 issue

- **#129**（resolved, PR #87）: add モードにプレビュー対応 + ラベル整理を入れたが、トーストとリスト UI 構造は本 issue の担当範囲外だった
- **#115**（resolved）: ローカル保留方式の根拠
- **#136**（open）: 明細編集の変更破棄ダイアログの UI 不統一（同系統の UI 一貫性課題）

## 問題

### 発見 1: 新規追加モードでアップロード成功トーストが出ない

`expense-saas/frontend/src/pages/reports/AttachmentArea.tsx` の `AttachmentAreaAddMode`（L240-267）は `AttachmentUploader mode="add"` のみ描画し、`AppToast` を置いていない。さらに `onUploadSuccess={() => {}}` の空実装。

編集モード（`AttachmentAreaContent` L188-231）ではファイル選択後に「ファイルをアップロードしました」トーストが表示される。

### 発見 2: 新規追加モードで添付削除完了トーストが出ない

同じく `AttachmentAreaAddMode` に `AppToast` がないため、保留ファイル削除（`×` ボタン）時のフィードバックなし。

編集モードでは `AttachmentAreaContent.tsx:162` で `showToast('success', '添付ファイルを削除しました')` が発火する。

### 発見 3: ファイル名表示領域の UI 構造が編集モードと異なる

| 要素 | 編集モード（AttachmentList.tsx L147-184） | 新規追加モード（AttachmentUploader.tsx L320-354） |
|------|-----|-----|
| 要素構造 | `<ul><li>` | MUI `<List dense><ListItem>` |
| ファイル名 | `<Button variant="text">` | `<Button variant="text">`（同じ） |
| ダウンロードアイコン | `<IconButton>↓</IconButton>` あり | **なし**（仕様通り: サーバー未保存のため） |
| サイズ表示 | `<span>` | `<Typography variant="caption">` |
| 削除ボタン | `<Button color="error">削除</Button>` | `<IconButton>×</IconButton>`（`secondaryAction`） |

ユーザー指摘: **編集モード側の UI が正**（構造・削除ボタンの見た目を編集モードに合わせる）。ダウンロードアイコン不在は pending file の仕様上正しいのでそのまま。

## なぜ #129 で対処されなかったのか

#129 のスコープは「プレビュー対応 + ラベル整理」に限定されていた。原 issue の「現状の挙動（編集モードとの差分）」表にはトーストとリスト UI 構造が含まれていなかった。本 issue は #129 が未到達だったギャップを補完する。

## 提案

### 採用方針: 案 A（UI 一貫性優先、内部実装の差異をユーザーに見せない）

- **トースト文言は編集モードと完全同一**: 「ファイルをアップロードしました」「添付ファイルを削除しました」
- ローカル保留方式という実装詳細は UI 上に露出しない（#129 の方針「実装詳細を UI に露出させない」と整合）
- フロント上ではアップロードしているのでユーザー視点では嘘ではない、という判断

### 却下案

#### 案 B: 文言を「ファイルを保留しました」「保留ファイルを削除しました」に変更
却下理由: 実装詳細（ローカル保留方式）が UI に露出し、#129 の方針と矛盾する。

#### 案 C: トーストなし、pendingFiles 行の追加で視覚フィードバック
却下理由: 編集モードでトーストがあるのに add モードだけないのは一貫性欠如。

### 実装スコープ

#### 1. `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`
- `AttachmentAreaAddMode` に toast state + `AppToast` を追加
- `onUploadSuccess` を「ファイルをアップロードしました」トーストを表示するハンドラに置換
- pending file 削除時のトースト発火経路を追加（AttachmentUploader の `handlePendingFileRemove` からコールバックを受けられるよう props 追加、または AttachmentUploader 内で直接トースト表示する設計に変更）

#### 2. `expense-saas/frontend/src/pages/reports/AttachmentUploader.tsx`（mode='add' 経路）
- pending file 表示の UI を `<ul><li>` 構造に揃える
- 削除ボタンを `<Button variant="text" color="error">削除</Button>` に変更
- サイズ表示を `<span>` に変更
- ダウンロードアイコンは追加しない（サーバー未保存のため意味がない）

### 設計書更新スコープ

- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6〜§7（添付エリア）
  - add/edit 両モードでトースト文言・リスト UI 構造が統一されることを明記
  - ダウンロードアイコンが add モードでのみ非表示であることを明記
- `dev-journal/deliverables/docs/55_ui_component/state-management.md`
  - 添付アップロード成功/削除成功のトーストトリガーが add/edit 両モードで発火することを明記
- 必要に応じて `55_ui_component/common-components.md` にも AttachmentList / AttachmentUploader の UI 仕様差異を無くす旨を追記

### テストスコープ

- `dev-journal/deliverables/docs/60_test/test_cases/attachments.md` または `items.md` の ATT 系
  - add モードのアップロード成功トースト発火テスト
  - add モードの pending 削除トースト発火テスト
  - リスト UI 構造（`<ul><li>` + 削除ボタン text）のテスト
- 対応する Vitest ケースを追加

## 完了条件

- 新規追加モードでファイルを選択すると「ファイルをアップロードしました」トーストが表示される
- 新規追加モードで `×` で保留ファイルを削除すると「添付ファイルを削除しました」トーストが表示される
- 新規追加モードのファイル一覧が編集モードと同じ UI 構造（`<ul><li>` + text 「削除」ボタン + `<span>` サイズ）で描画される
- ダウンロードアイコンは add モードでのみ非表示（pending file はサーバー未保存のため）
- 設計書（report-detail.md §6-7, state-management.md）が新仕様を反映
- test_cases ID が新規採番されテストが追加

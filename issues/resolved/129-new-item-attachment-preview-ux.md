# 新規追加モードの添付 UX を編集モードと同等にする（プレビュー対応 + ラベル整理）

## 発見日
2026-04-21

## カテゴリ
implementation / ui-design

## 影響度
中（新規追加モードでプレビュー不可 + 実装詳細が UI 文言に露出しており、編集モードとの UX 乖離が認知負荷を生む）

## 発見経緯
user-report（Step 11-A SMK-031 / SMK-032 実施中にユーザーから指摘）

## 関連ステップ
Step 11-A（ローカル動作確認）/ Step 10-G（添付ファイル機能実装）

## ブロッカー
なし（Step 11-A のブロッカーではない後追い改善）

## 問題

### 現状の挙動（編集モードとの差分）

| 要素 | 編集モード | 新規追加モード | 備考 |
|------|----------|--------------|------|
| プレビュー表示 | ○ 署名付き URL でプレビュー可能 | × 不可 | ユーザーはファイル選択後に内容確認できない |
| ラベル | ファイル名のみ | 「保存後にアップロード予定」ラベルが付与される | 実装詳細（#115 の local buffer 方式）が UI に露出 |
| 冗長な説明文 | なし | 「ファイルを選択すると保存時にまとめてアップロードします。」（`AttachmentUploader.tsx:331-335`） | 動作が自明な内容を案内しており、認知負荷を増やしている |

### 関連既存 issue

- **#115**（resolved: new-item-attachment-local-buffer）: 新規追加モードで `itemId` 未確定のため即時アップロード不可という API 制約を回避するため、local buffer 方式 + 保存時順次アップロードを採用。本 issue は #115 のアーキテクチャを維持しつつ、プレビューと文言のみを編集モードと揃える
- **#102**（resolved: attachment-preview-missing）: 編集モードの uploaded 添付にプレビュー機能を導入。新規追加モードの pending file は対象外だった

### なぜ新規追加モードだけ挙動が違うのか（根拠）

添付アップロード API `POST /api/reports/:reportId/items/:itemId/attachments` は `itemId` 必須で、新規追加モードでは保存前に `itemId` が存在しない。DB 側も `attachments.item_id NOT NULL` 制約あり。
このためサーバーへの即時アップロードは不可能で、#115 ではフロントのみで local buffer 方式を採用した（他の代替案は却下: 自動 draft 作成＝孤児レコード、temp API＝BE 改修、multipart 複合 POST＝API 分岐）。

本 issue ではこのアーキテクチャ判断は維持する。**変更するのはプレビュー表示と文言のみ**で、API / BE / DB は一切変更しない。

## 影響

- UX: 中（保存前にファイル内容を確認できず、ラベル文言が実装詳細を露出）
- 実害: なし（データ・認可・ロジックへの影響なし）

## 提案

### 方針: 案 α（フロントのみ、API 追加なし）

`URL.createObjectURL(file)` で `File` オブジェクトからブラウザ内プレビュー URL を生成し、既存のプレビュー UI を pending file にも適用する。

### 変更スコープ（推定）

| ファイル | 変更内容 |
|---------|--------|
| `frontend/src/pages/reports/AttachmentList.tsx`（または `AttachmentArea.tsx` の add モード分岐） | pending file に対して `URL.createObjectURL` でプレビュー URL を生成し、ファイル名クリック = プレビュー、↓ アイコン = （保存前は無効 or 非表示）で UI 統一 |
| `frontend/src/pages/reports/AttachmentUploader.tsx:331-335` | 「ファイルを選択すると保存時にまとめてアップロードします。」を削除 |
| `frontend/src/pages/reports/AttachmentList.tsx` または AttachmentItemRow 相当 | 「保存後にアップロード予定」ラベルを削除 or 控えめ表示（ファイル名横のバッジ等、存在はするが主張しない形） |
| `frontend/src/pages/reports/__tests__/*.test.tsx` | `URL.createObjectURL` / `URL.revokeObjectURL` 呼び出しの検証、プレビュー UI の pending file 対応を検証 |
| `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6 §7 | 新規追加モードのプレビュー挙動を設計書に明記（#115 の記述を更新） |

### 技術的注意点

- **メモリリーク対策**: `URL.createObjectURL` で生成した URL は `URL.revokeObjectURL` で明示解放しないとリーク。以下のタイミングで revoke する:
  - コンポーネント unmount 時（`useEffect` の cleanup）
  - 保留ファイルが削除された時
  - 保存成功してサーバー側 URL に切り替える時
- **ダウンロード操作の扱い**: 保存前の pending file に対して「ダウンロード」操作を提供するかは設計判断（ブラウザのファイルシステム上に既にあるので意味が薄い）。MVP では非表示 or 無効化で十分
- **保留ファイル削除**: #115 実装済み（local buffer からの除外）。UI はそのまま使用

### 却下案

- **β: サーバー即時反映まで含めた完全一致** — BE 改修（temp API 新設 or 自動 draft 作成）が必要。孤児レコード or cleanup ジョブの複雑度が増し、ポートフォリオ規模ではオーバーキル。#115 の却下理由がそのまま有効

## 完了条件

- 新規追加モードでファイル選択時に、編集モードと同じプレビュー UI（ファイル名クリック = プレビュー表示）が動作する
- 「ファイルを選択すると保存時にまとめてアップロードします。」説明文が削除されている
- 「保存後にアップロード予定」ラベルが削除 or 控えめ表示に変更されている
- `URL.createObjectURL` で作成した URL が適切なタイミングで `URL.revokeObjectURL` により解放されている（メモリリークなし）
- 既存の local buffer 方式（#115）と保存時順次アップロードの挙動は変わらない
- `report-detail.md` §6 §7 が新規モードのプレビュー挙動を反映して更新されている
- FE 既存テストが通過 + プレビュー周りの新規テストが追加されている

## 関連

- **#115**（resolved）: 新規追加モードの local buffer 方式の根拠。本 issue はそのアーキテクチャ判断を維持しつつ UX のみ改善
- **#102**（resolved）: 編集モード添付のプレビュー機能。本 issue は同じプレビュー UI を pending file にも拡張
- **#100**（resolved）: AttachmentUploader UI 改善。本 issue はさらに文言整理を追加
- **#108**（resolved）: フォーム編集中の操作整合性（破棄確認ダイアログ含む）。pending file のプレビュー追加時に破棄フローへの影響なしを確認する

---

## 解決内容

PR #87（commit 8dd13f5）にて対応。新規追加モードで `URL.createObjectURL` を用いたブラウザ内プレビューを実装し、編集モードと同等の添付 UX を実現。「ファイルを選択すると保存時にまとめてアップロードします。」説明文および「保存後にアップロード予定」ラベルを削除。`URL.revokeObjectURL` による適切なメモリ解放も含む。

## 解決日

2026-04-22

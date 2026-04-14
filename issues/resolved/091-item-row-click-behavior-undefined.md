# draft 所有者が明細行クリックした場合の挙動が設計書に未定義

## 発見日
2026-04-14

## カテゴリ
design / frontend / clarification

## 影響度
中（設計ギャップ。実装は動作しているが仕様上曖昧）

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 前提データ準備中に、明細行をクリックしただけで添付ファイル選択 UI が操作可能になることを観察。仕様書を確認したが、draft 所有者の行クリック挙動が明記されていなかった

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（実装は動作しているが曖昧仕様）

## 概要

`report-detail.md` §5「明細テーブルの操作」L208-211 で明細行クリックの挙動が次のように定義されている:

| # | 操作 | 表示条件 | 挙動 |
|---|------|---------|------|
| 2 | 明細編集 | 所有者 AND status == draft | 各行の「編集」ボタン。スライドパネルに既存データをプリフィルして開く |
| 4 | 明細行クリック | 全ロール、全状態 | スライドパネルを閲覧モードで開く（draft 以外の状態、または所有者でない場合） |

しかし **「draft 所有者が明細行をクリックした場合」** の挙動が未定義。実装では以下のようになっている:

- `ReportDetailPage.tsx:348-354` — 常に `panelMode='view'` で開く
- `ItemSlidePanel.tsx:78` — `canModify = isOwner && reportStatus === 'draft'` → `true`
- `AttachmentArea.tsx:106` — `canModify=true` なら AttachmentUploader を表示 → **閲覧モードでも添付操作可能**

結果として、**「他項目は readonly だが添付だけ操作可能」という中途半端な状態** になっている。

## 事実

- 仕様書 L211 の括弧書き「(draft 以外の状態、または所有者でない場合)」は、draft 所有者の場合は行クリックで view モードに入らない、という解釈も可能
- しかし行クリックで何が起きるべきかは明記されていない
- 実装は「常に view で開く」という最小工数の選択をしている
- この結果、添付 UI が view モードでも利用可能になっている

## 設計判断の選択肢

### 案 A: draft 所有者の行クリック → 編集モードで開く（編集ボタンと統合）

- UX: 行クリックだけで編集開始できる。編集ボタンが冗長になる
- 実装: `handleItemClick` で `isOwner && status==='draft'` のとき `panelMode='edit'` に切り替え

### 案 B: 閲覧モードのときは添付操作も禁止

- UX: 閲覧はあくまで閲覧。編集操作したい場合は編集ボタン経由
- 実装: `AttachmentArea.canModify` の判定を `isOwner && status==='draft' && panelMode !== 'view'` に変更（または canModify の意味を変える）

### 案 C: 現状維持（閲覧モード + 添付操作可能）

- UX: 添付だけ操作できる中途半端な状態を明示的に許容
- 実装: 変更なし
- ドキュメント: `report-detail.md` §5 と §6 に「閲覧モードでも draft 所有者は添付操作のみ可能」と明記

## 推奨

**案 B** を推奨。理由:
- 閲覧モードの意味が明確（すべての操作ができない）
- 他項目が readonly なのに添付だけ操作可能は UX 上の不整合
- 編集ボタン経由なら添付操作可能、という導線がシンプル

## 要確認事項

- ユーザーにどの案を採用するか判断を仰ぐ
- 判断後、設計書（`report-detail.md` §5 L211）を修正

## 修正対象ファイル（案 B 採用時）

- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` — §5 の「明細行クリック」項目の挙動を明確化
- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx` — `canModify` の判定に `mode !== 'view'` を追加
- `expense-saas/frontend/src/pages/reports/__tests__/ItemSlidePanel.test.tsx` — view モード時に AttachmentArea の操作 UI が非表示になるテスト追加

## 完了条件

- 設計書で draft 所有者の明細行クリック挙動が明確に定義されている
- 実装が設計書に沿った挙動になっている
- 既存テストが通過し、新規テストが追加されて通過する

## 関連 issue

- 090（明細フォームプリフィル未実装）— 本 issue の修正と範囲が重なる可能性あり

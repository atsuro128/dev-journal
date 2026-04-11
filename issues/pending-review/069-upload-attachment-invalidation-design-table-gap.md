# useUploadAttachment の invalidation が設計テーブルより広い

## 発見日
2026-04-11

## カテゴリ
detail-design

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 10（機能実装）

## ブロッカー
なし

## 問題

state-management.md のミューテーション後キャッシュ無効化テーブルでは、`useUploadAttachment` の invalidation 対象は `['reports', 'detail', reportId]` のみと定義されている。

しかし実装では `['reports', reportId, 'items', itemId, 'attachments']`（useAttachments のクエリキー）も追加で invalidation している。これは `AttachmentArea` が `useAttachments` Hook で添付一覧を個別取得しており、レポート詳細の invalidation だけでは一覧が更新されないため。

### 対称性の問題

`useDeleteAttachment` は設計テーブル通り `['reports', 'detail', reportId]` のみで実装されている。レポート詳細レスポンスに `items[].attachments[]` がインラインで含まれるため動作はするが、`useUploadAttachment` と `useDeleteAttachment` で invalidation 方針が非対称。

## 影響

- 設計テーブルと実装の不一致
- Upload と Delete で invalidation パターンが非対称

## 提案

state-management.md の invalidation テーブルを更新し、添付ファイル系 Hook の invalidation 対象に `['reports', reportId, 'items', itemId, 'attachments']` を追加する。Upload / Delete で対称にする。

---

## 解決内容

## 解決日

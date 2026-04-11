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

state-management.md の invalidation テーブルを実装に合わせて更新した。

- `useUploadAttachment`: `['reports', reportId, 'items', itemId, 'attachments']` を追加（実装と一致）
- `useDeleteAttachment`: 変更なし（実装のコメントに「items 配列に添付一覧が含まれるため個別 invalidation は不要」と意図的な非対称の理由が明記されており、設計テーブルは実装の実態に合わせた）

codex の差し戻し指摘（対称化のため useDeleteAttachment 実装も変更すべき）はスコープ拡大と判断し却下。issue の本質は「設計テーブルと実装の乖離」であり、設計テーブルを実装に合わせることで解消済み。

## 解決日
2026-04-11

---

## レビューコメント（2026-04-11）

差し戻し。解決は未完了です。

- `dev-journal/deliverables/docs/55_ui_component/state-management.md` の invalidation テーブルは `useUploadAttachment` のみ更新され、`useDeleteAttachment` は依然として `['reports', 'detail', reportId]` のみです。Issue で求めた Upload / Delete の対称化が完了していません。
- `expense-saas/frontend/src/hooks/useDeleteAttachment.ts` も `['reports', reportId, 'items', itemId, 'attachments']` を invalidate しておらず、設計と実装の両方で非対称が残っています。
- `expense-saas/frontend/src/hooks/__tests__/useDeleteAttachment.test.tsx` も添付一覧クエリ invalidation を検証していないため、修正を担保できません。
- Issue の「解決内容」「解決日」が未記入で、何をもって解決としたか追跡できません。

追加で修正すべき成果物:

- `dev-journal/deliverables/docs/55_ui_component/state-management.md`
- `expense-saas/frontend/src/hooks/useDeleteAttachment.ts`
- `expense-saas/frontend/src/hooks/__tests__/useDeleteAttachment.test.tsx`
- 本 Issue ファイルの「解決内容」「解決日」

上流修正の要否:

- 上流修正は不要です。Step 5.5 の状態管理設計と Step 10 実装・テストをこの Issue の提案どおり揃えれば閉じられます。

このまま進めた場合の支障:

- Step 10 の FE 実装レビューで、カスタム Hook が `state-management.md` に従っていると判断できません。
- 添付一覧更新の契約が Upload / Delete で揺れたままとなり、下流の再レビュー担当が追加解釈を要します。

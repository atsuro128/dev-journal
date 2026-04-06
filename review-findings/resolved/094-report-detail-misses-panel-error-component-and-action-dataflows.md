# 094: レポート詳細の設計からパネルエラー表示と主要アクションのデータフローが欠落している

## 指摘概要
`report-detail.md` は最も複雑な画面であるにもかかわらず、明細パネルのサーバーサイドエラー表示コンポーネントが定義されておらず、さらにデータフロー図も `GET /api/reports/:id` と `GET /api/categories` しか書かれていない。Hook 一覧には提出・承認・却下・支払完了・明細 CRUD・添付操作まで列挙されているため、現在の記述では「どの親がどの Hook を持ち、どの Props で子に渡すか」が主要操作で追跡できず、Step 10 実装と Step 6/9 テスト設計の根拠として不足している。

## 根拠
- `50_detail_design/screens/report-detail.md:300-310` では、明細追加/編集失敗時のサーバーサイドエラーを「パネル上部にエラーメッセージを表示」と定義している。
- `55_ui_component/screens/report-detail.md:456-485` の `ItemFormProps` には `apiError` があるが、コンポーネントツリーと定義一覧にはその表示を担う `FormAlert` 相当のコンポーネントが存在しない。
- `55_ui_component/screens/report-detail.md:567-596` のデータフロー図は `useReport` と `useCategories` しか記載していない。
- その一方で `55_ui_component/screens/report-detail.md:602-616` では `useSubmitReport`, `useDeleteReport`, `useApproveReport`, `useRejectReport`, `useMarkAsPaid`, `useCreateItem`, `useUpdateItem`, `useDeleteItem`, `useAttachments`, `useAttachmentDownload`, `useUploadAttachment`, `useDeleteAttachment` を使用 Hook として列挙している。
- `55_ui_component/screens/report-detail.md:691-693` は §11〜§14 との対応を宣言しており、主要操作のデータフロー未記載はトレーサビリティ欠落に当たる。

## 判定
高 / 完了条件未達

## 修正方針案
`report-detail.md` に少なくとも以下を追加する。
- 明細パネル上部エラー表示を担うコンポーネント定義（既存 `FormAlert` を再利用するか、別名コンポーネントにするかを明示）
- 提出・削除・承認・却下・支払完了・明細 CRUD・添付一覧/ダウンロード/アップロード/削除の各 API について、Hook から親コンポーネント、子 Props への伝播までをデータフロー図に追記
- どのコンポーネントが成功時の再取得やパネルクローズを担うかを記述して、後続実装者が追加解釈なしで配線できる状態にする

## 再レビュー結果
- `FormAlert` を用いた明細パネル上部エラー表示、および主要操作のデータフロー自体は `55_ui_component/screens/report-detail.md` に追加されており、元の欠落は大筋で解消されている。
- 前回差し戻し対象だった `invalidateQueries` の不一致は解消された。`useSubmitReport` と `useApproveReport` の成功フローは `55_ui_component/state-management.md` §3「ミューテーション後のキャッシュ無効化」テーブルと一致している。
- ただし、同じワークフロー操作フロー内に別の契約不整合が残っている。`55_ui_component/state-management.md` では `useDeleteReport` の入力型を `string`（レポート ID）としている一方、`55_ui_component/screens/report-detail.md` では `useDeleteReport.mutate({ id })` とオブジェクト入力で記載されている。
- この不一致が残ると、Step 10 実装者は `mutate(id)` と `mutate({ id })` のどちらを正として実装すべきか判断できず、Step 9 の Hook/画面テストでも削除操作の呼び出し契約がぶれる。

## 追加で確認すべきファイルや観点
- `dev-journal/deliverables/docs/55_ui_component/screens/report-detail.md`
- `dev-journal/deliverables/docs/55_ui_component/state-management.md`
- 主要ミューテーションの `mutate` 入力型を Step 5.5 成果物間で統一すること
- 特に `useDeleteReport` の呼び出し契約を `state-management.md` と `report-detail.md` の両方で同一表記にそろえること

## 下流影響
- Step 10 実装時に削除操作の Hook 呼び出しシグネチャが画面設計と状態管理方針のどちらを正とすべきか割れ、実装者が追加解釈を強いられる。
- Step 6/9 の FE テストでも、削除操作の引数期待値を `string` と `{ id }` のどちらで固定すべきか曖昧なまま残る。

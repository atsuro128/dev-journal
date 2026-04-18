# 新規明細での添付方法（ローカル保持 + 保存時順次アップロード方式）

## 発見日
2026-04-18（issue 108 議論ログから分離・独立起票）

## カテゴリ
implementation / frontend / api / ux

## 影響度
中（新規明細追加時に添付が一切できない現状の UX 不整合を解消）

## 発見経緯
issue 108 議論中に、新規明細追加モードで AttachmentArea が表示されず添付ができない制約が UX として不自然との指摘あり。原因は API パスが `/api/reports/{reportId}/items/{itemId}/attachments` で `itemId` 必須のため、未保存の新規明細では呼び出せないこと。issue 108 のスコープを「フォーム編集中の操作整合性」に限定し、本 issue として独立起票。

## 関連ステップ
Step 5（API 詳細設計）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 問題

### 現状の挙動

1. 「+ 明細追加」ボタン → 新規明細フォーム表示
2. 金額・日付・カテゴリを入力
3. **AttachmentArea が非表示** （`item === null` で条件分岐、`AttachmentArea.tsx:144-147`）
4. ユーザーは添付するために以下の 2 ステップを強制される:
   - 一旦「保存」して明細を作成
   - 再度明細を開いて添付追加

### 技術的な原因

- API パスが `/api/reports/{reportId}/items/{itemId}/attachments` で `itemId` 必須（`openapi.yaml:1119-1250`）
- DB 制約: `attachments.item_id NOT NULL REFERENCES expense_items(item_id)`（`db_schema.md:410-429`）
- 新規明細は保存前に `itemId` が確定しないため、即時アップロード方式では添付不可

### 設計書での欠落

- `report-detail.md` §6 §7 に「新規明細では添付不可」の制限が**明記されていない**
- ユーザーが事前に把握できない

## 対応方針（合意済み）

**C パターン A**: ローカル保持 + 既存 API 順次呼び出し（フロントのみ改修）

### 仕組み

新規明細追加モード:
1. ファイル選択 → `File` オブジェクトをフロントの state に保持（メモリ）
2. ローカルに保留中のファイル一覧を「保存後にアップロード予定」として表示
3. クライアント側でバリデーション（MIME / サイズ）は選択時に実施
4. 「保存」ボタン押下時の処理:
   - a. POST `/api/reports/:id/items` で明細作成 → `itemId` 取得
   - b. 取得した `itemId` で各 `File` を順次 POST `/api/reports/:id/items/:itemId/attachments`
   - c. 全成功 → UI 更新、部分失敗 → エラートースト + 残った File を再試行可能な状態で UI 残留

編集モード（既存明細の編集）:
- **挙動変更なし**（即時アップロード方式を維持、issue 114 で仕様明示）

### 採用しない案

#### C-① 案内表示のみ
- 「明細を保存後に添付できます」案内のみ追加、2 ステップ操作のまま
- → UX が改善しないため不採用

#### C-② temp upload API 新設
- `POST /api/attachments/temp` + 明細保存時に `attach_to_item`
- → API 追加 + temp の cleanup ジョブが必要、複雑度が高い

#### C-③ 新規明細を draft 即時保存
- フォーム表示時に明細を即時 DB INSERT
- → DB に未確定データが残るリスク、cleanup 必要

#### C パターン B（複合 POST API）
- `POST /api/reports/:id/items` を multipart/form-data に拡張、明細 + 添付を 1 リクエスト
- → トランザクション整合性は良好だが、API 設計変更 + バックエンド改修が大きい
- 編集モード（即時アップロード）と API 形式が分岐する不整合

### 部分失敗時のロールバック方針

「保存」押下時に明細作成は成功したが添付の一部が失敗したケース:
- 明細自体は **作成済みのまま残す**（ロールバックしない）
- 失敗した添付ファイルはユーザーに警告トーストで通知（「N 件の添付ファイルがアップロードに失敗しました。再試行してください」）
- ユーザーは作成された明細を編集モードで開いて、失敗ファイルを再アップロードできる
- 既存の即時アップロード方式と挙動が一致するため認知負荷が小さい

## 「新規 vs 編集」のモード分岐について

本 issue の方針では、新規モードと編集モードで添付の挙動が分岐する:

| モード | 添付選択時 | 添付確定タイミング |
|--------|----------|-----------------|
| 新規 | ローカル state に保留 | 「保存」ボタン押下時 |
| 編集 | 即時 S3+DB 保存 | 即時 |

ユーザー視点での説明:
- 新規時: 「保存後に DB に反映されます」案内
- 編集時: 「添付は選択時点で保存されます」案内（issue 114 と同じ）

UX として混乱を招かないよう、AttachmentArea 内に**モード別の案内文**を表示する。

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`（新規モード分岐の追加、ローカル保持 state）
- `expense-saas/frontend/src/pages/reports/AttachmentList.tsx`（保留中ファイルの表示）
- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx`（保存時の順次アップロード制御）
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`（保存処理の拡張）
- `expense-saas/frontend/src/hooks/useUploadAttachment.ts`（既存。順次呼び出しの制御は呼び出し元で実装）
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6 §7（モード別の挙動を明記）

## 完了条件

- 新規明細追加モードで AttachmentArea が表示される
- 新規モードでファイル選択するとローカル state に保留され、「保存後にアップロード予定」として一覧表示される
- 保留中の添付に対するクライアント側バリデーション（MIME / サイズ）が動作する
- 「保存」押下時に明細作成 → 添付順次アップロードが実行され、全成功でパネルが閉じる
- 部分失敗時に明細は作成され、失敗添付の警告トーストが表示される
- 編集モードの挙動は変更なし（即時アップロード継続）
- AttachmentArea にモード別の案内文（新規: 「保存後にアップロード」/ 編集: 「即時保存」）が表示される
- report-detail.md §6 §7 に新規モードのローカル保持方式が明記されている
- 既存テスト（`AttachmentArea.integration.test.tsx` 等）が通過する
- 新規モード用のテスト（ローカル保持・順次アップロード・部分失敗ハンドリング）が追加されている

## 関連

- 108: フォーム編集中の操作整合性 — 本 issue の発見起点。アップロード中の操作制御方針と整合する必要あり
- 114: 添付の永続化タイミング仕様明示 — 編集モードの挙動定義。本 issue は新規モードのみ別方針
- 100: AttachmentUploader UI 改善 — 解決済み

## 次セッション着手手順（別セッションでワークツリー対応想定）

1. `/issue 対応 115` で本 issue を読み込む
2. 上流確認: `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6 §7、`expense-saas/frontend/src/pages/reports/{AttachmentArea,AttachmentList,ItemForm,ItemSlidePanel}.tsx`、`expense-saas/openapi.yaml`（attachments API 形式の確認）
3. 設計書追記方針を提示 → ユーザー合意
4. designer で設計書追記（report-detail.md §6 §7） → 内部レビュー
5. `/implement issue-115` で frontend-developer をワークツリー起動（ブランチ名例: `feat/115-new-item-attachment-local-buffer`）
6. PR 作成 → /test → reviewer → codex → マージ
7. 設計書追記は dev-journal リポジトリで独立コミット

並列化: 本 issue は AttachmentArea/ItemForm を全面的に触るため、issue 108（並行操作 + 破棄ダイアログ）と**ファイル重複あり**。原則的には 108 → 115、または 108 と 115 を同一 PR にまとめる。issue 114（設計書 + 案内文）は先に独立対応可能。

## 検討事項（実装時に詰める）

- **保留中添付の削除**: ローカル state に保留中のファイルを「× 削除」できる UI が必要（保存前なので S3 / DB は呼ばない）
- **「保存」処理中の進捗表示**: 順次アップロード中は「アップロード中... (N/M 件完了)」のような表示
- **「保存」処理のキャンセル**: 順次アップロード中にユーザーが中断したい場合の挙動（issue 108 の AbortController 方針と整合）
- **メモリ上限**: 5MB × N 件のファイル保持。実用上問題ないが、極端に多い場合（10 件以上等）の警告

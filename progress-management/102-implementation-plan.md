# issue 102 添付プレビュー機能 — BE/FE 実装計画

## Context

- issue 102（添付プレビュー機能）の設計書修正（D1〜D11）は `f8faa83` でコミット済み
- BE 実装（I1〜I6）+ BE テスト（T1, T2, T6）、FE 実装（I7〜I10）+ FE テスト（T3〜T5）が未着手
- issue 102 L152-206 の合意どおり「統合ブランチ + stacked PR」方式を採用する
- `expense-saas/` は master（`f98b8be`）で clean、origin と同期済み
- 設計書側の最終形（検証済み）:
  - `openapi.yaml` L1265-1370: `getAttachmentDownload` は `/download` パス、`getAttachmentPreview` 新設（どちらも `AttachmentAccess` を返す）
  - `openapi.yaml` L2241-2263: `AttachmentAccess` スキーマ（`url`, `file_name`, `mime_type`, `file_size`, `expires_at`）
  - `files.md` §4.1-4.5: 用途別 Content-Disposition、クリック同期 `window.open` パターン
  - `state-management.md` L127-129, L190-198, L294-295: `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl`、`enabled: false` + 明示的 `refetch()` 方式
  - `55_ui_component/screens/report-detail.md` L523-541: `onPreview` / `onDownload` props、AttachmentArea で orchestration
- 既存実装の乖離ポイント:
  - BE: ルートが `/attachments/:attId`（末尾に `/download` なし）、DTO は `AttachmentDownload` / `DownloadURL`、`StorageClient.PresignGetObject` は `disposition` 引数なし
  - FE: `useAttachmentDownload` は初期 fetch 有効、`AttachmentArea.handleDownload` は `<a download>` 方式、`AttachmentList` は ↓ アイコン・`onPreview` 未実装

## タスク実行フェーズの全体像

| フェーズ | ブランチ | ベース | 担当 | 前提 |
|---------|---------|-------|------|------|
| 0. Setup | `integration/102-attachment-preview` | master | architect（指揮役） | master が最新であること |
| 1. BE 実装 | `step11/102-be-preview` | integration/102-attachment-preview | backend-developer（worktree） | Setup 完了 |
| 2. FE 実装 | `step11/102-fe-preview` | integration/102-attachment-preview | frontend-developer（worktree） | BE の PR #Y マージ済み（ルートが `/download` に変わり、FE の型整合が成立する） |
| 3. 結合動作確認 | integration/102-attachment-preview | - | architect（指揮役） | PR #Y, #Z マージ済み |
| 4. 最終 PR | master（マージ先） | integration/102-attachment-preview | architect（指揮役） | 結合 PASS |

---

## 0. Setup 手順（architect 実施）

expense-saas リポジトリで実行する（統合ブランチは `expense-saas` 側のブランチ。dev-journal 側の設計書は `f8faa83` で master に merge 済みのため integration ブランチは不要）。

```bash
cd /root-project/expense-saas
git checkout master
git pull origin master
# 最新の master が f98b8be 以上であることを確認
git log --oneline -1

# 統合ブランチ作成
git checkout -b integration/102-attachment-preview
git push -u origin integration/102-attachment-preview
```

**完了条件**: origin に `integration/102-attachment-preview` ブランチが存在すること。

---

## 1. PR #Y: BE 実装 + テスト

### 1.1 ブランチ

- ブランチ名: `step11/102-be-preview`
- ベース: `integration/102-attachment-preview`
- 担当: backend-developer（**worktree 必須**）
- PR タイトル: `102: 添付プレビューエンドポイント追加 + AttachmentAccess スキーマ移行（BE）`

### 1.2 タスク分解と実装順序

実装は以下の順序で進める（依存関係順）。同一 PR 内で 3 コミットに分割することを推奨。

#### コミット 1: DTO / interface の刷新（I5, I6）

**ファイル**（worktree 内の相対パスで指示）:
- `internal/service/dto.go` L70-78: `AttachmentDownload` を `AttachmentAccess` にリネーム、`DownloadURL` → `URL`
- `internal/service/interfaces.go` L13-23: `StorageClient.PresignGetObject` の第 5 引数に `disposition string` を追加
- `internal/service/interfaces.go` L71-81: `AttachmentService` に `GetAttachmentPreview` メソッドを追加、`GetAttachmentDownload` の戻り値型を `*AttachmentAccess` に変更

**完了条件**:
- `*AttachmentAccess` を返す interface になっている
- `StorageClient` の `PresignGetObject` シグネチャが `(ctx, key, fileName, mimeType, disposition, expiry)` の順に変更されている
- `go build ./...` PASS
- DTO コメントが日本語で、`download_url` → `url` の JSON タグが更新されている

#### コミット 2: service / StorageClient / handler / ルーティングの実装（I1, I2, I3, I4）

**ファイル**:
- `internal/pkg/s3/*.go`: `StorageClient.PresignGetObject` の実装（AWS SDK 版 + `InMemoryClient` モック）を `disposition` 引数対応に更新
  - 実 S3 版: `ResponseContentDisposition` を `disposition` でそのまま指定（`attachment; filename="..."` または `inline; filename="..."` を service 層で組み立てる）
  - InMemory 版: 引数受け取りのみでよいが URL にクエリとして含めてテストで検証しやすくする
- `internal/service/attachment_service.go`:
  - 既存 `GetAttachmentDownload` の `PresignGetObject` 呼び出しを `disposition = "attachment; filename=\"<file_name>\""` に更新
  - 新規 `GetAttachmentPreview` を追加（`GetAttachmentDownload` と同じ認可フロー + `disposition = "inline; filename=\"<file_name>\""`）
  - 共通化: private 関数 `generatePresignedURL(ctx, actor, reportID, itemID, attachmentID, disposition)` を切り出して 2 つの public メソッドから呼ぶ
  - `files.md` §7.1 準拠の `file_name` エンコード（RFC 5987）を利用する場合は既存実装を踏襲
- `internal/handler/attachment.go`:
  - L34-42 `attachmentDownloadResponse` を `attachmentAccessResponse`（`URL string json:"url"`）にリネーム
  - `toAttachmentDownloadResponse` を `toAttachmentAccessResponse` にリネーム
  - `GetAttachmentDownload` は維持（ルーターで `/download` パスにマッピング）
  - 新規 `GetAttachmentPreview` を追加（handler のコードは `GetAttachmentDownload` とほぼ同じ、service 呼び出しだけ変える）
- `cmd/server/main.go` L210: `/attachments/{attId}` を `/attachments/{attId}/download` に変更、`/attachments/{attId}/preview` を追加
- `internal/testutil/http.go` L123: 同上の変更を適用

**完了条件**:
- `go build ./...` PASS
- `go vet ./...` PASS
- ルーター順序で DELETE `/attachments/{attId}` が `/download` / `/preview` GET より後に来ていて chi の衝突がない（実際には `/attachments/{attId}` は DELETE、`/attachments/{attId}/download` は GET なので衝突しないことを確認）
- 既存の `attachment_handler_test.go` が一時的に失敗しても良い（コミット 3 で更新する）

#### コミット 3: テスト実装と既存テストの更新（T1, T2, T6）

**ファイル**:
- `internal/handler/attachment_handler_test.go`:
  - 既存 `ATT-030`〜`ATT-041` 系: URL を `/attachments/:attId/download` に更新、レスポンスボディのキーを `data.url` / `data.file_name` / `data.mime_type` / `data.file_size` / `data.expires_at` に更新（`download_url` → `url`）、traceability コメント更新
  - 新規 `ATT-055`〜`ATT-060` を追加（test_cases/attachments.md §3-5）:
    - `TestGetAttachmentPreview_Success_Owner`（ATT-055）
    - `TestGetAttachmentPreview_Success_Admin`（ATT-056）
    - `TestGetAttachmentPreview_Success_Accounting`（ATT-057）
    - `TestGetAttachmentPreview_Success_Approver_SubmittedReport`（ATT-058）
    - `TestGetAttachmentPreview_Forbidden_Member_OtherOwner`（ATT-059）
    - `TestGetAttachmentPreview_AttachmentNotFound`（ATT-060）
  - T6: 越境テスト（CRS-010b）を追加 — `TestTenantIsolation_GetAttachmentPreview_OtherTenant_404`。CRS-010 の既存ケースが存在すればそれと同じ構造で実装
  - 各テストで `InMemoryClient` が受け取った `disposition` を確認できるように、モックを `ResponseContentDisposition` を保持・検証可能にする（InMemoryClient に「直近 presign 呼び出しの引数を記録する」機能を追加しても良い）
- `internal/service/attachment_service_test.go`（**新設**、T2）:
  - `TestAttachmentService_GetAttachmentDownload_PresignDisposition`: mock storage の `PresignGetObject` に渡される `disposition` が `attachment; filename="<file_name>"` で始まること
  - `TestAttachmentService_GetAttachmentPreview_PresignDisposition`: `disposition` が `inline; filename="<file_name>"` で始まること
  - `TestAttachmentService_GetAttachmentPreview_AuthzForbidden_NotCallPresign`: 認可失敗時 `PresignGetObject` が呼ばれないこと（ATT-011 / ATT-038 のプレビュー版）
  - `TestAttachmentService_*_NotFound`: 添付不在で 404（ErrResourceNotFound）になること

**完了条件**:
- `go test -tags=integration ./internal/handler/... -run 'TestGetAttachment(Download|Preview)|TestTenantIsolation_GetAttachmentPreview' -count=1` PASS
- `go test ./internal/service/... -run TestAttachmentService -count=1` PASS
- 既存 ATT-001〜054 の他のテスト（upload/list/delete 系）を壊していない
- ローカルでの full CI は **回さない**（メモリ `feedback_no_local_test_run`）。lint / build と上記ピンポイントテストまで

### 1.3 backend-developer への指示プロンプト案

```
issue 102 の BE 実装 + テストを担当してください。

## 背景
- 統合ブランチ方式の sub PR 1/2 本目（BE）
- 設計書修正（D1〜D11）は master に merge 済み（f8faa83）

## 作業環境
- worktree で作業してください（実装共通ルール: /root-project/.claude/rules/implementation-workflow.md）
- ブランチ名: `step11/102-be-preview`
- ベース: `integration/102-attachment-preview`（master ではない）
  - worktree 起動後、最初に `git fetch origin && git checkout integration/102-attachment-preview && git pull && git checkout -b step11/102-be-preview` を実行
  - すでに worktree の自動ブランチがある場合は `git branch -m step11/102-be-preview`
- dev-journal は読み取り専用（/root-project/dev-journal/... の絶対パスで参照）

## 必読資料
- /root-project/dev-journal/issues/open/102-attachment-preview-missing.md L23-218
- /root-project/dev-journal/deliverables/docs/50_detail_design/openapi.yaml L1265-1370, L2241-2263
- /root-project/dev-journal/deliverables/docs/50_detail_design/files.md §4
- /root-project/dev-journal/deliverables/docs/60_test/test_cases/attachments.md §3, §3-5
- /root-project/dev-journal/deliverables/docs/60_test/test_cases/cross-cutting.md CRS-010 / CRS-010b
- /root-project/dev-journal/progress-management/102-implementation-plan.md §1（本計画書）

## 成果物
§1.2 のコミット 1〜3 を順番に実装してください。各コミットの完了条件は §1.2 に記載。

## 品質ゲート（self-check）
- `go build ./...` PASS
- `go vet ./...` PASS
- `golangci-lint run ./...` PASS
- 以下のピンポイントテスト PASS:
  - `go test -tags=integration ./internal/handler/... -run 'TestGetAttachment(Download|Preview)|TestTenantIsolation_GetAttachmentPreview' -count=1`
  - `go test ./internal/service/... -run TestAttachmentService -count=1`
- CI フルスイートはローカルで回さない（CI に任せる）

## 注意
- API 互換は破壊変更（`download_url` → `url`、URL パスに `/download` を追加）。integration ブランチで FE と揃えるため master は守られる
- 日本語コメントを維持（godoc も日本語）
- PR タイトル: `102: 添付プレビューエンドポイント追加 + AttachmentAccess スキーマ移行（BE）`
- PR 本文に絵文字フッターは含めない（memory: feedback_no_emoji_in_pr）
- PR 作成後、PR URL を返してください
```

### 1.4 受け入れ判定基準（PR #Y レビュー時）

- [ ] `cmd/server/main.go` と `internal/testutil/http.go` のルーティングが `openapi.yaml` L1265 / L1320 と一致
- [ ] `service/dto.go` の `AttachmentAccess` が `openapi.yaml` L2241-2263 のフィールド名・JSON タグと一致（`url`, `file_name`, `mime_type`, `file_size`, `expires_at`）
- [ ] `StorageClient.PresignGetObject` が `disposition` を受け取り、AWS SDK 版で `ResponseContentDisposition` に設定されている
- [ ] `GetAttachmentPreview` の認可ロジックが `GetAttachmentDownload` と同一（`CanViewReport`）
- [ ] `ATT-055〜060` のテスト 6 件と `CRS-010b` が新規追加され PASS
- [ ] 既存 `ATT-030〜041` のテストが `/download` URL + `data.url` 前提に更新され PASS
- [ ] handler test で「認可失敗時に `PresignGetObject` が呼ばれない」（`ATT-038` の preview 版）を mock で検証
- [ ] `golangci-lint run ./...` / `go vet ./...` PASS
- [ ] PR 本文に絵文字なし、変更ファイル一覧と影響範囲の記載あり

---

## 2. PR #Z: FE 実装 + テスト

### 2.1 前提条件

- **PR #Y が integration ブランチにマージ済み**であること（BE の URL / DTO が新仕様になっている）
- integration ブランチを最新化してから作業開始

### 2.2 ブランチ

- ブランチ名: `step11/102-fe-preview`
- ベース: `integration/102-attachment-preview`
- 担当: frontend-developer（**worktree 必須**）
- PR タイトル: `102: 添付プレビュー UI 追加 + useAttachment{Download,Preview}Url 分割（FE）`

### 2.3 タスク分解と実装順序

同一 PR 内で 3 コミットに分割することを推奨。

#### コミット 1: 型定義と hook の刷新（I10, I7）

**ファイル**（worktree 内）:
- `frontend/src/api/types.ts` L113-119:
  - `AttachmentDownload` を `AttachmentAccess` にリネーム
  - `download_url` → `url`
  - `AttachmentDownload` の名前が他で利用されていないか確認し、残っていれば全部 `AttachmentAccess` に置換
- `frontend/src/hooks/useAttachmentDownload.ts` を `useAttachmentDownloadUrl.ts` にリネーム（内部実装も以下に変更）:
  - クエリキー: `['reports', reportId, 'items', itemId, 'attachments', attId, 'download']`
  - URL: `/api/reports/${reportId}/items/${itemId}/attachments/${attId}/download`
  - **`enabled: false` + 明示的 `refetch()` 方式**（state-management.md L193）
  - レスポンス型: `ApiResponse<AttachmentAccess>`
- `frontend/src/hooks/useAttachmentPreviewUrl.ts`（**新設**）:
  - クエリキー: `['reports', reportId, 'items', itemId, 'attachments', attId, 'preview']`
  - URL: `/api/reports/${reportId}/items/${itemId}/attachments/${attId}/preview`
  - `enabled: false` + 明示的 `refetch()` 方式
- `frontend/src/hooks/useAttachments.ts` 等の export が `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` を再 export している場合は更新（要 grep 確認）

**注意**:
- **`enabled: false` + `refetch()` の作法を他 hook と揃える**: 既存プロジェクト内に同種のパターン（クリック時のみ fetch する hook）が他にあるか grep 確認（`enabled: false` + `refetch` の利用箇所）。無ければこの hook が先例となる
- `useAttachmentDownload` を参照している箇所を grep で洗い出し、全置換（本計画書 §6 参照: `AttachmentArea.tsx`、`useAttachmentDownload.test.tsx` の 2 箇所）

**完了条件**:
- `npm run lint` PASS
- `npx tsc --noEmit` PASS
- `useAttachmentDownload` の import が残っていないこと（`rg 'useAttachmentDownload[^U]'` で 0 件）

#### コミット 2: UI コンポーネント更新（I8, I9）

**ファイル**:
- `frontend/src/pages/reports/AttachmentList.tsx`:
  - props に `onPreview: (attId: string) => void` を追加（`onDownload` は維持）
  - ファイル名 Button の `onClick` を `onPreview(att.id)` に変更（`data-testid` は `attachment-preview-${att.id}` に変更）
  - ファイル名 Button の直後に ↓ アイコンボタン（`IconButton`、aria-label="ダウンロード"）を追加（`data-testid="attachment-download-${att.id}"`）
  - ↓ アイコンの `onClick` は `onDownload(att.id)`
  - 削除ボタンの `data-testid="attachment-delete-${att.id}"` は維持
- `frontend/src/pages/reports/AttachmentArea.tsx`:
  - `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` を hook 呼び出し時点では**インスタンス化せず**（params が固定しないため）、**クリック時に `fetchQuery` で取得**するか、または `AttachmentItem` サブコンポーネントに分離して per-item で hook を使う
  - 推奨: `AttachmentArea` 内で直接 `api.get` を呼ぶのではなく、**クリックハンドラ内で `queryClient.fetchQuery` または直接 `api.get` を呼び、`window.open('about:blank')` を先に実行する**
  - 具体的な handlePreview 実装（issue 102 L66-82 に準拠）:
    ```ts
    const handlePreview = (attId: string) => {
      const newWindow = window.open('about:blank', '_blank');
      if (!newWindow) {
        showToast('error', 'ポップアップがブロックされました。ブラウザ設定を確認してください');
        return;
      }
      api.get<ApiResponse<AttachmentAccess>>(
        `/api/reports/${reportId}/items/${itemId}/attachments/${attId}/preview`,
      )
        .then(res => { newWindow.location.href = res.data.url; })
        .catch(() => {
          newWindow.close();
          showToast('error', 'プレビューの取得に失敗しました');
        });
    };
    ```
  - `handleDownload` も同様のパターン（URL は `/download`、エラーメッセージは「ダウンロードの取得に失敗しました」）
  - 既存の `<a download>` 方式（AttachmentArea.tsx L63-78）を完全削除
  - `AttachmentList` に `onPreview={handlePreview}` と `onDownload={handleDownload}` を渡す
- **hook 分割方針の最終判断**:
  - `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` を `enabled: false` で定義する設計書指示（state-management.md L193, L198）を守るため、`AttachmentArea` では直接 `api.get` ではなく、**`useAttachmentPreviewUrl({...})` を attId 可変にできる形で呼び出す**か、**hook 内で `queryFn` のみ export して `queryClient.fetchQuery` で呼び出す**かのどちらか
  - 推奨案（2 hook + `refetch`）: `AttachmentList` の各行ごとに `useAttachmentPreviewUrl({reportId, itemId, attId})` をインスタンス化するのは hooks rule 的に不適切。よって **`AttachmentArea` でクリック時に hook の関数本体（queryFn）だけを切り出して使う**方針が無難
  - 最終判断は frontend-developer に委譲（上記 2 案のどちらを採用しても `enabled: false` + 明示的呼び出しの原則を守れば良い）。ただし**hook の中身（URL、レスポンス型、queryKey）は必ず設計書通りに定義する**こと
  - ※ もし hook の `refetch()` をクリック時に呼ぶ方式にするなら、attId を state で持ち、クリック時に setState → `useEffect` で refetch、のような複雑な制御が必要になる。その場合は AttachmentArea 内に `AttachmentItem` 的な sub component を切って hook を per-item で持つのが素直。

**完了条件**:
- UI 上でファイル名クリック = プレビュー、↓ アイコンクリック = ダウンロードとなっている
- クリック同期で `window.open('about:blank')` が呼ばれている（非同期 fetch 前に）
- 失敗時に `newWindow.close()` + エラートーストが表示される

#### コミット 3: テスト実装と更新（T3, T4, T5）

**ファイル**:
- `frontend/src/hooks/__tests__/useAttachmentDownload.test.tsx` を `useAttachmentDownloadUrl.test.tsx` にリネーム:
  - URL 検証を `/attachments/att-001/download` に更新
  - レスポンスプロパティを `data.url` に更新
  - `enabled: false` 動作の検証を追加（初期レンダリング時に fetch が呼ばれないこと、`refetch()` 後のみ呼ばれること）
- `frontend/src/hooks/__tests__/useAttachmentPreviewUrl.test.tsx`（**新設**、T3）:
  - `useAttachmentDownloadUrl.test.tsx` と同じ構造で URL を `/preview` に、test ID を ATT-FE-049 周辺で分割
- `frontend/src/pages/reports/__tests__/AttachmentList.test.tsx`（T4）:
  - `onPreview` コールバックがファイル名クリックで発火する（`data-testid="attachment-preview-${id}"`）
  - `onDownload` コールバックが ↓ アイコンクリックで発火する（`data-testid="attachment-download-${id}"`）
  - **`window.open` の検証は含めない**（presentational の責務外）
- `frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx`（T5、ATT-FE-047/048/049 周辺）:
  - ATT-FE-049（既存）: ↓ アイコンクリック → `window.open('about:blank')` 呼び出し → `/attachments/.../download` API 呼び出し → `newWindow.location.href` が署名付き URL になる
  - 新規 ATT-FE-049b（プレビュー版）: ファイル名クリック → `window.open('about:blank')` → `/attachments/.../preview` API 呼び出し → `newWindow.location.href` 差し替え
  - 新規 ATT-FE-049c: ポップアップブロック時（`window.open` が null を返す）にエラートースト表示
  - 新規 ATT-FE-049d: API エラー時に `newWindow.close()` 呼び出し + エラートースト表示
  - `window.open` をモックする（`vi.spyOn(window, 'open').mockReturnValue({ location: { href: '' }, close: vi.fn() } as any)`）

**完了条件**:
- `npm run lint` PASS
- `npx tsc --noEmit` PASS
- `npm test -- --run src/hooks/__tests__/useAttachment{DownloadUrl,PreviewUrl}.test.tsx src/pages/reports/__tests__/Attachment{List,Area.integration}.test.tsx` PASS
- `npm run build` PASS
- フルスイート `npm test` は CI に任せる（ローカルでは回さない）

### 2.4 frontend-developer への指示プロンプト案

```
issue 102 の FE 実装 + テストを担当してください。

## 背景
- 統合ブランチ方式の sub PR 2/2 本目（FE）
- BE（PR #Y）は既にマージ済みで、API は `/download` / `/preview`、レスポンスは `AttachmentAccess` になっている

## 作業環境
- worktree で作業してください（/root-project/.claude/rules/implementation-workflow.md）
- ブランチ名: `step11/102-fe-preview`
- ベース: `integration/102-attachment-preview`
  - worktree 起動後、最初に `git fetch origin && git checkout integration/102-attachment-preview && git pull && git checkout -b step11/102-fe-preview`
- dev-journal は読み取り専用

## 必読資料
- /root-project/dev-journal/issues/open/102-attachment-preview-missing.md L23-218
- /root-project/dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md §7
- /root-project/dev-journal/deliverables/docs/50_detail_design/files.md §4.5
- /root-project/dev-journal/deliverables/docs/55_ui_component/state-management.md L127-129, L190-198, L294-295
- /root-project/dev-journal/deliverables/docs/55_ui_component/screens/report-detail.md §AttachmentList props, §データフロー L663-678
- /root-project/dev-journal/deliverables/docs/60_test/test_cases/attachments.md ATT-FE-049 周辺
- /root-project/dev-journal/progress-management/102-implementation-plan.md §2（本計画書）

## 成果物
§2.3 のコミット 1〜3 を順番に実装してください。各コミットの完了条件は §2.3 に記載。

## 品質ゲート（self-check）
- `npm run lint` PASS
- `npx tsc --noEmit` PASS
- `npm run build` PASS
- 以下のピンポイントテスト PASS:
  - `npm test -- --run src/hooks/__tests__/useAttachmentDownloadUrl.test.tsx`
  - `npm test -- --run src/hooks/__tests__/useAttachmentPreviewUrl.test.tsx`
  - `npm test -- --run src/pages/reports/__tests__/AttachmentList.test.tsx`
  - `npm test -- --run src/pages/reports/__tests__/AttachmentArea.integration.test.tsx`
- フルスイート `npm test` は CI に任せる

## 実装上の要注意
- クリック同期 `window.open('about:blank')` → 非同期 fetch → `location.href` 差し替え のパターンを厳守（issue 102 L62-82）
- 失敗時は `newWindow.close()` を忘れずに
- `useAttachment{Download,Preview}Url` は `enabled: false` + 明示的 `refetch()` 方式（state-management.md L193）
- AttachmentList は presentational（`window.open` 呼び出しを含めない）、AttachmentArea が orchestration
- 既存の `useAttachmentDownload` import 箇所を grep で洗い出し全て `useAttachmentDownloadUrl` に置換
- 日本語コメント維持
- PR タイトル: `102: 添付プレビュー UI 追加 + useAttachment{Download,Preview}Url 分割（FE）`
- PR 本文に絵文字フッターなし
- PR 作成後、PR URL を返してください
```

### 2.5 受け入れ判定基準（PR #Z レビュー時）

- [ ] `types.ts` の `AttachmentAccess` が BE の `AttachmentAccess` スキーマと一致（フィールド名・snake_case）
- [ ] `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` が `enabled: false` + `refetch()` 方式で定義されている
- [ ] `AttachmentList` に ↓ アイコンボタンが追加され、`onPreview` / `onDownload` コールバックが props で受け取られている（`onPreview` と `onDownload` で呼び分け）
- [ ] `AttachmentArea` が `window.open('about:blank', '_blank')` を**クリック同期で**呼んでいる（非同期 fetch より前）
- [ ] 失敗時に `newWindow.close()` + エラートーストが表示される
- [ ] `useAttachmentDownload` の import・参照が 0 件（grep で確認）
- [ ] 既存テスト ATT-FE-049 が新仕様で PASS、新規 ATT-FE-049b〜d（もしくは同等テスト）が追加されて PASS
- [ ] `AttachmentList.test.tsx` が `onPreview` / `onDownload` の呼び分けを検証（`window.open` は検証しない）
- [ ] `npm run build` PASS
- [ ] PR 本文に絵文字なし

---

## 3. 結合動作確認（PR #Y, #Z マージ後、integration ブランチで）

### 3.1 手順

architect（指揮役）が実施。

```bash
cd /root-project/expense-saas
git checkout integration/102-attachment-preview
git pull origin integration/102-attachment-preview

# docker compose で BE + FE + MinIO を起動
docker compose up -d

# 初期データ投入（マイグレーション + seed）
# プロジェクトの慣例に従う
```

### 3.2 確認項目

- [ ] SMK-037（新仕様）:
  - ファイル名クリック → 新タブが開き、画像/PDF がインライン表示される（JPEG / PNG / PDF の 3 種類）
  - ↓ アイコン押下 → 新タブが開き、ブラウザダウンロードが開始される
- [ ] 認可マトリクス（4 ロール × 2 操作 = 8 ケース）: `authz.md` §10 の添付閲覧ルールどおり:
  - Member: 自分のレポートの添付のみ preview/download 可、他の Member のレポートは 403
  - Approver: submitted レポート + 自分が承認/却下したレポート + 自分のレポートのみ可
  - Accounting: 同一テナント全レポート可
  - Admin: 同一テナント全レポート可
- [ ] テナント越境: 他テナントの attId でリクエスト → 404（preview / download 両方）
- [ ] 署名付き URL の `expires_at` が 15 分後
- [ ] ポップアップブロック回避: ブラウザのポップアップブロックを有効にした状態でもプレビュー・ダウンロードが動作する
- [ ] CI（integration ブランチ push 時）が PASS

### 3.3 失敗時の対応

- BE 側の問題: `step11/102-be-preview` を再開して fix PR を integration に向けて出す
- FE 側の問題: `step11/102-fe-preview` を再開して fix PR を integration に向けて出す
- 設計書の欠落・矛盾が見つかった場合: issue 102 に追記して合意形成 → 必要なら D12 相当の設計修正 PR を integration に出す

---

## 4. 最終 PR: integration/102-attachment-preview → master

### 4.1 手順

```bash
# 結合動作確認 PASS 後
cd /root-project/expense-saas
git checkout integration/102-attachment-preview
git pull

# master との差分を確認
git log master..integration/102-attachment-preview --oneline

# PR 作成（squash merge 推奨）
gh pr create \
  --base master \
  --head integration/102-attachment-preview \
  --title "102: 添付プレビュー機能（BE + FE 結合）" \
  --body "..."
```

### 4.2 PR 本文に含める項目

- issue 102 への参照
- sub PR（#Y, #Z）へのリンク
- 結合動作確認のサマリー（SMK-037 PASS、認可 8 ケース PASS）
- breaking change の明記（`download_url` → `url`、URL パスに `/download` 追加）— ただし master には統合された形で投入されるので外部向けには非破壊

### 4.3 マージ方式

**squash merge** を採用（issue 102 L163 合意）。統合ブランチ上の複数コミットを 1 つに圧縮して master に載せる。

### 4.4 マージ後のクリーンアップ

```bash
# integration ブランチと sub PR ブランチを削除
git push origin --delete integration/102-attachment-preview
git push origin --delete step11/102-be-preview
git push origin --delete step11/102-fe-preview

# issue 102 を closed に移動
mv dev-journal/issues/open/102-attachment-preview-missing.md dev-journal/issues/closed/
```

---

## 5. リスク・注意点

### 5.1 API 互換破壊の影響範囲

- **`download_url` → `url`**: FE で `AttachmentDownload.download_url` を参照している箇所を洗い出し済み（§6 参照）。FE 側の PR #Z で全置換する
- **URL パスに `/download` 追加**: FE のみが呼び出し元（CLI / 外部連携なし）。FE 側の PR #Z で URL を更新
- integration ブランチ内で BE + FE を結合してから master に入れるため、**master のユーザー影響は 0**
- 既存の BE 単体テスト（ATT-030〜041）と統合テスト（AttachmentArea.integration.test.tsx の ATT-FE-049）は互換性のために必ず更新する（旧 URL / 旧プロパティ名を残さない）

### 5.2 ルーティング変更が既存コード・テストに与える影響

**BE 側の影響を受ける箇所**（grep 済み）:
- `cmd/server/main.go` L210
- `internal/testutil/http.go` L123
- `internal/handler/attachment_handler_test.go` ATT-030〜041（URL 使用箇所）
- `internal/service/dto.go` L72 `AttachmentDownload` 型（型名を外部参照しているコードがあるか要 grep 確認: `grep -rn "AttachmentDownload" internal/`）

**FE 側の影響を受ける箇所**（grep 済み、§6 参照）:
- `frontend/src/api/types.ts` L113 `AttachmentDownload` 型定義
- `frontend/src/hooks/useAttachmentDownload.ts` 全体
- `frontend/src/hooks/__tests__/useAttachmentDownload.test.tsx` 全体
- `frontend/src/pages/reports/AttachmentArea.tsx` L64-78
- `frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx` L324-392

**いずれも PR #Y または PR #Z で同一コミット内で更新**。integration ブランチ上で BE + FE が揃うまで master にマージしない。

### 5.3 フロント hook の作法整合性

- `enabled: false` + 明示的 `refetch()` の作法を他 hook と揃えるため、frontend-developer に **実装開始前に `rg 'enabled: false' frontend/src/hooks/`** で既存パターンを確認するよう指示する
- 既存 hook に `enabled: false` パターンがなければ、本 PR が先例となる（設計書 state-management.md の規定に従う）
- `useAttachmentPreviewUrl` / `useAttachmentDownloadUrl` の 2 hook は完全に mirror 構造にして、命名・シグネチャ・テストパターンを揃える（将来の hook 分割のリファレンスとして）

### 5.4 既存 `useAttachmentDownload` を参照しているコンポーネント

grep 結果（本計画書作成時点）:

| # | ファイル | 参照形式 | 対応 |
|---|---------|---------|------|
| 1 | `frontend/src/hooks/useAttachmentDownload.ts` | 定義元 | リネーム（PR #Z コミット 1） |
| 2 | `frontend/src/hooks/__tests__/useAttachmentDownload.test.tsx` | import + describe | リネーム + URL/プロパティ更新（PR #Z コミット 3） |
| 3 | `frontend/src/pages/reports/AttachmentArea.tsx` | 直接 `api.get` 経由で呼び出し（hook 未使用） | クリック同期 `window.open` + fetch パターンに書き換え（PR #Z コミット 2） |
| 4 | `frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx` | fetch モックのレスポンス shape `download_url` を使用 | `url` に更新 + window.open モック検証追加（PR #Z コミット 3） |

**`useAttachmentDownload` を import して hook として呼び出している箇所は現状 0 件**（現状 `AttachmentArea.tsx` は `api.get` 直接呼び出しになっている）。そのため「hook 名変更による呼び出し側の破壊」はないが、代わりに **「AttachmentArea の実装パターンが仕様（hook 経由）と乖離している」点の是正**が PR #Z の中心となる。

### 5.5 InMemoryClient（テスト用 S3 モック）の後方互換

- `StorageClient.PresignGetObject` に `disposition` 引数を追加するため、**InMemoryClient（`internal/pkg/s3/memory.go` 等）も同時に更新必須**
- 他のテストが InMemoryClient の旧シグネチャに依存していないか grep（`PresignGetObject\(` で検索）
- BE コミット 1 の段階で interface を変更するため、この段階で build が通らなくなる箇所を全て洗い出し → コミット 2 でまとめて修正

### 5.6 統合ブランチの維持

- BE の PR #Y レビュー中に master が進むケース: integration ブランチで `git merge master` して追従（issue 102 L204）
- sub PR のマージ順序は **BE → FE 固定**（逆にすると FE テストが BE の新仕様を前提にして壊れる）
- 万一 integration ブランチで結合後に追加修正が必要になった場合、別途小 PR を integration に向けて出す（master には向けない）

### 5.7 コード編集ルールの徹底

- サブエージェントには worktree 必須を明示（/root-project/expense-saas/ への絶対パス直接アクセス禁止）
- 日本語コメント維持（godoc / JSDoc）
- ローカルでフルスイートを回さない（memory: feedback_no_local_test_run）。ピンポイントテスト + CI 任せ
- PR 本文に絵文字フッターを含めない（memory: feedback_no_emoji_in_pr）
- 重大問題（ブランチ消失等）はサブエージェントから指揮役にエスカレーション（memory: feedback_escalate_on_major_issues）

---

## 6. 既存 `useAttachmentDownload` 参照の最終洗い出し（grep 結果）

PR #Z の作業着手前に frontend-developer が再確認する必要がある。本計画書作成時点の grep 結果:

```
expense-saas/frontend/src/hooks/useAttachmentDownload.ts
expense-saas/frontend/src/hooks/__tests__/useAttachmentDownload.test.tsx
expense-saas/frontend/src/api/types.ts              (AttachmentDownload 型)
expense-saas/frontend/src/pages/reports/AttachmentArea.tsx
expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx
expense-saas/frontend/src/pages/reports/__tests__/AttachmentList.test.tsx  (onDownload 参照)
expense-saas/frontend/src/pages/reports/AttachmentList.tsx                 (onDownload 参照)
```

BE 側で `AttachmentDownload` 型の参照箇所:

```
expense-saas/internal/handler/attachment.go                 (attachmentDownloadResponse, toAttachmentDownloadResponse, GetAttachmentDownload)
expense-saas/internal/handler/attachment_handler_test.go    (TestGetAttachmentDownload_*, traceability コメント)
expense-saas/internal/service/attachment_service.go         (AttachmentDownload 戻り値型)
expense-saas/internal/service/dto.go                        (type AttachmentDownload struct)
expense-saas/internal/service/interfaces.go                 (AttachmentService.GetAttachmentDownload メソッド)
```

---

## 7. 完了条件（issue 102 全体）

- [ ] integration ブランチが master に squash merge 済み
- [ ] 結合動作確認（§3.2）全項目 PASS
- [ ] `traceability.md` / `test_cases/attachments.md` / `smoke_check.md` の記述と実装が一致
- [ ] issue 102 が `open/` → `closed/` に移動
- [ ] progress.md の Step 11-A タスク状態が更新されている
- [ ] memory（`.claude/memory/`）に統合ブランチ運用の学びがあれば追記

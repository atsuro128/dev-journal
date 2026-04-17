# 添付ファイルのプレビュー機能が未実装（設計 vs 実装の乖離 + 承認者 UX 改善）

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / backend / ux / design-implementation-gap

## 影響度
中（業務 UX に直接影響。承認者が添付確認のたびにダウンロード→開く→閉じるを繰り返す必要があり、承認業務効率が悪い。設計書との乖離も含む）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-037（添付ダウンロード）を実施中、ユーザーから「承認者がいちいちダウンロードしないと添付を確認できないのは不便ではないか、ブラウザ上で見られないのか」という指摘があり、設計書を確認したところ「新しいタブで表示」が当初想定だったが実装が追従していないことが判明。

## 関連ステップ
Step 5（詳細設計）/ Step 10-G（添付ファイル機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項。ただし承認者業務の根幹に近いので Step 11-A 完了前に方針確定したい）

---

## 合意事項（2026-04-15）

論点 1〜5 はユーザーとの議論で決着済み。論点 6（設計書修正範囲）の詳細は codex レビューで監査予定。

### 決定 1: プレビュー対象ファイル形式 → **案 A（全形式対応）**
- JPEG / PNG / PDF すべて対応
- モバイルでの PDF 挙動差は論点 5 で吸収（PC 優先）

### 決定 2: UI 併存方法 → **案 A/C（ファイル名=プレビュー、↓ アイコン=ダウンロード）**
- ファイル名クリック = プレビュー（`window.open(preview_url)`）
- 横の ↓ アイコンボタン = ダウンロード（`window.open(download_url)`）
- 削除ボタンは Group A PR #57 で ConfirmDialog 実装済み（所有者のみ）
- 将来の案 D（モーダル内 iframe/img）への移行コストは小（1 箇所の書き換えで済む）

### 決定 3: Content-Disposition / API 設計 → **案 A（別エンドポイント）**
- `GET /api/reports/:id/items/:itemId/attachments/:attId/download`（既存をリネーム）
  - 署名付き URL 発行時に `ResponseContentDisposition=attachment; filename="..."`
- `GET /api/reports/:id/items/:itemId/attachments/:attId/preview`（新設）
  - 署名付き URL 発行時に `ResponseContentDisposition=inline; filename="..."`
- 両方とも同じ認可ロジックを呼ぶ（service 層に共通関数 `GeneratePresignedURL(disposition string)`）

#### レスポンススキーマ: 共通 `AttachmentAccess` へ統合

既存 `AttachmentDownload` スキーマ（`download_url`, `file_name`, `mime_type`, `file_size`, `expires_at` の 5 フィールド）を **`AttachmentAccess` にリネーム**し、`download_url` → `url` にリネーム。`/download` と `/preview` の両方が同じスキーマを返す。

```yaml
AttachmentAccess:
  type: object
  required: [url, file_name, mime_type, file_size, expires_at]
  properties:
    url:        { type: string, format: uri, description: "署名付き URL。Content-Disposition は発行エンドポイントで決まる" }
    file_name:  { type: string }
    mime_type:  { type: string }
    file_size:  { type: integer }
    expires_at: { type: string, format: date-time }
```

影響範囲: `openapi.yaml`, `files.md`, `types.ts`, `dto.go`, `test_cases/attachments.md`, `traceability.md`, `55_ui_component/screens/report-detail.md` 等、既存 5 フィールド前提の全箇所を更新する（詳細は決定 6 参照）。

#### フロント呼び出しパターン: クリック同期で空タブ open → URL 差し替え（ポップアップブロック回避）

非同期 fetch 後に `window.open(url)` を呼ぶと、クリックイベントと切り離されてブラウザのポップアップブロックに引っかかる。これを回避するため、**クリック同期で空タブを先に開き、URL 取得後に location を差し替える**パターンを採用する。

```ts
// AttachmentArea.tsx (orchestration 担当)
const handlePreview = (attId: string) => {
  // クリック同期で空タブを先に開く（ポップアップブロック回避）
  const newWindow = window.open('about:blank', '_blank');
  if (!newWindow) {
    showErrorToast('ポップアップがブロックされました。ブラウザ設定を確認してください');
    return;
  }
  // URL を非同期で取得
  fetchPreviewUrl(reportId, itemId, attId)
    .then(data => { newWindow.location.href = data.url; })
    .catch(() => {
      newWindow.close();
      showErrorToast('プレビューの取得に失敗しました');
    });
};
```

設計書（`report-detail.md` §7 / `files.md`）にもこの実装制約を明記する。

- 実際の API 呼び出しと `window.open` 呼び出しは `AttachmentArea.tsx`（orchestration 担当）に配置
- `AttachmentList.tsx`（presentational）は `onPreview` / `onDownload` コールバックを props で受け取るだけ
- フロント hook も 2 つに分割: `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl`
  - いずれも `enabled: false` + 明示的 `refetch()` 方式（初期ロード不要）

### 決定 4: 認可・セキュリティ → **案 A（既存 15 分・調整なし）**
- 認可ルール: 既存 attachment 取得 API と同じ（同一テナント + 閲覧権限）
- 署名付き URL の有効期限: download / preview とも 15 分（`S3_PRESIGNED_URL_EXPIRY=15m` のまま）
- 有効期限が切れたら再クリックで復旧。セキュリティトレードオフを避ける
- CSP / CORS: 新タブで開く（`window.open`）方式なので調整不要

### 決定 5: モバイル対応範囲 → **案 A（PC 優先・モバイル best-effort）**
- 実装は「新タブで開く」のみ。iOS Safari で PDF がダウンロードに退化してもフォールバックとして許容
- 画像（JPEG/PNG）はモバイル両方で正常動作
- smoke_check.md SMK-060〜063 のレスポンシブカバレッジ不足は **issue 104 に統合**（ロール別表示制御 + レスポンシブを 1 つの監査マトリクスで扱う）
- モバイル対応そのものは NFR-UX-001（正式要件）として維持

### 決定 6: 設計書修正範囲 → **codex 監査指摘を反映（2026-04-15）**

初版の叩き台は `dev-journal/review-findings/open/101-issue102-attachment-preview-audit.md` の指摘 3〜6 を受けて全面改訂。横断参照、テスト責務、実装ファイルパス、API 契約互換性の問題を全て反映した。

#### 設計書（修正対象）

| # | ファイル | 修正内容 |
|---|---|---|
| D1 | `deliverables/docs/50_detail_design/openapi.yaml` | 既存 `getAttachmentDownload` オペレーション（L1265 付近）を `/download` パスにリネーム。新規 `getAttachmentPreview` オペレーション追加。`AttachmentDownload` スキーマ（L2187 付近）を `AttachmentAccess` にリネームし `download_url` → `url` に変更。両オペレーションのレスポンスは `AttachmentAccess` を参照 |
| D2 | `deliverables/docs/50_detail_design/files.md` §3〜§4 | `Content-Disposition` の扱いを「ダウンロード時は `attachment`、プレビュー時は `inline`」に変更。L305 以降のレスポンス例と L328 以降の署名付き URL 設計を `url` / `AttachmentAccess` 前提で書き換え。L333 の `ResponseContentDisposition` 記述を用途別に分割。新タブ方式での「クリック同期で空タブ open → location 差し替え」パターンを実装制約として明記 |
| D3 | `deliverables/docs/50_detail_design/screens/report-detail.md` §7 | 操作方法を「ファイル名クリック = プレビュー、↓ アイコン = ダウンロード」に変更。UI モック（L266 付近）に ↓ アイコン追加。L360 の「署名付き URL を取得し、新しいタブでファイルを表示/ダウンロード」を「クリック同期で空タブを開き、非同期取得した署名付き URL で location を差し替える」と明確化 |
| D4 | `deliverables/docs/50_detail_design/authz.md` | L195 のルート定義断片を `/download` と `/preview` の 2 エンドポイントに分割。L325 の添付 API マトリクスに `/preview` を追加（4 ロール × 2 操作）。認可ルールは既存 download と同じ（同一テナント + 閲覧権限）と明記 |
| D5 | `deliverables/docs/50_detail_design/security.md` | L425-426 の CSP `img-src` と S3 アクセス方針を、preview 導入後の表現に整理。新タブ `window.open` は CSP 対象外である旨を明記 |
| D6 | `deliverables/docs/55_ui_component/state-management.md` | L127 の `useAttachmentDownload` 前提を `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` の 2 hook 分割前提に更新 |
| D7 | `deliverables/docs/55_ui_component/screens/report-detail.md` | L512 付近の添付ダウンロード前提 UI 定義を、プレビュー + ダウンロード併存前提に更新 |
| D8 | `deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-037 | プレビュー動作の期待結果を追加。「1. ファイル名テキストを押下 → クリック同期で新しいタブが開き、取得完了後に画像/PDF が表示される。2. ↓ アイコン押下 → ダウンロード開始」 |
| D9 | `deliverables/docs/60_test/test_cases/attachments.md` | L160 付近の `data.download_url` 前提を `data.url` + `AttachmentAccess` 前提に書き換え。プレビュー関連のテストケース追加（認可 4 ロール × 2 エンドポイント） |
| D10 | `deliverables/docs/60_test/test_cases/cross-cutting.md` | L147 付近の `CRS-010` `GetAttachmentDownload` 前提に、`GetAttachmentPreview` の越境ケースを追加 |
| D11 | `deliverables/docs/60_test/traceability.md` | L79 付近の `ATT-F03` 正本参照を `getAttachmentDownload` / `getAttachmentPreview` の 2 参照に更新 |

**codex レビューで `architecture.md` は現行ツリーに存在しないため修正対象から除外**。

#### テスト（新規 / 更新）

| # | 種別 | ファイル | 内容 |
|---|---|---|---|
| T1 | BE handler test | `expense-saas/internal/handler/attachment_handler_test.go`（既存、`ATT-030`〜`ATT-041` に追加）| `/download` はリネーム対応、`/preview` 系ケースを mirror で追加（4 ロール × 認可テスト、テナント越境テスト）|
| T2 | BE service test | `expense-saas/internal/service/attachment_service_test.go`（新規）| `GeneratePresignedURL(disposition string)` の分岐テスト。既存 service には unit test がないため新設 |
| T3 | FE hook test | `expense-saas/frontend/src/hooks/__tests__/useAttachmentDownloadUrl.test.tsx`（既存をリネーム）+ 新規 `useAttachmentPreviewUrl.test.tsx` | エンドポイント変更を反映、両 hook の API 呼び出しと返却型（`AttachmentAccess`）検証 |
| T4 | FE presentational test | `expense-saas/frontend/src/pages/reports/__tests__/AttachmentList.test.tsx`（既存） | ファイル名クリックで `onPreview(attId)` コールバック発火、↓ アイコンで `onDownload(attId)` コールバック発火を検証。**`window.open` 呼び出しは検証しない**（presentational の責務外）|
| T5 | FE integration test | `expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx`（既存、ATT-FE-047/048 周辺に追加）| 実際の API 呼び出しと `window.open('about:blank')` → `location.href` 差し替えパターンを検証（preview / download 両方）|
| T6 | cross-cutting BE test | Step 11-B で `cross_cutting_test.go` が新設される予定だが、**現時点では存在しないため、`attachment_handler_test.go` に越境ケースを追加**。Step 11-B 着手時に cross_cutting_test.go へ移行する |

#### 実装（新規 / 更新）

| # | 種別 | ファイル | 内容 |
|---|---|---|---|
| I1 | BE handler | `expense-saas/internal/handler/attachment.go`（既存、`attachment_handler.go` ではない）| `GetAttachmentDownload` を `/download` パスにリネーム対応、新規 `GetAttachmentPreview` メソッド追加 |
| I2 | BE service | `expense-saas/internal/service/attachment_service.go`（既存想定、要確認）| `GeneratePresignedURL(disposition string)` 共通関数追加、`GenerateDownloadURL` / `GeneratePreviewURL` の公開関数 2 つ |
| I3 | BE ルーティング | `expense-saas/cmd/server/main.go`（既存、L188 付近）| 既存 `/attachments/:attId` ルートを `/attachments/:attId/download` にリネーム、`/attachments/:attId/preview` を新設 |
| I4 | BE test 用ルーティング | `expense-saas/internal/testutil/http.go`（既存、L102 付近）| 本番ルーティングと同じ変更を適用 |
| I5 | BE DTO | `expense-saas/internal/domain/dto.go`（既存、L68 付近）| `AttachmentDownload` DTO を `AttachmentAccess` にリネーム、`DownloadURL` フィールドを `URL` にリネーム |
| I6 | BE service interface | `expense-saas/internal/service/interfaces.go`（既存、L75 付近）| service の interface を preview/download 対応に更新 |
| I7 | FE hook | `expense-saas/frontend/src/hooks/useAttachmentDownloadUrl.ts`（既存 `useAttachmentDownload.ts` をリネーム）+ 新規 `useAttachmentPreviewUrl.ts` | `enabled: false` + 明示的 `refetch()` 方式に変更 |
| I8 | FE orchestration | `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`（既存、L58 付近）| `handlePreview` と `handleDownload` を追加。クリック同期で `window.open('about:blank')` → fetch → `location.href` 差し替えパターンを実装 |
| I9 | FE presentational | `expense-saas/frontend/src/pages/reports/AttachmentList.tsx`（既存、L28 付近、L44 付近のクリックハンドラ）| `onPreview` / `onDownload` コールバック props を追加。ファイル名 onClick と ↓ IconButton を配置 |
| I10 | FE 型定義 | `expense-saas/frontend/src/api/types.ts`（既存、L96〜L104 付近）| `AttachmentDownload` 型を `AttachmentAccess` にリネーム、`download_url` → `url` に変更 |

### 実装の PR 分割案 → **統合ブランチパターン（stacked PR）**

master を中間破壊から守るため、**統合ブランチに sub PR を積む方式**を採用:

```
master
  └─ integration/102-attachment-preview       （master から分岐、他の master 更新を追従）
       ├─ PR #X: 設計書修正                    → integration ブランチへ merge
       ├─ PR #Y: BE 実装 + テスト              → integration ブランチへ merge
       └─ PR #Z: FE 実装 + テスト              → integration ブランチへ merge（BE マージ後）
  
  最後に integration/102-attachment-preview → master を単一 PR で squash merge
```

#### PR の流れ

1. **Setup**: `git checkout master && git pull && git checkout -b integration/102-attachment-preview && git push -u origin integration/102-attachment-preview`

2. **PR #X: 設計書修正**
   - ベース: `integration/102-attachment-preview`
   - ブランチ: `step11/102-design-docs`
   - 内容: D1〜D11 を 1〜2 コミットに分割
   - フロー: 設計成果物フロー（reviewer → codex → 合意）
   - ローカル CI 不要（ドキュメントのみ）

3. **PR #Y: BE 実装 + テスト**
   - ベース: `integration/102-attachment-preview`
   - ブランチ: `step11/102-be-preview`
   - 内容: I1〜I6 + T1〜T2, T6
   - フロー: PR フロー（reviewer → codex）
   - ローカル CI: `go test ./...`, `golangci-lint run ./...`
   - BE 単体で動作確認（curl で新エンドポイントを叩く）

4. **PR #Z: FE 実装 + テスト**
   - ベース: `integration/102-attachment-preview`（PR #Y マージ後）
   - ブランチ: `step11/102-fe-preview`
   - 内容: I7〜I10 + T3〜T5
   - フロー: PR フロー（reviewer → codex）
   - ローカル CI: `npm run lint`, `npx tsc --noEmit`, `npm test`, `npm run build`

5. **結合動作確認**（integration ブランチ上）
   - BE + FE が結合した状態で `docker compose up`
   - SMK-037（プレビュー / ダウンロード両方）を実行
   - 4 ロール × 2 操作の認可確認

6. **最終 PR: integration → master**
   - 結合動作確認 PASS 後に作成
   - squash merge でコミット履歴をきれいに統合
   - master が初めて preview 機能を受け入れる

#### 統合ブランチの運用注意

- integration ブランチは master の更新を定期的に merge する（他の PR がマージされた際）
- sub PR 作成時は必ず integration ブランチを最新化してから分岐
- sub PR のレビューは独立して実施できるが、マージ順序は 設計書 → BE → FE を守る

### 影響範囲の最終まとめ（改訂版）

- **設計書**: 11 ファイル（50_detail_design 5 / 55_ui_component 2 / 60_test 4）
- **BE 実装**: handler 1, service 1, router 2（main.go + testutil/http.go）, DTO 1, interfaces 1 = 6 ファイル
- **BE test**: 2 ファイル（attachment_handler_test.go 更新 + attachment_service_test.go 新設）
- **FE 実装**: hook 2（1 リネーム + 1 新設）, AttachmentArea.tsx 更新, AttachmentList.tsx 更新, types.ts 更新 = 5 ファイル
- **FE test**: 3 ファイル（hook test 2 + AttachmentList.test.tsx 更新 + AttachmentArea.integration.test.tsx 更新）
- **既存の認可ロジック / S3 設定 / env 変数 / DB schema は変更なし**
- **マイグレーション不要**
- **DTO スキーマ変更（AttachmentDownload → AttachmentAccess, download_url → url）は breaking change だが、integration ブランチで BE+FE を結合するため master は常に整合**

---

## 対応前の合意確認（必須）

UI/UX/API/署名付き URL の発行ロジックすべてに影響する課題。実装着手前に下記論点を合意してから進める。

### 論点 1: プレビュー対象ファイル形式

- 案 A: JPEG / PNG / PDF すべて（≒ ブラウザネイティブで表示可能なものはすべて）
- 案 B: 画像のみ（JPEG / PNG）。PDF はダウンロードに留める（ブラウザ間の挙動差を避けたい場合）
- 案 C: 実装は全形式対応にしておき、ブラウザが対応していなければダウンロードフォールバック

→ ポートフォリオ品質では案 A 推奨。承認者 UX に直結。

### 論点 2: ダウンロードとプレビューの併存方法

- 案 A: ファイル名クリック = プレビュー（新タブ）/ 別途「ダウンロード」アイコン or リンクを併設
- 案 B: ファイル名クリック = プレビュー、長押しまたは右クリックメニューでダウンロード（実装コスト高）
- 案 C: ファイル名クリック = プレビュー、ファイル名横に「↓」アイコンボタンでダウンロード（推奨）
- 案 D: 詳細パネル/モーダル内で `<img>` または `<iframe>` を使って即時プレビュー、別途ダウンロードボタン併設

→ 推奨は **案 C**（一覧画面の情報密度を保ちつつ両操作を提供）または **案 D**（詳細表示モーダル新設で情報量増）。

### 論点 3: バックエンドの Content-Disposition の扱い

現状 `files.md L333`:
```
| ResponseContentDisposition | `attachment; filename="..."` | ブラウザでのダウンロード時に元ファイル名を使用 |
```

S3 署名付き URL に `attachment` 固定で付けているため、ブラウザは強制ダウンロードする。プレビューを実現するには:

- 案 A: ダウンロード用エンドポイントとプレビュー用エンドポイントを 2 つ作る（`?disposition=inline` クエリ等で切り替え）
- 案 B: API レスポンスで両方の URL（download_url / preview_url）を返す
- 案 C: フロント側で `<img src={signedUrl}>` を使う（ブラウザが Content-Type で判断 → ただし `Content-Disposition: attachment` だと img タグでも問題出る場合あり）

→ 推奨は **案 B**（API 設計上分かりやすく、認可チェックも 1 回で済む）。

### 論点 4: 認可とセキュリティ

- プレビュー URL も同じ認可ルール（同一テナント + 閲覧権限）が適用される
- 署名付き URL の有効期限はダウンロードと同じ 15 分で良いか（プレビューは閲覧中継続的にアクセスする可能性があるので長めにすべきか）
- 画像/PDF を `<img>`/`<iframe>` 直接埋め込みする場合、CSP / CORS の調整が必要か

### 論点 5: モバイル表示

スマホブラウザでの新タブ表示の挙動（特に PDF）。iOS Safari / Android Chrome で異なる場合の対応方針。
本プロジェクトは MVP ではモバイル必須ではないので「PC ブラウザ動作優先」で良いか。

### 論点 6: 設計書の修正範囲

- `screens/report-detail.md` §7 — プレビュー仕様を明記
- `files.md` §3〜§4 — Content-Disposition の使い分けを反映
- `openapi.yaml` — `preview_url` フィールド追加 or 新エンドポイント追加
- `smoke_check.md` SMK-037 — issue 101 と合わせて文言調整

---

## 事実

### 設計書（report-detail.md §7 ダウンロード L358-360）

```
| 操作方法 | ファイル名クリック |
| API | GET /api/reports/:id/items/:itemId/attachments/:attId |
| 挙動 | 署名付き URL を取得し、新しいタブでファイルを表示/ダウンロード |
```

→ 「**新しいタブでファイルを表示**/ダウンロード」と明記。プレビュー（表示）が当初想定されていた。

### 実装（AttachmentList.tsx + useAttachmentDownload.ts）

実装は同タブで blob ダウンロードのみ。新しいタブを開く処理も、`<img>` / `<iframe>` での表示処理もない。

`expense-saas/frontend/src/hooks/useAttachmentDownload.ts`（推測パス、要確認）で blob 取得 → `<a download>` 誘導 → クリック というパターンになっており、プレビュー手段なし。

### 矛盾点

設計書側にも内部矛盾がある:
- `screens/report-detail.md` §7: 「新しいタブで表示/ダウンロード」 ← プレビュー想定
- `files.md` §4: `Content-Disposition: attachment` 固定 ← ダウンロード強制

設計書時点でこの 2 つが矛盾しており、実装側はダウンロード強制側に倒した結果、プレビューが落ちた。

## 修正方針（合意後）

論点合意後に書き起こす。最低限の構成案:

### バックエンド
- `GET /api/reports/:id/items/:itemId/attachments/:attId/download` （既存維持）
- `GET /api/reports/:id/items/:itemId/attachments/:attId/preview` 新設、または既存エンドポイントに `?mode=preview` クエリ追加
- 認可は同じルール、レスポンスは S3 署名付き URL（`Content-Disposition: inline`）

### フロントエンド
- `AttachmentList.tsx` の各添付行に「プレビュー」「ダウンロード」のアクションを併設
- ファイル名クリック = プレビュー（新タブ）
- 横の「↓」アイコンボタン = ダウンロード
- ロール別表示は変更なし（Member / Approver / Accounting / Admin すべてプレビュー可、Member 所有者のみ削除可）

### テスト
- 認可テスト（preview / download それぞれ）
- フロント統合テスト（プレビュー時の `window.open` 呼び出し、ダウンロード時の blob 処理）

## 修正対象ファイル

合意後に確定するが、最低でも以下が対象:

### 設計書
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md`
- `dev-journal/deliverables/docs/50_detail_design/files.md`
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml`
- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-037

### 実装
- `expense-saas/internal/handler/attachment.go`（新エンドポイント or クエリパラメータ対応）
- `expense-saas/internal/service/...`（署名付き URL 発行ロジックの分岐）
- `expense-saas/frontend/src/pages/reports/AttachmentList.tsx`
- `expense-saas/frontend/src/hooks/useAttachmentDownload.ts` および新規 `useAttachmentPreview.ts`
- 関連テストファイル

## 完了条件

- 論点 1〜6 がユーザーと合意され、合意内容が本 issue に追記されている
- バックエンドにプレビュー用 URL 発行手段が実装され、認可テストが通過する
- フロントエンドでファイル名クリックでプレビュー、別途ダウンロード操作が提供される
- JPEG / PNG / PDF を Member / Approver / Accounting / Admin それぞれで動作確認できる
- 設計書（report-detail.md, files.md, openapi.yaml）が更新され、内部矛盾が解消されている
- smoke_check.md SMK-037 が新挙動に合わせて更新されている

## 実装ログ（2026-04-17）

### ブランチ戦略

統合ブランチ `integration/102-attachment-preview` + stacked PR（BE → FE）方式を採用。

### BE PR #64（`step11/102-be-preview`、マージ済み `26c86c8`）

- `AttachmentAccess` スキーマへの DTO リネーム（`AttachmentDownload` / `DownloadURL` を廃止）
- `/download` / `/preview` エンドポイント分割ルーティング
- `PresignGetObject(disposition)` 対応（`GeneratePresignedURL` 共通化）
- テスト: ATT-055〜060（preview 4 ロール + 不在 + Forbidden）+ CRS-010b（テナント越境）

### FE PR #65（`step11/102-fe-preview`、レビュー完了・マージ待ち）

- 型 `AttachmentDownload` → `AttachmentAccess` リネーム（`download_url` → `url`）
- hook 2 分割: `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl`（`enabled: false` + 明示的 `refetch()` 方式）
- **重要な設計調整**: 当初の計画（`AttachmentArea` orchestration + `AttachmentList` presentational）は hooks rules 制約（ループ内での hook 呼び出し禁止）で成立せず、`AttachmentList` に per-item hook orchestration を内包する構造（`AttachmentItemRow` 内部コンポーネント）に変更した
- 設計書（`report-detail.md` / `files.md` / `test_cases/attachments.md`）も同時更新（dev-journal `4bddeeb`）

### 残作業

FE PR #65 マージ → integration ブランチで結合動作確認（SMK-037、認可 4 ロール × 2 操作）→ 最終 PR（integration → master squash）

---

## 関連

- 101: SMK-037 文言誤記 — 本 issue の対応で SMK 期待結果も書き換わる
- 088 / PR #53: 認可エラー UX — 同じく承認者業務まわりの UX 改善系
- files.md / report-detail.md の内部矛盾は元々 Step 5 詳細設計時点で見落とされていた論点

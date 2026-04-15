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
- 両方とも同じ認可ロジックを呼ぶ（service 層に共通関数）
- レスポンス型は同じ `{ url, expires_at }`
- フロント hook も 2 つに分割: `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl`

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

### 決定 6: 設計書修正範囲 → **叩き台を提示、codex レビューで監査**

以下は指揮役が整理した叩き台。抜け漏れ・不要項目を codex に監査してもらう。

#### 設計書（修正対象）

| # | ファイル | 修正内容 |
|---|---|---|
| D1 | `deliverables/docs/50_detail_design/openapi.yaml` | 既存 `GET /api/.../attachments/:attId` を `/download` にリネーム。新規 `/preview` エンドポイント追加。両方ともレスポンススキーマ `{ url, expires_at }` を定義、認可ルールを既存同等と明記 |
| D2 | `deliverables/docs/50_detail_design/files.md` §3〜§4 | `Content-Disposition` の扱いを「ダウンロード時は `attachment`、プレビュー時は `inline`」に変更。`ResponseContentDisposition` パラメータを用途別に記述。既存の `attachment` 固定記述を修正 |
| D3 | `deliverables/docs/50_detail_design/screens/report-detail.md` §7 | 操作方法を「ファイル名クリック = プレビュー、↓ アイコン = ダウンロード」に変更。UI モック（L266 付近）に ↓ アイコン追加。挙動記述を「新しいタブでファイルを表示（画像・PDF）またはダウンロード」に明確化 |
| D4 | `deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-037 | プレビュー動作の期待結果を追加。「1. ファイル名テキストを押下 → 新しいタブで表示。2. ↓ アイコン押下 → ダウンロード開始」 |
| D5 | `deliverables/docs/60_test/test_cases/attachments.md`（存在確認要）| プレビュー関連のテストケース追加（認可 4 ロール × 2 操作）|

#### テスト（新規 / 更新）

| # | 種別 | ファイル（想定）| 内容 |
|---|---|---|---|
| T1 | BE handler test | `internal/handler/attachment_handler_test.go` | `TestGetPreviewURL` 追加、4 ロール × 2 エンドポイントの認可テスト |
| T2 | BE service test | `internal/service/attachment_service_test.go` | `GeneratePreviewURL` / `GenerateDownloadURL` の分岐テスト |
| T3 | FE hook test | `frontend/src/hooks/__tests__/useAttachmentDownloadUrl.test.tsx`（リネーム）+ 新規 `useAttachmentPreviewUrl.test.tsx` | エンドポイント変更を反映、両 hook の API 呼び出し検証 |
| T4 | FE component test | `frontend/src/pages/reports/__tests__/AttachmentList.test.tsx` | ファイル名クリックで `window.open(preview_url)`、↓ アイコンで `window.open(download_url)` を検証 |
| T5 | Cross-cutting test | `internal/handler/cross_cutting_test.go`（Step 11-B で実装予定）| RBAC マトリクスに `/preview` エンドポイント追加 |

#### 実装（新規 / 更新）

| # | 種別 | ファイル（想定）| 内容 |
|---|---|---|---|
| I1 | BE handler | `internal/handler/attachment_handler.go` | `GetPreviewURL` メソッド追加、`GetDownloadURL` にリネーム（既存ロジック流用）|
| I2 | BE service | `internal/service/attachment_service.go` | `GeneratePresignedURL(disposition string)` 共通関数 + 公開関数 2 つ |
| I3 | BE ルーティング | `internal/router/router.go`（想定）| 新エンドポイント 2 つを登録（既存 1 つはリネーム）|
| I4 | FE hook | `frontend/src/hooks/useAttachmentDownloadUrl.ts` + 新規 `useAttachmentPreviewUrl.ts` | 既存 `useAttachmentDownload.ts` を分割 |
| I5 | FE component | `frontend/src/pages/reports/AttachmentList.tsx` | ファイル名 onClick、↓ IconButton 追加、window.open 呼び出し |
| I6 | FE 型定義 | `frontend/src/api/types.ts`（想定）| `AttachmentDownload` 型を用途別に分割 or リネーム |

### 実装の PR 分割案

設計成果物 / BE / FE の順で 3 段階に分割:

1. **PR-1: 設計書修正**（設計成果物フロー）
   - D1〜D5 を 1 コミット
   - reviewer → codex レビュー → 合意
2. **PR-2: BE 実装 + テスト**（PR フロー）
   - I1〜I3 + T1〜T2, T5
   - ローカル CI (go test / golangci-lint) 通過で merge
3. **PR-3: FE 実装 + テスト**（PR フロー）
   - I4〜I6 + T3〜T4
   - PR-2 マージ後。ローカル CI (vitest / tsc / eslint / build) 通過で merge

### 影響範囲の最終まとめ

- **BE 変更**: handler 1, service 1, router 1、新エンドポイント 1、既存エンドポイント 1 リネーム
- **FE 変更**: hook 1 → 2 に分割、AttachmentList UI 修正、test 2 ファイル追加
- **設計書変更**: 4〜5 ファイル
- **既存の認可ロジック / S3 設定 / env 変数 / DB schema は変更なし**
- **マイグレーション不要**

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

## 関連

- 101: SMK-037 文言誤記 — 本 issue の対応で SMK 期待結果も書き換わる
- 088 / PR #53: 認可エラー UX — 同じく承認者業務まわりの UX 改善系
- files.md / report-detail.md の内部矛盾は元々 Step 5 詳細設計時点で見落とされていた論点

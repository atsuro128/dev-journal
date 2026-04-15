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

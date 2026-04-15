# 101: issue 102 の論点 6 叩き台は横断更新範囲と実装導線を一部取りこぼしている

## 指摘 1
### 重大度
warning

### 指摘概要
PR 分割案の `PR-2: BE 実装 + テスト` で既存 `GET /api/reports/{id}/items/{itemId}/attachments/{attId}` を `/download` へリネームすると、`PR-3` がマージされるまで現行フロントエンドが即時に壊れる。設計書 → BE → FE の 3 段階は順序自体は理解できるが、BE PR を単独で main に入れる前提だと互換性維持策が必要。

### 根拠
- 現行 FE は [AttachmentArea.tsx](/root-project/expense-saas/frontend/src/pages/reports/AttachmentArea.tsx:58) で `GET /api/reports/${reportId}/items/${itemId}/attachments/${attachmentId}` を直接呼んでいる。
- 現行 Hook も [useAttachmentDownload.ts](/root-project/expense-saas/frontend/src/hooks/useAttachmentDownload.ts:19) で同一エンドポイントを呼んでいる。
- 現行 BE ルーティングは [main.go](/root-project/expense-saas/cmd/server/main.go:209) と [http.go](/root-project/expense-saas/internal/testutil/http.go:122) の両方で旧パスを登録している。
- 叩き台は D1/I3 で既存エンドポイントを `/download` にリネームすると明記しており、互換用 alias を残す記述がない。

### 修正方針案
- `PR-2` では旧パスを一時互換 alias として残し、`PR-3` 完了後に削除する。
- もしくは BE/FE を同一 PR にまとめ、main 上で API 契約が中間破壊される期間を作らない。

## 指摘 2
### 重大度
warning

### 指摘概要
論点 2/3 の「ファイル名クリック = `window.open(preview_url)`」は、署名付き URL を API 取得したあとに新規タブを開く前提だと、ブラウザのポップアップブロックに引っかかる実装経路を残す。新タブ方式を採るなら、クリック同期で空タブを開いてから遷移させるか、互換的な導線を設計書で固定したほうがよい。

### 根拠
- 現行設計は [report-detail.md](/root-project/dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md:360) で「署名付き URL を取得し、新しいタブでファイルを表示/ダウンロード」としており、API 取得後にタブ遷移する前提になっている。
- 現行実装も [AttachmentArea.tsx](/root-project/expense-saas/frontend/src/pages/reports/AttachmentArea.tsx:58) のように、クリック後に非同期で URL を取得してからブラウザ操作を行う構造である。
- 叩き台の決定 2 は `window.open(preview_url)` をそのまま想定しているが、非同期 fetch 後の `window.open` の扱いは設計に明文化されていない。

### 修正方針案
- 設計書に「クリック時に空タブを同期で開き、URL 取得後に `location` を差し替える」などの実装制約を追記する。
- もしくは preview のみ暫定的に同タブ遷移へ倒すか、互換ブラウザ方針を明記する。

## 指摘 3
### 重大度
suggestion

### 指摘概要
設計書修正対象 D1〜D5 だけでは閉じず、`50_detail_design`・`55_ui_component`・`60_test` の横断参照を更新しないと、添付プレビュー仕様と API 契約の正本が分裂したまま残る。`architecture.md` は現行ツリーに存在しない一方、`authz.md` と `security.md` は実更新対象。

### 根拠
- `50_detail_design` 側では [authz.md](/root-project/dev-journal/deliverables/docs/50_detail_design/authz.md:195) にルート定義断片、[authz.md](/root-project/dev-journal/deliverables/docs/50_detail_design/authz.md:325) に添付 API マトリクスがあり、旧 `/attachments/{attId}` GET 前提になっている。
- [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md:425) は `img-src` に S3 を許可しつつ、[security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md:426) では「S3 ダウンロードはブラウザの直接アクセスではなく API 経由の署名付き URL リダイレクト」と記載している。preview 導入後は表現整理が必要。
- `55_ui_component` 側では [state-management.md](/root-project/dev-journal/deliverables/docs/55_ui_component/state-management.md:127) と [screens/report-detail.md](/root-project/dev-journal/deliverables/docs/55_ui_component/screens/report-detail.md:512) が `useAttachmentDownload` とダウンロード前提 UI を定義している。
- `60_test` 側では [traceability.md](/root-project/dev-journal/deliverables/docs/60_test/traceability.md:79) が ATT-F03 の正本参照を `getAttachmentDownload` 前提で保持している。
- `50_detail_design/architecture.md` は現行リポジトリに存在しない。

### 修正方針案
- D1〜D5 に加えて少なくとも `50_detail_design/authz.md`, `50_detail_design/security.md`, `55_ui_component/state-management.md`, `55_ui_component/screens/report-detail.md`, `60_test/traceability.md` を対象へ加える。
- `architecture.md` は対象候補から外し、存在するファイルに限定して修正範囲を再定義する。

## 指摘 4
### 重大度
suggestion

### 指摘概要
T1〜T5 は既存のテストファイル構成と一部合っておらず、preview 追加時に更新すべき既存テスト資産も取りこぼしている。特に `AttachmentList.test.tsx` に `window.open` を持たせる想定と、`internal/handler/cross_cutting_test.go` 想定は現行構造とずれている。

### 根拠
- 既存 BE の添付関連統合テストは [attachment_handler_test.go](/root-project/expense-saas/internal/handler/attachment_handler_test.go:45) に `ATT-030`〜`ATT-041` とテナント分離ケースまでまとまっており、`internal/handler/cross_cutting_test.go` は存在しない。
- 既存 cross-cutting 正本は [cross-cutting.md](/root-project/dev-journal/deliverables/docs/60_test/test_cases/cross-cutting.md:147) で、添付の越境 GET は `CRS-010` として `GetAttachmentDownload` 前提になっている。
- `AttachmentList` は presentational component であり、実際の API 呼び出しとブラウザ操作は [AttachmentArea.tsx](/root-project/expense-saas/frontend/src/pages/reports/AttachmentArea.tsx:58) が持っている。したがって [AttachmentList.test.tsx](/root-project/expense-saas/frontend/src/pages/reports/__tests__/AttachmentList.test.tsx:44) で直接 `window.open(...)` を検証する責務ではない。
- 既存 FE 統合テストは [AttachmentArea.integration.test.tsx](/root-project/expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.integration.test.tsx:378) にダウンロード導線の API 呼び出し検証がある。
- `internal/service/attachment_service_test.go` は現状存在せず、追加自体は可能だが「既存構成と合っているか」という観点では新設扱いになる。

### 修正方針案
- T1 は `attachment_handler_test.go` に preview 系の mirror ケースを追加する前提で整理する。
- T4 は `AttachmentList.test.tsx` では `onPreview` / `onDownload` コールバック検証に留め、`window.open` や API 呼び出しは `AttachmentArea.integration.test.tsx` 側へ寄せる。
- T5 は nonexistent な `cross_cutting_test.go` 前提を外し、既存の `attachment_handler_test.go` か将来の明示的な新規 test file のどちらかに寄せて記載する。
- `attachments.md`, `cross-cutting.md`, `traceability.md` の preview ケース追加も T 項目に含める。

## 指摘 5
### 重大度
suggestion

### 指摘概要
I1〜I6 の実装ファイルマップに実在しないパスや責務の誤配置がある。特に BE ルーティング先と FE のイベント起点が現行実装と一致していない。

### 根拠
- 実ハンドラ本体は [attachment.go](/root-project/expense-saas/internal/handler/attachment.go:1) であり、叩き台の `internal/handler/attachment_handler.go` は存在しない。
- ルーティングは [main.go](/root-project/expense-saas/cmd/server/main.go:188) と [http.go](/root-project/expense-saas/internal/testutil/http.go:102) が持っており、`internal/router/router.go` は存在しない。
- FE でダウンロード API を叩いているのは [AttachmentArea.tsx](/root-project/expense-saas/frontend/src/pages/reports/AttachmentArea.tsx:58) で、[AttachmentList.tsx](/root-project/expense-saas/frontend/src/pages/reports/AttachmentList.tsx:28) は callback を受けるだけである。
- API 契約を変える場合、[dto.go](/root-project/expense-saas/internal/domain/dto.go:68), [interfaces.go](/root-project/expense-saas/internal/service/interfaces.go:75), [types.ts](/root-project/expense-saas/frontend/src/api/types.ts:96) も追従が必要だが、I1〜I6 には含まれていない。

### 修正方針案
- 実装対象を `internal/handler/attachment.go`, `cmd/server/main.go`, `internal/testutil/http.go`, `frontend/src/pages/reports/AttachmentArea.tsx`, `frontend/src/pages/reports/AttachmentList.tsx`, `internal/domain/dto.go`, `internal/service/interfaces.go`, `frontend/src/api/types.ts` ベースで引き直す。
- Hook 分割を採るなら `AttachmentArea.tsx` から直接 `api.get` を外し、そこへ preview/download の orchestration を寄せる前提を明文化する。

## 指摘 6
### 重大度
suggestion

### 指摘概要
openapi.yaml の「リネーム + 新設」は、単にパスを 2 本に増やすだけではなく、既存 `AttachmentDownload` 契約をどう扱うかまで明示しないと影響範囲を取りこぼす。叩き台の `{ url, expires_at }` への簡素化は可能だが、現行仕様・型・テストは `download_url`, `file_name`, `mime_type`, `file_size`, `expires_at` を前提にしている。

### 根拠
- 現行 OpenAPI は [openapi.yaml](/root-project/dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1265) で `getAttachmentDownload` を定義し、[openapi.yaml](/root-project/dev-journal/deliverables/docs/50_detail_design/openapi.yaml:2187) の `AttachmentDownload` スキーマは 5 フィールド必須である。
- 現行 Files 設計も [files.md](/root-project/dev-journal/deliverables/docs/50_detail_design/files.md:305) 以降のレスポンス例と [files.md](/root-project/dev-journal/deliverables/docs/50_detail_design/files.md:328) の署名付き URL 設計が `download_url` 前提で書かれている。
- 現行 FE 型は [types.ts](/root-project/expense-saas/frontend/src/api/types.ts:104) の `AttachmentDownload` が同じ 5 フィールドを持つ。
- `attachments.md` も [attachments.md](/root-project/dev-journal/deliverables/docs/60_test/test_cases/attachments.md:160) で `data.download_url`, `file_name`, `mime_type`, `file_size`, `expires_at` を期待している。

### 修正方針案
- `preview` / `download` の両レスポンスを本当に `{ url, expires_at }` に統一するか、既存 `AttachmentDownload` 互換を残すかを論点 6 に明示する。
- 前者を選ぶなら `openapi.yaml`, `files.md`, `types.ts`, `dto.go`, `attachments.md`, `traceability.md`, `55_ui_component` を同時更新対象に含める。
- 後者を選ぶなら preview 用に `preview_url` を持つ別スキーマか、共通 `AttachmentAccess` スキーマを切るほうが追跡しやすい。

## 総評
- 論点 1, 4, 5 の方向性自体には大きな矛盾は見当たらない。
- 論点 2, 3 も採用可能だが、`window.open` の実装条件と API 互換性を仕様として固定しないと、PR 分割時とブラウザ実挙動で事故りやすい。
- 論点 6 は「D1〜D5 だけで閉じる」前提が甘く、実際には `authz/security/55_ui_component/traceability` まで含めた横断修正として再定義したほうがよい。

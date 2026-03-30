# 077: API クライアントが multipart/form-data を送信できない

## 指摘概要
`api.post()` / `api.put()` が渡された `body` を無条件に `JSON.stringify()` しているため、添付アップロードで必要な `FormData` をそのまま送信できない。8-5 の共通 API クライアント基盤が Step 10 の添付機能実装の前提を満たしておらず、下流担当が追加解釈なしに実装を進められない。

## 根拠
- `expense-saas/frontend/src/api/client.ts` では `FormData` の場合に `Content-Type` を自動付与しない分岐があるが、判定対象は `init.body instanceof FormData` である。`api.post()` / `api.put()` はその前段で `body` を `JSON.stringify()` しており、`FormData` が `RequestInit.body` に到達しない。
- `expense-saas/frontend/src/api/client.ts` L59-L61, L130-L138
- `dev-journal/deliverables/docs/30_arch/architecture.md` L357-L357 では `/api/reports/:id/items/:itemId/attachments` を `multipart/form-data` と定義している。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml` L1113-L1153 でも同エンドポイントの `requestBody` を `multipart/form-data` と定義している。

## 判定
高 / 実装不備

## 修正方針案
- `api.post()` / `api.put()` で `body` を一律 stringify せず、`FormData`・`string`・`undefined` をそのまま透過できるようにする。
- もしくは `api.request()` のような汎用入口を追加し、Step 10 側が `RequestInit` を直接渡せる契約にする。
- 併せて multipart 送信のユニットテストか最小疎通確認を追加し、添付アップロード実装の前提を固定する。

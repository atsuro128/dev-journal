# 079: エラーコード定数が正本仕様を網羅していない

## 指摘概要
`constants.ts` の `ERROR_CODES` は上流仕様で定義済みのエラーコードを一部しか持っておらず、フロントエンドの共通エラーハンドリング契約として不完全である。Step 10 の実装者がどのコードを定数参照し、どれを文字列直書きするのか判断できない。

## 根拠
- `expense-saas/frontend/src/lib/constants.ts` L1-L9 にあるのは `TOKEN_EXPIRED` / `INVALID_TOKEN` / `UNAUTHORIZED` / `FORBIDDEN` / `RATE_LIMIT_EXCEEDED` / `VALIDATION_ERROR` / `INVALID_CREDENTIALS` のみ。
- 一方で正本仕様の `dev-journal/deliverables/docs/50_detail_design/security.md` L551-L572 には `BAD_REQUEST`, `RESOURCE_NOT_FOUND`, `CONFLICT`, `FILE_TOO_LARGE`, `INVALID_FILE_TYPE`, `INVALID_STATE_TRANSITION`, `SELF_APPROVAL_NOT_ALLOWED`, `SELF_PAYMENT_NOT_ALLOWED` など、Step 10 で扱うコードが追加で定義されている。
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md` L196-L208 は Step 10 に渡す共通契約として「API エラーレスポンス形式」と「フロントエンド API クライアントの責務（共通エラーハンドリング）」の固定を求めている。

## 判定
中 / 契約不完全

## 修正方針案
- `security.md` の正本に合わせて `ERROR_CODES` を網羅化し、フロントエンドが参照すべき定数の唯一の入口として固定する。
- もし auth 系だけを先行定義する意図なら、その責務境界をチケットかコードコメントに明記し、業務 API 用定数をどこで管理するかを別途固定する。

# 084: （タイトル：フロントエンド CI テストジョブがテスト不在で必ず失敗する）

## 指摘概要
`expense-saas/frontend` に Vitest のテストファイルが存在しない一方で、CI は `npx vitest run` を必須ジョブとして実行している。現状のままでは PR 時の `lint → test → build` パイプラインが常に `test-frontend` で失敗し、Step 8-9 の完了条件を満たせない。

## 根拠
- `dev-journal/deliverables/docs/30_arch/architecture.md:481-484`
  - PR パイプラインの frontend test は `vitest run` と定義されている。
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md:116`
  - 完了条件として「CI パイプラインが動作する（PR 時に lint / test / build が自動実行される）」ことを要求している。
- `expense-saas/.github/workflows/ci.yml:114-116`
  - `test-frontend` ジョブで `npx vitest run` を実行している。
- 実行結果
  - `/root-project/expense-saas/frontend` で `npm test` を実行すると `No test files found, exiting with code 1` で終了した。

## 判定
高 / FIX

## 修正方針案
- 最低 1 本でも Vitest テストを追加し、`npx vitest run` が成功する状態にする。
- もし Step 8 時点で frontend test を未着手扱いにするなら、architecture.md / work-breakdown / CI 定義を同時に修正し、Step 8 の完了条件と矛盾しないようにする。

## 解決内容
- `frontend/src/test/App.test.tsx` にスモークテストを追加し `vitest run` が成功する状態にした
- `frontend/tsconfig.json` に `exclude: ["src/test"]` を追加し、テストファイルが `tsc -b` ビルドに含まれないようにした

## 解決日
2026-03-31

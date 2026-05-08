# deploy.yml の e2e ジョブが Playwright プロジェクト構成と不整合（5 件）

## 発見日
2026-05-08

## カテゴリ
infrastructure

## 影響度
中

## 発見経緯
review

## 関連ステップ
Step 11-E（デプロイ・スモークテスト）

## ブロッカー
なし（現状 `if: false` でジョブは無効化されており、master のビルド・デプロイは塞がない。ただし 11-E で `if: false` を外す前に必ず解消する必要がある）

## 問題

`expense-saas/.github/workflows/deploy.yml` の `e2e` ジョブ（L157-203）が、PR #138（11-C）以降に確定した Playwright のプロジェクト構成と整合していない。Step 11-D の codex 横断レビューでは PR #142 の補足として「`npx playwright test` の `--config` 未指定」が指摘されたが、精査したところ実際には e2e ジョブ全体に 5 件の不整合がある。

| # | 行 | 現状 | 問題 |
|---|----|------|------|
| 1 | L150 | `npm ci` を `working-directory: frontend` で実行 | `@playwright/test` は **expense-saas root** の `package.json`（`"name": "expense-saas-e2e"`）に依存定義されている。frontend には未インストール |
| 2 | L170 | `cache-dependency-path: frontend/package-lock.json` | root の `package-lock.json` がキャッシュ対象に含まれていない |
| 3 | L177-178 | `working-directory: frontend` で `npx playwright install --with-deps chromium` | playwright が frontend に存在しないため、ブラウザインストールが失敗するか別バージョンが入る |
| 4 | L186-188 | `working-directory: frontend` で `npx playwright test`（config 未指定） | (a) ディレクトリ違い（playwright config は root の `e2e/playwright.config.ts`） (b) `--config e2e/playwright.config.ts` が未指定。**PR #142 codex 補足の本件** |
| 5 | L196-198 | レポートアップロード `path: frontend/playwright-report/` | 実際のレポート出力は `playwright.config.ts` 既定の root `playwright-report/` |

## 影響

- 11-E でデプロイパイプラインの `if: false` を外した際、e2e ジョブが「セットアップ段階で失敗 → デプロイ前ゲートが機能しない」状態になる
- 仮にローカル `npm run e2e` 経路と CI 経路で別の挙動になり、CI 上で「テスト未実行のまま PASS 扱い」になるリスク
- Step 11-C で確定した Playwright 構成（root ディレクトリの e2e ワークスペース、`npm run e2e` スクリプトに `--config` 指定済み）が CI 側に反映されておらず、整合管理が崩れている

## 提案

e2e ジョブを **expense-saas root を作業ディレクトリ前提** に書き換え、root の `npm ci` + `npm run e2e` を使う形に整理する。

具体的な修正案:

1. `cache-dependency-path` を root の `package-lock.json` に変更（または frontend / root 両方を列挙）
2. `npm ci` を `working-directory:` 省略（= リポジトリ root）で実行
3. `npx playwright install --with-deps chromium` も root で実行
4. テスト実行を `npm run e2e`（既存スクリプトが `--config e2e/playwright.config.ts` を含む）に置き換え、または `npx playwright test --config e2e/playwright.config.ts` を root で直接呼ぶ
5. `actions/upload-artifact` の `path:` を root `playwright-report/` に修正

このスコープでは `if: false` の解除は行わない。解除は 11-E 本番で AWS デプロイ手順と合わせて判断する。

### 受け入れ条件

- 5 件の不整合がすべて解消されている
- `gh workflow run` 等で e2e ジョブを単体実行した場合に、セットアップ → playwright install → `npm run e2e` 相当のコマンドが root 配下で正しく走る経路になっている（`if: false` のため CI 上では実行されないが、yaml の妥当性は確保）
- Step 11-C の Playwright 構成（`expense-saas/e2e/playwright.config.ts`、`expense-saas/package.json` の `e2e` スクリプト）と参照経路が一致している

## 追加対応（2026-05-08、PR #144 codex レビューで発見）

PR #144 codex レビュー（gpt-5.4）で `Wait for services` ステップが `/healthz` を待っているが、実装は `cmd/server/main.go:169` で `/health` のみ公開している既存バグを発見。`02_scope.md:72` / `runbook.md:81` も `/health` を正本としているため、`if: false` 解除（11-E 本番）時に必ず e2e/smoke ジョブが起動失敗する。

### 追加修正範囲

- `deploy.yml:184`（e2e ジョブの `Wait for services`）: `/healthz` → `/health`
- `deploy.yml:220`（smoke ジョブの `Wait for services`）: `/healthz` → `/health`（同一 bug、grep で発見、ユーザー判断でスコープ含めた）

issue 本来の 5 件不整合と趣旨（「e2e ジョブを正しく動かす」）が一致するため PR #144 に追加コミットで同梱。

---

## 解決内容

PR #144 (`step11/175-fix-deploy-yml-e2e-job`) を 2026-05-08 に master へスカッシュマージ（merge commit `6c99001`）して解決。

### 変更ファイル

- `expense-saas/.github/workflows/deploy.yml`（13 行変更）

### コミット履歴

1. `925a53b` fix(ci): align deploy.yml e2e job with playwright workspace at repo root (issue #175)
   - 5 件の不整合を一括修正:
     - `cache-dependency-path: frontend/package-lock.json` → `package-lock.json`（root）
     - `working-directory: frontend` を 3 箇所（npm ci / playwright install / playwright test）から削除
     - `npx playwright test` → `npm run e2e`（root の `--config e2e/playwright.config.ts` 既存スクリプト経由）
     - `path: frontend/playwright-report/` → `path: playwright-report/`
2. `f88e4b4` fix(ci): use /health instead of /healthz in deploy.yml wait-on (issue #175 codex review)
   - PR #144 codex レビューで発見した既存バグを追加対応:
     - `wait-on http://localhost:8080/healthz` → `/health`（e2e ジョブ L184 + smoke ジョブ L220 の 2 箇所）

### レビュー履歴

- 内部レビュー (reviewer エージェント): PASS（blocker / warning 0、info 1）
- codex レビュー初回 (gpt-5.4): FAIL（blocker `/healthz` vs `/health` 不整合 1 件）
- codex 再レビュー (gpt-5.4): PASS（blocker / warning 0、追加指摘なし）
  - 注: 2 回目の再レビュー実行時に古い別ブランチ `step11/issue-144-report-info-card-labels` を誤特定する事故があり、ブランチ名・HEAD SHA を明示した 3 回目の実行で正しく再レビュー完了した

### `if: false` の扱い

このスコープでは `if: false` を解除しない（11-E 本番で AWS 環境準備と合わせて実施）。本 PR は yaml の構造的妥当性確保のみが目的。

## 解決日
2026-05-08

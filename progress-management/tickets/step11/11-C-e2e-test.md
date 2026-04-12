# E2E テスト（Playwright）

- 担当: test-implementer
- 依存: 11-A
- ブランチ: `step11/11-C-e2e-test`
- 出力先: expense-saas/e2e/
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テスト戦略 | deliverables/docs/60_test/test_strategy.md | §10（Playwright E2E 方針: ブラウザ選定・ファイル配置・テストアカウント） |
| 横断テストケース | deliverables/docs/60_test/test_cases/cross-cutting.md | §3（CRS-055〜CRS-075 E2E シナリオ） |
| 全機能の実装コード | expense-saas/ | フロントエンド・バックエンド |

## 責務

- **基盤セットアップ**:
  - Playwright 依存の追加（`package.json` に `@playwright/test`）
  - `expense-saas/e2e/` ディレクトリ作成
  - `playwright.config.ts` 作成（Chromium のみ、test_strategy.md §10.1 準拠）
  - E2E 用フィクスチャ投入の仕組み構築（`e2e/setup.ts`）
  - docker-compose.yml への E2E 実行用設定追加（必要に応じて）
- **テスト実装**:
  - フロー1: 申請 → 承認 → 支払完了（CRS-055〜CRS-065）: `e2e/flow1_test.ts`
  - フロー2: 却下 → 再申請（CRS-066〜CRS-072）: `e2e/flow2_test.ts`
  - フロー3（オプション）: Admin テナント管理（CRS-073〜CRS-075）: `e2e/flow3_admin_test.ts` — 低優先度、ベストエフォート
- CI ワークフローに Playwright テストステップを追加
- 含めない: Go 横断テスト（11-B で対応）

## 完了条件

- Playwright の基盤（依存・設定ファイル・ディレクトリ）がセットアップされている
- CRS-055（フロー1）と CRS-066（フロー2）の E2E テストが実装され、ローカルまたは CI で通過している
- 申請 → 承認 → 支払の一連フローが UI 操作で通ることが E2E テストで検証されている

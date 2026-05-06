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
- ホスト側でローカル実行できるようにする（test_strategy.md §5、work-breakdown §11-C 準拠。11-B 非機能テストと同タイミング）
- 含めない: Go 横断テスト（11-B で対応）、CI ワークフローへの組み込み（ローカル実行で担保）

## 対応テストケース

設計上の対応関係を示す（実施状況は「実施ログ」を参照）。

| ID | テストケース | 実装先 | メモ |
|----|--------------|--------|------|
| CRS-055 | フロー1 全体（申請 → 承認 → 支払完了）E2E | `expense-saas/e2e/flow1_test.ts` | CRS-056〜CRS-065 を一連の E2E として実行 |
| CRS-056 | Member ログイン | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-057 | レポート作成 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-058 | 明細追加 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-059 | 提出 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-060 | Approver ログイン | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-061 | 承認待ち一覧表示 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-062 | 承認 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-063 | Accounting ログイン | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-064 | 支払対象一覧表示 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-065 | 支払完了 | `expense-saas/e2e/flow1_test.ts` | フロー1 内ステップ |
| CRS-066 | フロー2 全体（却下 → 再申請）E2E | `expense-saas/e2e/flow2_test.ts` | CRS-067〜CRS-072 を一連の E2E として実行 |
| CRS-067 | 提出までの初期動作 | `expense-saas/e2e/flow2_test.ts` | フロー2 内ステップ |
| CRS-068 | Approver による却下 | `expense-saas/e2e/flow2_test.ts` | フロー2 内ステップ |
| CRS-069 | 却下理由の表示確認 | `expense-saas/e2e/flow2_test.ts` | フロー2 内ステップ |
| CRS-070 | 再申請（明細コピー） | `expense-saas/e2e/flow2_test.ts` | `openapi.yaml#createReport` の説明に依存 |
| CRS-071 | 再申請（添付未コピー）の確認 | `expense-saas/e2e/flow2_test.ts` | `openapi.yaml#createReport` の説明に依存 |
| CRS-072 | 再申請レポートの提出 | `expense-saas/e2e/flow2_test.ts` | フロー2 内ステップ |
| CRS-073 | Admin ダッシュボード表示 | `expense-saas/e2e/flow3_admin_test.ts` | 低優先度・ベストエフォート |
| CRS-074 | Admin による全レポート一覧 | `expense-saas/e2e/flow3_admin_test.ts` | 低優先度・ベストエフォート |
| CRS-075 | Admin によるテナント情報閲覧 | `expense-saas/e2e/flow3_admin_test.ts` | 低優先度・ベストエフォート |

## 進め方

1. `test_strategy.md` §10 と `cross-cutting.md` §3 を読み、E2E の対象ブラウザ・テストアカウント・フィクスチャ方針を確認する
   - CRS-070 / CRS-071（再申請レポートの明細コピー、添付未コピー）は `openapi.yaml#createReport` の説明（再申請時に明細はコピーされ、添付はコピーされない）に依存するため、該当箇所を併読する
2. Playwright の設定と E2E ディレクトリを作成し、ローカルで起動済みアプリに接続できる最小テストを通す
3. フロー1（申請 → 承認 → 支払完了）を実装し、UI 操作だけで最後まで通ることを確認する
4. フロー2（却下 → 再申請）を実装し、状態遷移と表示が仕様通りであることを確認する
5. Admin フローは必須フロー完了後に、残時間に応じてベストエフォートで実装する
6. 失敗時は、テスト不備・データ不備・機能不具合・仕様不明のどれかに分類して記録する

## 実施ログ

進捗記録用（設計上の対応関係は上記「対応テストケース」を参照）。

| 区分 | 対象 | 結果 | メモ |
|------|------|------|------|
| E2E 基盤 | Playwright 設定・起動確認 | 未実施 | |
| フロー1 | CRS-055〜CRS-065 | 未実施 | 申請 → 承認 → 支払完了 |
| フロー2 | CRS-066〜CRS-072 | 未実施 | 却下 → 再申請 |
| フロー3 | CRS-073〜CRS-075 | 未実施 | Admin。低優先度、ベストエフォート |

## 11-D への引き継ぎ

- E2E で検出した仕様不一致・画面導線の不明点
- テストデータ投入やログイン処理など、E2E 基盤側に依存する制約
- 主要業務フローのうち自動化できず手動確認に回した箇所
- UI と API の状態表示に差異があった箇所

## 完了条件

- Playwright の基盤（依存・設定ファイル・ディレクトリ）がセットアップされている
- CRS-055（フロー1）と CRS-066（フロー2）の E2E テストが実装され、ローカルで通過している
- 申請 → 承認 → 支払の一連フローが UI 操作で通ることが E2E テストで検証されている

## 更新履歴

- 2026-04-12: 初版起票（commit `a2d642d`）
- 2026-04-23: 進め方・実施ログ・引き継ぎセクション追加（commit `07bd2ec`）
- 2026-05-06: 上流成果物（test_strategy.md §5 / work-breakdown §11-C）の方針転換に追従
  - 責務: 「CI ワークフローに Playwright テストステップを追加」→「ホスト側でローカル実行」に変更（test_strategy.md §5 commit `92f723e` / work-breakdown commit `766dfa1` 準拠）
  - 完了条件: 「ローカルまたは CI で通過」→「ローカルで通過」に統一
  - 「対応テストケース」表（CRS-055〜CRS-075 と実装ファイルのマッピング）を追加
  - 進め方手順 1 に CRS-070 / CRS-071 が `openapi.yaml#createReport` の説明に依存する旨を追記

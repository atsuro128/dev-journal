# seed にレポート 0 件の Member ユーザーを追加（SMK-084 前提データ整備）

## メタ情報

| 項目 | 値 |
|---|---|
| ID | 148 |
| 起票日 | 2026-04-25 |
| 起票者 | Claude Code（指揮役） |

## 発見日
2026-04-25

## カテゴリ
test-data / implementation

## 影響度
中（既存ハンドラテスト 2 件と test_cases/dashboard.md の更新が必要）

## 発見経緯
Step 11-A Phase 2 SMK-084（空状態表示）の実施準備中、現状の seed に「レポート 0 件の Member ユーザー」が存在しないため SMK-084 を実施できないことが判明。

## 関連ステップ
Step 10-B（レポート UI 実装）/ Step 11-A（ローカル動作確認 Phase 2 SMK-084）

## ブロッカー
なし

## 関連 issue

- **#147**（open）: per_page UI セレクタ実装。本 issue と独立して並行着手可能
- SMK-084（smoke_check.md §11 Phase 2）: 本 issue 完了後に実施する空状態表示確認
- **注意**: 本 issue の修正は**すべて同一 PR で一括対応する**（seed 変更とテスト数値更新を分割すると、一方が先にマージされた時点で main の CI が壊れるため）

## 問題

SMK-084（空状態表示）を実施するには「レポート 0 件の Member ユーザー」でログインする必要があるが、現状の seed に存在するテナントAの Member は `test-member@example.com`（レポート 8 件）のみであり、0 件ユーザーが存在しない。

| ユーザー | テナント | レポート件数 |
|---|---|---|
| `test-member@example.com` | テナントA | 8 件（非ゼロ） |
| `test-member-b@example.com` | テナントB | 3 件（非ゼロ） |

どちらも非ゼロのため、SMK-084（空状態: レポートが 0 件のとき EmptyState コンポーネントが表示される）の手動確認が不可能。

## 採用方針（合意済み）

テナントA（Approver / Accounting / Admin が揃っており観察に支障なし）に、レポート 0 件の Member ユーザーを追加する。

| 項目 | 値 |
|---|---|
| メール | `test-member-empty@example.com` |
| ユーザー名 | `Test Member Empty` |
| ID 定数 | `UserMemberEmptyID = "aaaaaaaa-3333-3333-3333-000000000004"` |
| パスワード | `TestPass1!`（既存と同じ仮パスワード） |
| 所属テナント | テナントA（`TenantAID`） |
| ロール | `domain.RoleMember` |
| レポート | 0 件（追加なし） |

## 提案

### 1. `expense-saas/internal/seed/seed.go`

- **定数追加**: `UserMemberEmptyID = "aaaaaaaa-3333-3333-3333-000000000004"` を既存 `UserMemberBID` の直後に追加
- **`users` 配列**: `{UserMemberEmptyID, "test-member-empty@example.com", "Test Member Empty"}` を追加（既存 Member と同じハッシュ生成ロジックを使用）
- **`memberships` 配列**: `{TenantAID, UserMemberEmptyID, domain.RoleMember}` を追加

レポートは追加しない（0 件が前提のため）。

### 2. `dev-journal/deliverables/docs/60_test/test_strategy.md` §4.2

テナントAのユーザー表（161-168 行目付近）の既存 5 列構成（`ロール / ユーザー名 / メールアドレス / ユーザーID / テスト用パスワード`）を維持し、以下の行を追加する。

| ロール | ユーザー名 | メールアドレス | ユーザーID | テスト用パスワード |
|---|---|---|---|---|
| Member | Test Member Empty | `test-member-empty@example.com` | `aaaaaaaa-3333-3333-3333-000000000004` | `TestPass1!` |

「用途」列は既存メンバーに記載がないため新設しない。SMK-084 用である旨は本 issue と smoke_check.md SMK-084 行の対象ユーザー記述で担保する。

### 3. `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`

SMK-084 行（159 行目付近）の「前提データ」または「対象ユーザー」欄に `test-member-empty@example.com` と明記する。

### 4. `expense-saas/internal/handler/dashboard_handler_test.go` / `tenant_handler_test.go`

- DSH-015（dashboard_handler_test.go）: L510 付近のフィクスチャコメントと L528 のハードコード `4` を `5` に更新
- TNT-006（tenant_handler_test.go）: L167 のコメントと L192-193 のハードコード `4` を `5` に更新
- **TNT-008（tenant_handler_test.go、`TestListTenantMembers_ReturnsOwnTenantOnly`）**: L275 のコメント `（4 件）` → `（5 件）`、L276-281 の `tenantAMemberIDs` map に `testutil.UserMemberEmptyID: true,` を追加
- 過剰実装は避け、`4 → 5` の数値更新とホワイトリスト 1 行追加で対応する（`len([]string{...})` 化はしない）

### 5. `dev-journal/deliverables/docs/60_test/test_cases/dashboard.md` L110 / `test_cases/tenant.md` L83

- `test_cases/dashboard.md` L110（DSH-015）: 前提と期待値（`tenant_member_count == 4`）を **4 → 5** に更新する。
- `test_cases/tenant.md` L83（TNT-006）: 前提「テナントAに Admin, Approver, Member, Accounting の **4** ユーザーが登録済み」と期待値「`data` が配列で、**4** 件のメンバーが返ること」を **5** に更新する。
- DSH-015 / TNT-006 のテストコード（§4）と test_cases ドキュメントの値が乖離しないように同一 PR で揃える。

### 6. `expense-saas/README.md` L79-89

既存表構成（同じ列）を維持して「Test Member Empty」行を追加する。

### 7. `expense-saas/cmd/seed/main.go` L83-89

seed 完了 slog 出力のアカウント列挙に `test-member-empty@example.com` を追加する。

### 8. `expense-saas/internal/testutil/fixture.go`

L18-26 の定数再エクスポート群に `UserMemberEmptyID = seed.UserMemberEmptyID` を追加する（既存 `UserMemberBID = seed.UserMemberBID` と同パターン）。TNT-008 修正で `testutil.UserMemberEmptyID` を参照するために必要。

## 影響確認・テスト

### seed_test.go への影響

`expense-saas/internal/seed/seed_test.go` をグレップした結果、テナントAの Member 数や `tenant_memberships` の行数をアサートしている箇所は存在しない。

検証内容は以下の 4 項目のみ（テーブル参照: `expense_reports`）:

| テスト関数 | 検証内容 | 影響 |
|---|---|---|
| `TestSeed_PaidReportCount` | `status='paid'` の件数 >= 3 | なし（新規 Member のレポートは 0 件） |
| `TestSeed_PaidReportPeriodDistribution` | paid レポートの月別分散 | なし |
| `TestSeed_PaidReportTotalAmountPositive` | paid レポートの total_amount > 0 | なし |
| `TestSeed_Idempotent` | 2 回実行後にレポート件数が変化しない | なし（ON CONFLICT DO NOTHING により冪等） |

既存テスト改修は不要。冪等性は `TestSeed_Idempotent` の既存テストで自動検証される。

## 修正対象ファイル一覧

| ファイル | 変更種別 |
|---|---|
| `expense-saas/internal/seed/seed.go` | 定数追加・users エントリ追加・memberships エントリ追加 |
| `dev-journal/deliverables/docs/60_test/test_strategy.md` | §4.2 テナントAユーザー表に行追加 |
| `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` | SMK-084 の対象ユーザー明記 |
| `expense-saas/internal/handler/dashboard_handler_test.go` | DSH-015 期待値更新（`4` → `5`）+ フィクスチャコメント更新 |
| `expense-saas/internal/handler/tenant_handler_test.go` | TNT-006 期待値更新（`4` → `5`）+ コメント更新 + TNT-008 ホワイトリストへの `UserMemberEmptyID` 追加 + L275 コメント `（4 件）` → `（5 件）` |
| `expense-saas/internal/testutil/fixture.go` | `UserMemberEmptyID = seed.UserMemberEmptyID` 再エクスポート追加 |
| `dev-journal/deliverables/docs/60_test/test_cases/dashboard.md` | DSH-015 前提・期待値を 4 → 5 に更新 |
| `dev-journal/deliverables/docs/60_test/test_cases/tenant.md` | TNT-006 前提・期待値を 4 → 5 に更新 |
| `expense-saas/README.md` | テストアカウント表に「Test Member Empty」行追加 |
| `expense-saas/cmd/seed/main.go` | seed 完了 slog 出力に新規ユーザー追記 |

## MVP 区分

MVP（SMK-084 実施に必須）

## ラベル

- type: chore / test-data
- area: backend / docs

## 完了条件

- seed 投入後、`test-member-empty@example.com` でログインしレポート一覧に EmptyState が表示される
- `test_strategy.md` §4.2 の表に「Test Member Empty」行が追加されている
- `smoke_check.md` SMK-084 に `test-member-empty@example.com` が明記されている
- `seed_test.go` が引き続き通る（冪等性違反なし）
- `go test ./internal/handler/...` が通る（DSH-015 / TNT-006 の `4 → 5` 更新後の確認）
- `expense-saas/internal/testutil/fixture.go` に `UserMemberEmptyID = seed.UserMemberEmptyID` が追加されている
- TNT-008（`TestListTenantMembers_ReturnsOwnTenantOnly`）が新規ユーザー追加後も通る
- `test_cases/dashboard.md` DSH-015 の前提・期待値が 5 名前提に更新されている
- `test_cases/tenant.md` TNT-006 の前提・期待値が 5 名前提に更新されている
- `README.md` テストアカウント表に「Test Member Empty」行が追加されている
- `cmd/seed/main.go` の seed 完了 slog 出力に新規ユーザーが含まれている

## やらないこと

- seed_test.go への新規アサーション追加（既存テストで冪等性は担保される）
- テナントB への同種ユーザー追加（テナントA のみで SMK-084 は完結）

---

## 解決内容

PR #93（squash commit `d66a283`）で expense-saas 側 6 ファイルを修正:
- `internal/seed/seed.go`: `UserMemberEmptyID` 定数 + users / memberships エントリ追加
- `internal/testutil/fixture.go`: `UserMemberEmptyID = seed.UserMemberEmptyID` 再エクスポート
- `internal/handler/dashboard_handler_test.go`: DSH-015 期待値 4→5
- `internal/handler/tenant_handler_test.go`: TNT-006 期待値 4→5、TNT-008 ホワイトリストへ `UserMemberEmptyID` 追加
- `cmd/seed/main.go`: seed 完了 slog 出力に新規ユーザー追記
- `README.md`: テストアカウント表に「Test Member Empty」行追加

dev-journal 側 4 ファイルを commit `1b63b0f` / `2b22795` で修正:
- `deliverables/docs/60_test/test_strategy.md` §4.2: テナントAユーザー表に行追加
- `deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-084: 対象ユーザー明記
- `deliverables/docs/60_test/test_cases/dashboard.md` DSH-015 + L173: 5 名前提に更新
- `deliverables/docs/60_test/test_cases/tenant.md` TNT-006 + 擬似コードコメント: 5 名前提に更新

レビュー: 内部レビュー PASS（PR #93 / dev-journal 共に）+ codex レビュー PASS（dev-journal）+ codex レビュー REQUEST CHANGES（PR #93、テスト未実行指摘 → 指揮役判定で push back、マージ後 BE テストはユーザー実施）。

BE テストの実行確認はマージ後にユーザー側で実施予定。

## 解決日
2026-04-25

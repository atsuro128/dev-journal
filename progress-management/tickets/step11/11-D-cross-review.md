# 横断レビュー

- 担当: reviewer (codex)
- 依存: 11-B, 11-C
- ブランチ: なし
- 出力先: 進捗管理ディレクトリ内（レビューレポート）
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| Step 11 品質ゲート | ai-dev-framework/guide/work-breakdown/step11-system-test/review.md | 品質ゲート・レビュー観点 |
| テストケース定義 | deliverables/docs/60_test/test_cases/*.md | 全テストケース |
| 設計成果物全体 | deliverables/docs/ | API・DB・認可・セキュリティ・画面仕様 |
| 全機能の実装コード | expense-saas/internal/, expense-saas/frontend/src/ | 全体 |
| 横断テストコード | expense-saas/internal/handler/cross_cutting_test.go, rate_limit_test.go | 11-B の成果物 |
| E2E テストコード | expense-saas/e2e/ | 11-C の成果物 |

## 責務

- テスト品質・網羅性チェック（test_cases/*.md との対応）
- 全機能横断レビュー（セキュリティ・設計整合性・共通基盤の正しい使用）
- Step 8（基盤構築）で固定した共通契約が最終実装で崩れていないか確認
- ブロッカーがあれば修正してから次のタスクに進む
- 含めない: テストコードの実装・修正（指摘として起票するのみ）

## 進め方

1. 11-B / 11-C の実施ログ、テスト結果、未解決メモを確認する
2. `test_cases/*.md` と実装済みテストの対応を確認し、未カバー・重複・テスト目的のずれを記録する
3. 設計成果物と実装コードを横断し、セキュリティ・認可・状態遷移・共通基盤利用の逸脱を確認する
4. ブロッカー / 非ブロッカー / 要相談に分類し、ブロッカーは 11-E へ進む前に修正対象とする
5. レビューレポートに、合格判定・残リスク・11-E/11-F への引き継ぎを残す

## 重点レビュー観点

| 観点 | 確認内容 | 結果 | メモ |
|------|----------|------|------|
| 仕様設計とテストケースの対応 | 要件・画面仕様・API/DB/認可設計に対して、テストケースが不足・逸脱していないか | 未実施 | |
| テストケースとテスト実装の対応 | CRS ID・期待結果・前提データがテスト実装に正しく反映されているか | 未実施 | |
| テスト実装の妥当性 | 実装都合に合わせた弱いアサーション、モック過多、実害のある warning の見落としがないか | 未実施 | |
| テナント分離 | 他テナントリソースの参照・更新・添付アクセスが拒否されるか | 未実施 | |
| RBAC | Member / Approver / Accounting / Admin の許可・拒否が `authz.md` と一致するか | 未実施 | |
| 状態遷移 | 申請・承認・却下・再申請・支払完了の遷移が仕様通りか | 未実施 | |
| 添付ファイル | 署名付き URL、削除、プレビュー、ダウンロードが認可境界を越えないか | 未実施 | |
| 認証系 | JWT、リフレッシュ、ログアウト、パスワードリセットの扱いが設計通りか | 未実施 | |
| 共通基盤 | エラーハンドリング、API クライアント、ローディング、ログ、ヘルスチェックの共通契約が崩れていないか | 未実施 | |

## レビューレポート記録項目

- 判定: PASS / FAIL / CONDITIONAL PASS
- 確認したテスト結果: Go / frontend / E2E
- ブロッカー一覧
- 非ブロッカー一覧
- 仕様・テスト・実装の不整合一覧
- 11-E へ進む条件
- 11-F でユーザー確認が必要な残リスク

## 完了条件

- テスト網羅性が確認されている（横断テスト・E2E テストの全必須ケースが通過）
- 機能間の整合性が確認されている
- ブロッカーとなる問題がない（または解消済み）

---

## 実施結果（codex 11-D 横断レビュー、2026-05-07 CONDITIONAL PASS）

作成日: 2026-05-07 / 再レビュー日: 2026-05-07 / 担当: reviewer (codex)


### 1. 判定

**CONDITIONAL PASS**

初回 11-D のブロッカー B-01（JWT leeway 60s が Auth middleware 側に未適用）は、`expense-saas` master `59b268c` / `dev-journal` `b7adc3a` 時点で解消済みと判定する。`internal/pkg/jwt.Verifier.Verify` に `gojwt.WithLeeway(60*time.Second)` が追加され、`cmd/server/main.go` の保護 API 経路は `appjwt.NewVerifier(...)` を `middleware.Auth(verifier)` に渡す構成のため、リクエスト毎の JWT 検証にも leeway が適用される。

ただし、この再レビュー環境では test DB と起動済みアプリが無く、DB 付き統合テストと実 E2E は完走確認できていない。そのため、11-E 進入は可能だが、11-E の環境準備後に Go 統合テストと E2E smoke を再実行することを条件とする。

### 2. 確認したテスト結果

| 区分 | 結果 | メモ |
|---|---:|---|
| JWT middleware 経路単体 | PASS | `go test ./internal/pkg/jwt -count=100` PASS（57.196s）。AUTH-089〜092 の iat / exp 境界が flake なし |
| Auth/JWT 周辺 Go 単体 | PASS | `go test ./internal/pkg/jwt ./internal/domain ./internal/middleware ./internal/config` PASS |
| Go 全体再実行 | 環境要因で FAIL | `go test ./...` は `localhost:5433` の `expense_test` DB 不在により `internal/handler`, `internal/seed` が接続拒否で FAIL。今回修正に起因する失敗は確認されず |
| E2E config 確認 | PASS | `npm run e2e -- --list` が `e2e/playwright.config.ts` 経由で 10 件のみ列挙。frontend Vitest ファイル混入なし |
| E2E 実行 | 未実行 | 起動済み API / frontend / test DB が無いため未実行。11-E で CRS-055 / CRS-066 を含めて再確認する |

### 3. ブロッカー一覧

**現時点の未解消ブロッカー: なし**

#### B-01: JWT leeway 60s が Auth middleware 側に未適用

- 再レビュー判定: **解消済み**
- 対応: `internal/pkg/jwt/jwt.go` の `jwt.ParseWithClaims` に `jwt.WithLeeway(60*time.Second)` が追加済み。
- 経路確認: `cmd/server/main.go` は Auth middleware 用に `appjwt.NewVerifier(...)` を生成し、保護 API グループで `middleware.Auth(verifier)` を使用する。`middleware.Auth` は各リクエストで `verifier.Verify(tokenString)` を呼ぶため、middleware 経路のアクセストークン検証に leeway が効く。
- テスト確認: `internal/pkg/jwt/jwt_test.go` に AUTH-089〜092 が追加され、`iat +30s` 許可 / `iat +61s` 拒否 / `exp -30s` 許可 / `exp -61s` 拒否を検証している。
- 残確認: 実環境の時刻ドリフトを含む E2E 再現確認は 11-E の起動済み環境で実施する。

### 4. 非ブロッカー一覧

| ID | 分類 | 内容 | 再レビュー判断 |
|---|---|---|---|
| N-01 | 非ブロッカー | CRS-016 の横断テストが `my_draft_count <= 2` で、期待値の完全一致ではない | 前回通り非ブロッカー。DSH-018 の具体値テストと合わせて MVP 品質ゲートは阻害しない |
| N-02 | 非ブロッカー | CRS-071 は「添付はコピーされない」確認だが、元レポートに添付を付けず空配列を確認している | 前回通り非ブロッカー。添付あり元レポートでの強化は post-MVP 改善で扱う |
| N-03 | 非ブロッカー | CRS-077 の定義と実装経路の説明不足 | `cross-cutting.md` と `rate_limit_test.go` が `/health` で global IP 制限を検証する理由に整合済み。軽微扱いでよい |
| N-04 | 非ブロッカー | `npm run lint` は warning 1 件あり | 前回通り非ブロッカー。exit 0 の Fast Refresh warning のみ |
| N-05 | 非ブロッカー | `npm run build` が既存 `dist/` 権限で失敗 | 前回通り非ブロッカー。コード起因ではなく作業環境起因、クリーン outDir build 成功実績あり |

post-MVP は #081, #084, #104, #122, #133, #145, #146, #151, #167, #174 等の既存 issue に集約済みであり、11-D / 11-E のブロッカーにはしない。

### 5. 仕様・テスト・実装の不整合一覧

| ID | 分類 | 再レビュー判断 |
|---|---|---|
| M-01 | 解消済み | `security.md §2.1` は `domain.JWTVerifier`（service 経路）と `pkg/jwt.Verifier`（middleware 経路）の両系統で leeway 60s 適用と明記済み。実装・AUTH-081〜092 と整合 |
| M-02 | 非ブロッカー | CRS-071 の添付ありケース不足は残るが、post-MVP 改善扱いで妥当 |
| M-03 | 解消済み | `rate_limit_test.go` の `security.md §5.2` 参照は `§8.2` に訂正済み。CRS-077 の `/health` 使用理由も `cross-cutting.md` と一致 |
| M-04 | 解消済み | root `package.json` の `e2e` / `e2e:debug` が `--config e2e/playwright.config.ts` を指定済み。`e2e:report` は config 側の HTML reporter `outputFolder` と一致する `playwright-report` を参照する。`npm run e2e -- --list` で 10 件のみ列挙されることを確認 |

### 6. 11-E へ進む条件

1. 11-E 環境で test DB を起動し、少なくとも `go test ./internal/handler/... -tags=integration -run TestRateLimit` と主要 handler/seed 統合テストを再実行する。
2. 起動済み API / frontend に対して `npm run e2e` を実行し、CRS-055 / CRS-066 を含む 10 件が PASS することを記録する。
3. デプロイ環境でコンテナ / DB / ホスト時刻の大きなドリフトがないことを smoke で確認する。B-01 は実装上解消済みだが、時刻同期は運用前提として確認する。
4. frontend build は権限問題のないクリーン環境、または `dist/` 権限を直した環境で実行する。

### 7. 11-F でユーザー確認が必要な残リスク

- post-MVP 残リスク: #081, #084, #104, #122, #133, #145, #146, #151, #167, #174 は MVP ブロッカー扱いしない。UAT では既知制約として説明する。
- パスワードリセット: #151 によりメール送信は未実装。UAT ではデモ上の確認方法と本番運用時の未対応範囲を明示する。
- セッション期限切れ UX: #122 によりサイレントリダイレクトは既知の post-MVP 改善。ユーザーが許容できるか確認する。
- 添付ファイル: 署名付き URL、プレビュー、ダウンロード、削除の認可境界はテスト済みだが、CRS-071 の「添付あり再申請でコピーされない」自動検証は弱い。UAT で添付あり却下・再申請を確認するとよい。
- Admin 全レポート E2E: #174 により CRS-074 のアサーション強化は post-MVP。UAT では Admin / Accounting の全レポート一覧が実データで期待通り見えるか確認する。

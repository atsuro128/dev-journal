# Step 11-D 横断レビュー 再レビューレポート

作成日: 2026-05-07  
再レビュー日: 2026-05-07  
担当: reviewer (codex)

## 1. 判定

**CONDITIONAL PASS**

初回 11-D のブロッカー B-01（JWT leeway 60s が Auth middleware 側に未適用）は、`expense-saas` master `59b268c` / `dev-journal` `b7adc3a` 時点で解消済みと判定する。`internal/pkg/jwt.Verifier.Verify` に `gojwt.WithLeeway(60*time.Second)` が追加され、`cmd/server/main.go` の保護 API 経路は `appjwt.NewVerifier(...)` を `middleware.Auth(verifier)` に渡す構成のため、リクエスト毎の JWT 検証にも leeway が適用される。

ただし、この再レビュー環境では test DB と起動済みアプリが無く、DB 付き統合テストと実 E2E は完走確認できていない。そのため、11-E 進入は可能だが、11-E の環境準備後に Go 統合テストと E2E smoke を再実行することを条件とする。

## 2. 確認したテスト結果

| 区分 | 結果 | メモ |
|---|---:|---|
| JWT middleware 経路単体 | PASS | `go test ./internal/pkg/jwt -count=100` PASS（57.196s）。AUTH-089〜092 の iat / exp 境界が flake なし |
| Auth/JWT 周辺 Go 単体 | PASS | `go test ./internal/pkg/jwt ./internal/domain ./internal/middleware ./internal/config` PASS |
| Go 全体再実行 | 環境要因で FAIL | `go test ./...` は `localhost:5433` の `expense_test` DB 不在により `internal/handler`, `internal/seed` が接続拒否で FAIL。今回修正に起因する失敗は確認されず |
| E2E config 確認 | PASS | `npm run e2e -- --list` が `e2e/playwright.config.ts` 経由で 10 件のみ列挙。frontend Vitest ファイル混入なし |
| E2E 実行 | 未実行 | 起動済み API / frontend / test DB が無いため未実行。11-E で CRS-055 / CRS-066 を含めて再確認する |

## 3. ブロッカー一覧

**現時点の未解消ブロッカー: なし**

### B-01: JWT leeway 60s が Auth middleware 側に未適用

- 再レビュー判定: **解消済み**
- 対応: `internal/pkg/jwt/jwt.go` の `jwt.ParseWithClaims` に `jwt.WithLeeway(60*time.Second)` が追加済み。
- 経路確認: `cmd/server/main.go` は Auth middleware 用に `appjwt.NewVerifier(...)` を生成し、保護 API グループで `middleware.Auth(verifier)` を使用する。`middleware.Auth` は各リクエストで `verifier.Verify(tokenString)` を呼ぶため、middleware 経路のアクセストークン検証に leeway が効く。
- テスト確認: `internal/pkg/jwt/jwt_test.go` に AUTH-089〜092 が追加され、`iat +30s` 許可 / `iat +61s` 拒否 / `exp -30s` 許可 / `exp -61s` 拒否を検証している。
- 残確認: 実環境の時刻ドリフトを含む E2E 再現確認は 11-E の起動済み環境で実施する。

## 4. 非ブロッカー一覧

| ID | 分類 | 内容 | 再レビュー判断 |
|---|---|---|---|
| N-01 | 非ブロッカー | CRS-016 の横断テストが `my_draft_count <= 2` で、期待値の完全一致ではない | 前回通り非ブロッカー。DSH-018 の具体値テストと合わせて MVP 品質ゲートは阻害しない |
| N-02 | 非ブロッカー | CRS-071 は「添付はコピーされない」確認だが、元レポートに添付を付けず空配列を確認している | 前回通り非ブロッカー。添付あり元レポートでの強化は post-MVP 改善で扱う |
| N-03 | 非ブロッカー | CRS-077 の定義と実装経路の説明不足 | `cross-cutting.md` と `rate_limit_test.go` が `/health` で global IP 制限を検証する理由に整合済み。軽微扱いでよい |
| N-04 | 非ブロッカー | `npm run lint` は warning 1 件あり | 前回通り非ブロッカー。exit 0 の Fast Refresh warning のみ |
| N-05 | 非ブロッカー | `npm run build` が既存 `dist/` 権限で失敗 | 前回通り非ブロッカー。コード起因ではなく作業環境起因、クリーン outDir build 成功実績あり |

post-MVP は #081, #084, #104, #122, #133, #145, #146, #151, #167, #174 等の既存 issue に集約済みであり、11-D / 11-E のブロッカーにはしない。

## 5. 仕様・テスト・実装の不整合一覧

| ID | 分類 | 再レビュー判断 |
|---|---|---|
| M-01 | 解消済み | `security.md §2.1` は `domain.JWTVerifier`（service 経路）と `pkg/jwt.Verifier`（middleware 経路）の両系統で leeway 60s 適用と明記済み。実装・AUTH-081〜092 と整合 |
| M-02 | 非ブロッカー | CRS-071 の添付ありケース不足は残るが、post-MVP 改善扱いで妥当 |
| M-03 | 解消済み | `rate_limit_test.go` の `security.md §5.2` 参照は `§8.2` に訂正済み。CRS-077 の `/health` 使用理由も `cross-cutting.md` と一致 |
| M-04 | 解消済み | root `package.json` の `e2e` / `e2e:debug` が `--config e2e/playwright.config.ts` を指定済み。`e2e:report` は config 側の HTML reporter `outputFolder` と一致する `playwright-report` を参照する。`npm run e2e -- --list` で 10 件のみ列挙されることを確認 |

## 6. 11-E へ進む条件

1. 11-E 環境で test DB を起動し、少なくとも `go test ./internal/handler/... -tags=integration -run TestRateLimit` と主要 handler/seed 統合テストを再実行する。
2. 起動済み API / frontend に対して `npm run e2e` を実行し、CRS-055 / CRS-066 を含む 10 件が PASS することを記録する。
3. デプロイ環境でコンテナ / DB / ホスト時刻の大きなドリフトがないことを smoke で確認する。B-01 は実装上解消済みだが、時刻同期は運用前提として確認する。
4. frontend build は権限問題のないクリーン環境、または `dist/` 権限を直した環境で実行する。

## 7. 11-F でユーザー確認が必要な残リスク

- post-MVP 残リスク: #081, #084, #104, #122, #133, #145, #146, #151, #167, #174 は MVP ブロッカー扱いしない。UAT では既知制約として説明する。
- パスワードリセット: #151 によりメール送信は未実装。UAT ではデモ上の確認方法と本番運用時の未対応範囲を明示する。
- セッション期限切れ UX: #122 によりサイレントリダイレクトは既知の post-MVP 改善。ユーザーが許容できるか確認する。
- 添付ファイル: 署名付き URL、プレビュー、ダウンロード、削除の認可境界はテスト済みだが、CRS-071 の「添付あり再申請でコピーされない」自動検証は弱い。UAT で添付あり却下・再申請を確認するとよい。
- Admin 全レポート E2E: #174 により CRS-074 のアサーション強化は post-MVP。UAT では Admin / Accounting の全レポート一覧が実データで期待通り見えるか確認する。

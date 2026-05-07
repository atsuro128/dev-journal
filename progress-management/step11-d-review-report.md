# Step 11-D 横断レビューレポート

作成日: 2026-05-07  
担当: reviewer (codex)

## 1. 判定

**FAIL**

11-B / 11-C の完了実績と、テナント分離・RBAC・状態遷移・添付ファイル認可の大枠は確認できた。一方で、CRS-066 の真因として修正済み前提になっている **JWT leeway 60s が本番の Auth middleware 側のアクセストークン検証器に適用されていない**。このまま 11-E に進むと、Docker / ホスト / デプロイ先の時刻ずれで保護 API が再度 401 になるリスクが残るため、11-E 進行前のブロッカーと判定する。

## 2. 確認したテスト結果

| 区分 | 結果 | メモ |
|---|---:|---|
| 11-A ローカル動作確認 | PASS 前提 | `tickets/step11/11-A-local-verification.md` 上で SMK 62 件 PASS、FAIL 0 を確認 |
| 11-B Go 横断テスト | PASS 前提 | PR #139 マージ済み。CRS-001〜054 / CRS-076〜088 の実装・通過実績を進捗ログで確認 |
| 11-C E2E | PASS 前提 | PR #138 マージ済み。E2E 10/10 PASS、CRS-066 は PR #141 で解消済み前提と記録されている |
| Go ローカル再実行 | 部分 PASS | `go test ./internal/domain ./internal/pkg/jwt ./internal/middleware ./internal/config` PASS |
| Go 全体再実行 | 未完了 | `go test ./...` は `localhost:5433` の test DB 不在で FAIL。現在環境に `docker` が無く、統合テスト再実行不可 |
| Frontend 型チェック | PASS | `npx tsc --noEmit` PASS |
| Frontend unit | PASS | `npm test -- --run`: 103 files / 797 tests PASS |
| Frontend lint | PASS with warning | `npm run lint` exit 0。`ReportForm.tsx` の Fast Refresh warning 1 件 |
| Frontend build | 条件付き PASS | `npm run build` は既存 `dist/` が root 所有で EACCES。`npx vite build --outDir /tmp/...` は PASS |
| E2E ローカル再実行 | 未実行 | Docker / 起動済み app 不在。加えて root の `npm run e2e` は config 未指定で frontend tests まで拾い FAIL |

## 3. ブロッカー一覧

### B-01: JWT leeway 60s が Auth middleware 側に未適用

- 分類: **ブロッカー**
- 対象: `expense-saas/internal/pkg/jwt/jwt.go`, `expense-saas/internal/domain/auth.go`, `expense-saas/cmd/server/main.go`
- 内容: `security.md §2.1` は `exp` / `iat` / `nbf` に leeway 60s を適用すると定義している。`internal/domain/auth.go` の `JWTVerifier` には `WithLeeway(60*time.Second)` があるが、実際の保護 API で `middleware.Auth` が使う `internal/pkg/jwt.Verifier` には leeway が無い。
- 根拠: `cmd/server/main.go` は Auth middleware 用に `appjwt.NewVerifier(...)` を使う。該当 verifier の `ParseWithClaims` は `WithIssuedAt`, `WithIssuer`, `WithExpirationRequired` のみで、`WithLeeway` がない。
- 影響: CRS-066 の真因だった「トークン発行後にサーバ時刻が巻き戻り、`iat` が未来扱いで 401」への恒久対策が、保護 API 経路では成立していない可能性が高い。11-E のデプロイ環境でも主要フローが flake / 401 になる。
- 11-E 前の必須対応: `internal/pkg/jwt.Verifier` に leeway を適用し、middleware 経由のアクセストークンで `iat +30s` 許可 / `iat +61s` 拒否 / `exp -30s` 許可 / `exp -61s` 拒否をテスト追加する。その後 CRS-066 を含む E2E を再実行する。

## 4. 非ブロッカー一覧

| ID | 分類 | 内容 | 判断 |
|---|---|---|---|
| N-01 | 非ブロッカー | CRS-016 の横断テストが `my_draft_count <= 2` で、期待値の完全一致ではない | DSH-018 の具体値テストと合わせて概ね担保。ただし横断テスト単体のアサーションは弱い |
| N-02 | 非ブロッカー | CRS-071 は「添付はコピーされない」確認だが、元レポートに添付を付けず空配列を確認している | 実害は限定的だが、添付あり元レポートでの非コピー確認に強化余地あり |
| N-03 | 非ブロッカー | CRS-077 の定義は `POST /api/auth/login` 21 回だが、実装は `/health` で global IP 制限を検証している | login 専用制限との分離意図は合理的。テスト名またはテストケース定義の整合化が望ましい |
| N-04 | 非ブロッカー | `npm run lint` は warning 1 件あり | exit 0 のため品質ゲートは阻害しない。Fast Refresh warning のみ |
| N-05 | 非ブロッカー | `npm run build` が既存 `dist/` の権限で失敗 | コード起因ではなく作業環境起因。クリーン環境相当の別 outDir build は成功 |

## 5. 仕様・テスト・実装の不整合一覧

| ID | 分類 | 不整合 |
|---|---|---|
| M-01 | **ブロッカー** | `security.md` と #173/#141 の完了記録は JWT leeway を全クレーム・全トークンで適用済みとしているが、保護 API の Auth middleware は `internal/pkg/jwt.Verifier` を使い、leeway 未適用 |
| M-02 | 非ブロッカー | CRS-071 の期待結果「元レポートの添付はコピーされていないこと」に対し、E2E は元レポートに添付を作成していない |
| M-03 | 非ブロッカー | CRS-077 の入力定義と `rate_limit_test.go` の実リクエスト対象が異なる |
| M-04 | 要相談 | root `expense-saas/package.json` の `e2e` script が `e2e/playwright.config.ts` を指定しておらず、`npm run e2e -- --list` が frontend の Vitest ファイルまで拾って失敗する。11-C では手動で `--config` 指定して通しており、post-MVP 改善候補として扱われているが、11-E の手順書では明示が必要 |

## 6. 11-E へ進む条件

1. B-01 を修正し、Auth middleware 経由の JWT leeway テストを追加して PASS させる。
2. 修正後に Go 統合テストを test DB ありで再実行し、少なくとも `cross_cutting_test.go`, `rate_limit_test.go`, `auth_test.go` が PASS していることを記録する。
3. `npx playwright test --config e2e/playwright.config.ts` で CRS-055 / CRS-066 を再実行し、CRS-066 が再発しないことを確認する。
4. 11-E 手順では root の `npm run e2e` ではなく config 指定コマンドを使う、または npm script を修正する。
5. デプロイ前に `dist/` 権限問題を解消するか、クリーン build 環境で frontend build を実行する。

## 7. 11-F でユーザー確認が必要な残リスク

- post-MVP 残リスク: #081, #084, #104, #122, #133, #145, #146, #151, #167, #174 は MVP ブロッカー扱いしない。UAT では既知制約として説明する。
- パスワードリセット: #151 によりメール送信は未実装。UAT ではデモ上の確認方法と本番運用時の未対応範囲を明示する。
- セッション期限切れ UX: #122 によりサイレントリダイレクトは既知の post-MVP 改善。ユーザーが許容できるか確認する。
- 添付ファイル: 署名付き URL、プレビュー、ダウンロード、削除の認可境界はテスト済みだが、CRS-071 の「添付あり再申請でコピーされない」自動検証は弱い。UAT で添付あり却下・再申請を確認するとよい。
- Admin 全レポート E2E: #174 により CRS-074 のアサーション強化は post-MVP。UAT では Admin / Accounting の全レポート一覧が実データで期待通り見えるか確認する。
- デプロイ環境の時刻同期: B-01 修正後も、11-E ではコンテナ / DB / ホスト時刻のドリフトが主要フローに影響しないことをスモークで確認する。

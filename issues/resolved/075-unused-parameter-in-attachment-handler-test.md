# attachment_handler_test.go の addAuthHeader に未使用パラメータ

## 発見日
2026-04-12

## カテゴリ
implementation

## 影響度
低

## 発見経緯
proactive

## 関連ステップ
Step 9（テストコード実装）/ Step 10（機能実装）

## ブロッカー
なし

## 問題

`expense-saas/internal/handler/attachment_handler_test.go:217` の `addAuthHeader` 関数に未使用パラメータ `srv *testutil.TestServer` がある。Go の静的解析（unusedparams）で警告が出ている。

```go
func addAuthHeader(t *testing.T, srv *testutil.TestServer, req *http.Request, userID, tenantID, role string) *http.Request {
    t.Helper()

    token := testutil.GenerateTestToken(t, userID, tenantID, role)
    req.Header.Set("Authorization", "Bearer "+token)
    return req
}
```

`srv` は関数内で一切参照されていない。`testutil.GenerateTestToken` は `t` のみを引数に取っており、`srv` に依存していない。

## 影響

- 実害はない（機能・テスト結果に影響なし）
- コード品質の観点では削除すべき
- 同種の未使用パラメータが他のテストファイルにも潜在している可能性あり

## 提案

1. `attachment_handler_test.go:217` の `addAuthHeader` から `srv` パラメータを削除
2. 呼び出し側（同ファイル内の複数箇所）も併せて修正
3. 他のテストファイル・実装コードに同種の警告がないか静的解析で確認し、まとめて対処するか判断
4. CI で静的解析（unusedparams 等）を失敗扱いにするか検討（現状は警告のみで通過している）

---

## 解決内容

### 実施内容
- `internal/handler/attachment_handler_test.go:217` の関数定義から `srv *testutil.TestServer` 引数を削除
- 同ファイル内の 18 箇所の呼び出しから `srv` 引数を削除（機械的置換）
- 関数コメントを実態に合わせて更新（TestServer への言及を削除）

### 提案 #3 の調査結果
他のテストファイルで `*testutil.TestServer` を引数に取るヘルパーは `internal/handler/auth_test.go:76` の `loginAndGetTokens` のみだったが、こちらは関数内の 89 行目で `srv.Execute(req)` を実際に使用しているため**未使用ではなく、追加対応不要**。

### 提案 #4 の扱い
CI での静的解析（unusedparams）を失敗扱いにする検討は本 PR のスコープ外として保留。別途検討が必要なら issue として起票する。

### PR & レビュー結果
- **PR**: https://github.com/atsuro128/expense-saas/pull/46（commit `73956ff`）
- **CI**: 全 6 ジョブ PASS（Lint / Test / Build × BE / FE）
- **内部 reviewer**: PASS（blocker 0、warning 0、指摘ゼロ）
- **codex レビュー**: PASS（指摘なし、スコープ管理・コメント整合性・呼び出し箇所の更新漏れ全てクリア）
- **マージ**: `gh pr merge 46 --squash --delete-branch` で master にマージ済み（master 最新コミット `8e86f4f`）

## 解決日
2026-04-13

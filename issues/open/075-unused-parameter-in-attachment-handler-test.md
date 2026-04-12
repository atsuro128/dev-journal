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

## 解決日

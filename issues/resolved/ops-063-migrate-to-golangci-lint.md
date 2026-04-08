# staticcheck-action を golangci-lint に移行する

## 発見日
2026-04-08

## カテゴリ
infrastructure

## 影響度
低

## 発見経緯
proactive

## 関連ステップ
Step 8（基盤構築）

## ブロッカー
なし

## 問題
`dominikh/staticcheck-action@v1` と `actions/setup-go@v5` が同じ Go キャッシュディレクトリを二重に復元しようとし、tar が "Cannot open: File exists" で失敗する。現在は `install-go: false` + `use-cache: false` で回避しているが、staticcheck のキャッシュが効かない状態。

refs: actions/setup-go#506, actions/setup-go#403, actions/setup-go#722

## 影響
- staticcheck の実行が毎回フルスキャンになり CI が遅くなる
- go vet と staticcheck が別ステップで管理されており、リンター管理が分散している

## 提案
`golangci/golangci-lint-action@v9` に移行し、staticcheck + go vet を一元管理する。

1. `.golangci.yml` を作成し、有効リンターを定義（govet, staticcheck）
2. CI の lint-backend ジョブを golangci-lint-action に置き換え
3. staticcheck-action のステップを削除
4. キャッシュは golangci-lint-action が独自に管理（setup-go との競合なし）

---

## 解決内容
PR #20 で `golangci/golangci-lint-action@v9` に移行。`.golangci.yml`（version: 2）で govet + staticcheck を有効化。CI 全ジョブ PASS 確認済み。

## 解決日
2026-04-08

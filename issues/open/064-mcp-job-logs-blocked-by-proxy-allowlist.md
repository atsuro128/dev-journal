# GitHub MCP Server の get_job_logs が proxy allowlist でブロックされる

## 発見日
2026-04-10

## カテゴリ
infrastructure

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 10（機能実装）

## ブロッカー
なし

## 問題

GitHub MCP Server の `get_job_logs` ツールで CI ジョブログをダウンロードしようとすると、Azure Blob Storage URL へのリクエストが devcontainer の Squid proxy でブロックされ Forbidden エラーになる。

```
failed to download logs: Get "https://productionresultssa9.blob.core.windows.net/...": Forbidden
```

`proxy-allowlist.txt` には `productionresultssa15.blob.core.windows.net` のみ登録されているが、GitHub Actions は複数の Azure Blob Storage エンドポイント（`productionresultssa9`, `productionresultssa15` 等）にリダイレクトするため、番号が異なるとブロックされる。

`gh run view --log-failed` でも同じリダイレクト先を使うが、たまたま allowlist 内のサーバーに当たれば成功し、外れれば同様に失敗する（不安定）。

## 影響

- MCP 経由での CI ログ取得が不安定になり、`gh` CLI へのフォールバックが必要になる
- `gh` CLI でも同じ問題が起きる可能性がある（リダイレクト先のサーバー番号に依存）

## 提案

`render-squid-config.sh` にワイルドカード構文（例: `*.blob.core.windows.net`）のサポートを追加し、`proxy-allowlist.txt` の `productionresultssa15.blob.core.windows.net` を `*.blob.core.windows.net` に置き換える。

現在のスクリプトはホスト名を完全一致で検証（68行目: `^[a-z0-9.-]+$`）し、正規表現に変換している。ワイルドカードエントリを検出した場合に適切な正規表現（`^(https?://)?[a-z0-9-]+\.blob\.core\.windows\.net(:[0-9]+)?(/|$)`）を生成するように拡張する。

関連: issue 060（egress allowlist の選定根拠と検証厳格性の不整合）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

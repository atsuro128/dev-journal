# devcontainer の golangci-lint バージョンがプロジェクト設定と不一致

## 発見日
2026-04-16

## カテゴリ
infrastructure

## 影響度
中（ローカルテスト実行時に lint が通らない）

## 発見経緯
user-report — PR #60 の backend lint 実行時に発覚。devcontainer にインストール済みの golangci-lint v1.64.8 が、プロジェクトの `.golangci.yml`（version: "2"）と互換性がなくエラー終了した。

## 関連ステップ
Step 8（基盤構築）/ Step 11-A

## ブロッカー
なし

## 問題

devcontainer に golangci-lint v1 がインストールされているが、プロジェクトの `.golangci.yml` は v2 形式で記述されている。

```
$ golangci-lint version
golangci-lint has version v1.64.8

$ head -2 .golangci.yml
version: "2"
```

`golangci-lint run ./...` を実行すると以下のエラーで即座に失敗する:

```
Error: you are using a configuration file for golangci-lint v2 with golangci-lint v1
```

## 影響

- `/test` スキルの backend lint が実行できない
- ローカルテストフローが不完全になる

## 提案

`.devcontainer/` の設定を見直し、golangci-lint v2 をインストールするようにする。

確認すべき箇所:
- `Dockerfile` または `devcontainer.json` の features で golangci-lint のバージョン指定
- 他のツール（go, node 等）のバージョンも現行プロジェクトと整合しているか併せて確認

## 完了条件

- devcontainer rebuild 後に `golangci-lint version` が v2 系を返す
- `golangci-lint run ./...` が `.golangci.yml` v2 設定で正常動作する

---

## 解決内容

`.devcontainer/Dockerfile` と `.devcontainer/devcontainer.json` の golangci-lint バージョンを v1.64.8 → v2.11.4 に更新。`go install` のモジュールパスも v2 用（`.../golangci-lint/v2/cmd/...`）に変更。

### 完了条件の確認結果

- `golangci-lint version` → `v2.11.4`（Go 1.24.1 環境で toolchain auto-switch により go1.25.9 で動作）
- `golangci-lint run ./...` → `0 issues.`（`.golangci.yml` v2 設定で正常動作）

devcontainer rebuild は未実施（現在のコンテナは手動インストール済みの v2.11.4 で動作中）。次回 rebuild 時に Dockerfile の変更が反映される。

## 解決日
2026-04-16

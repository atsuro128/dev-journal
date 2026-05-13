# Go バージョン定義の環境間不整合（devcontainer / production build / test-be、post-MVP）

## 発見日
2026-05-13

## カテゴリ
infrastructure / build-reproducibility / post-mvp

## 影響度
低（現状は実害なし。Go の minor 1.24.x 互換性で動作している。リリース品質・説明可能性の弱さ）

## 発見経緯
proactive — Step 11-E Phase 0 着手時、`BE: full test` タスク実行手順を確認する過程で、Go バージョン定義の所在を点検した結果検出。

## 関連ステップ
- Step 11-E Phase 0 着手時に検出
- 対応は MVP リリース後（infra 整備スプリント）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外として扱う。MVP リリース前には対応しない。管理方式は **ops-080** で検討中。

## 問題

Go バージョンが 4 箇所で別々に定義されており、整合の保証が image / minor バージョンの互換性に暗黙に依存している。

### 現状の Go バージョン定義

| 場所 | 定義 | バージョン | 性質 |
|------|------|----------|------|
| devcontainer | `.devcontainer/Dockerfile` の `ARG GO_VERSION=1.24.1` | 1.24.1 | 明示固定 |
| production build | `expense-saas/Dockerfile` の `FROM golang:1.24-alpine` | 1.24-alpine | パッチ浮動（Docker Hub の最新 1.24.x） |
| go.mod | `expense-saas/go.mod` の `go 1.24.0` | 1.24.0 以上 | 最小要件のみ |
| test-be (compose) | `docker-compose.yml` の `image: golangci/golangci-lint:v2.11.4` | image 同梱の Go（外形不明） | 暗黙依存 |

### 想定リスク

1. **test-be image の Go 同梱バージョンが将来変わる**
   - `golangci/golangci-lint:v2.11.4` は固定タグだが、同タグでも image rebuild により同梱 Go が動く可能性
   - 仮に Go 1.23.x にダウングレードされれば go.mod の `go 1.24.0` 最小要件を下回りビルド不能
   - 仮に Go 1.25.x にアップグレードされれば production と挙動が乖離する可能性

2. **production build の `golang:1.24-alpine` がパッチ浮動**
   - Docker Hub のタグ更新で 1.24.1 → 1.24.X に勝手に上がる
   - test 環境（devcontainer + test-be）と production でパッチがズレる

3. **devcontainer の `GO_VERSION=1.24.1` 固定だけが孤立**
   - 他環境は固定されていないため、整合の保証は実行時のたまたまの一致に依存

### 現実害

なし。Go 1.24.x は backward compatibility が強く、minor 内のパッチ差で動作差が出ることはほぼない。ただし「契約として保証されていない」状態。

## 影響

### MVP リリース判定

なし。Step 11-E（デプロイ・スモークテスト）のブロッカーにならない。

### CI 再現性

- 全環境を 1.24.1 で揃えれば、production build / test / devcontainer の動作が完全に一致する
- 現状はそれが「ほぼ揃っている」状態（保証なし）

### 採用面接・ポートフォリオ品質

「依存固定どうしてる？」「test 環境と production の Go バージョンは？」と問われた際の説明:
- 現状: 「Go 1.24 系で揃えているが、test-be は image 同梱バージョンに依存」
- 目標: 「全環境 1.24.X で完全固定、Dockerfile で明示インストール」

## 提案

### 対応タイミング

**MVP リリース後**（Step 11-F UAT 完了後）。infra 整備スプリントの一部として扱う。

### 対応プラン

#### 案 B: test 専用 Dockerfile を作る（推奨）

1. `expense-saas/test.Dockerfile` を新規作成:
   ```dockerfile
   FROM golang:1.24.1-alpine
   ARG GOLANGCI_LINT_VERSION=v2.11.4
   RUN apk --no-cache add bash git curl ca-certificates
   RUN go install github.com/golangci/golangci-lint/cmd/golangci-lint@${GOLANGCI_LINT_VERSION}
   ```

2. `docker-compose.yml` の `test-be` サービス修正:
   ```yaml
   test-be:
     build:
       context: .
       dockerfile: test.Dockerfile
     # image: golangci/golangci-lint:v2.11.4  ← 削除
     profiles: [test]
     ...
   ```

3. `expense-saas/Dockerfile` の `FROM golang:1.24-alpine` を `FROM golang:1.24.1-alpine` に固定

4. `.devcontainer/Dockerfile` の `ARG GO_VERSION=1.24.1` はそのまま（既に整合）

5. `go.mod` の `go 1.24.0` も `go 1.24.1` に更新（最小要件と実バージョンを一致させる）

これで全環境が **Go 1.24.1** で完全固定。

#### 案 C: すべて参照する単一ソースを作る

`expense-saas/.go-version` ファイルに `1.24.1` と書き、各 Dockerfile / compose で ARG/build-arg 経由で参照。複雑度は上がるが Single Source of Truth が確立する。

ポートフォリオ用途では案 B で十分。

### 検証手順

1. 修正後 `BE: full test` で全件 PASS を確認
2. production build (`docker build -f Dockerfile .`) で正常ビルドを確認
3. devcontainer rebuild は不要（既に 1.24.1）
4. `docker compose --profile test run --rm test-be go version` で 1.24.1 を確認

### 受容方針（MVP 期間中）

現状のまま MVP リリースする。説明可能な状態を維持:
- issue #178 で対応プラン記録済み
- 「Go 1.24 系で実質統一されているが、契約として固定するのは post-MVP」と説明

## 関連 issue / PR

- 関連: ops-080（Post-MVP スコープ管理方法）— 本 issue の管理方式
- 関連: Step 11-E チケット（Phase 0 着手時に検出）

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

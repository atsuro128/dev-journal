# SPA embed 配信が未実装（architecture.md §4.0 と実装の乖離、Phase 7 ブロッカー）

## 発見日
2026-05-20

## カテゴリ
implementation

## 影響度
高

## 発見経緯
proactive

## 関連ステップ
Step 10（機能実装）/ Step 8（基盤構築）/ Step 11-E Phase 7（スモークテスト）

## ブロッカー
なし（先行で解決必須の issue はない）

## 問題

`architecture.md` §4.0 で確定した「**Go コンテナに Vite build 成果物を `go:embed` で同梱して配信**」する設計が、実装側に反映されていない。

具体的な乖離:

| 観点 | 設計（architecture.md §4.0） | 現状実装 |
|------|-----------------------------|---------|
| Dockerfile | frontend build stage を含み、`frontend/dist/` を Go builder stage にコピーする | frontend build stage が**存在しない**（backend のみ。50.1MB image） |
| Go コード | `//go:embed frontend/dist` で静的ファイルを埋め込み | embed ディレクティブ**なし**（`grep -rn "//go:embed" cmd/ internal/` で唯一ヒットするのは `seed.go:23` の receipt_sample.jpg のみ） |
| ルーティング | `/api/*` → API ハンドラ、`/health` → ヘルスチェック、その他 → SPA fallback (`index.html`) | `/api/*` + `/health` のみ。SPA fallback ハンドラ**なし** |

### 検出経緯

Step 11-E Phase 5 で `cd ~/expense-saas && sudo docker build -t expense-saas:portfolio .` を EC2 上で実行した際、ビルドが約 1.5 分（うち Go build 59 秒）で完走し、image size が 50.1MB（alpine + Go バイナリ 2 個のみ）と判明。frontend を含む構成なら 100MB 超 + Node build stage で 5-10 分はかかる想定だったため、Dockerfile を確認して frontend stage の欠落を発見した。

## 影響

1. **Step 11-E Phase 7（スモークテスト）が完了不可**
   - 11-E チケット §6.3.2「ブラウザで `http://<alb-dns>/` を開く → ログイン画面が表示される」が不成立
   - ALB → backend に直結する構成では、`/` リクエストは backend のルーティングテーブルに該当せず 404 or 別の挙動になる（SPA fallback ハンドラがないため）

2. **Step 11-F（UAT）も実質完了不可**
   - UAT は uat_check.md に従い**ブラウザで全画面**を確認する工程
   - frontend が配信されない状態では UAT 着手不能

3. **本番デプロイ後の動作確認手段がない**
   - 現状 ALB DNS でアクセス可能なのは `/health` の JSON 200 OK のみ
   - エンドユーザー視点で「SaaS として動いている」状態に到達できない

## 提案

### A 案: 設計通りに `go:embed` 方式で実装（推奨）

Step 10 機能実装 / Step 8 基盤構築の**実装漏れ**として扱い、設計通りに修正する。

**修正範囲**:
1. **Dockerfile**: Node build stage を追加
   - `FROM node:20-alpine AS frontend-builder`
   - `WORKDIR /build/frontend`
   - `COPY frontend/package*.json ./`
   - `RUN npm ci`
   - `COPY frontend/ ./`
   - `RUN npm run build`（`/build/frontend/dist/` に成果物生成）
2. **Dockerfile**: Go builder stage で frontend dist をコピー
   - `COPY --from=frontend-builder /build/frontend/dist /build/frontend/dist`
3. **Go コード**: SPA embed + fallback ハンドラ追加
   - `cmd/server/main.go`（または専用の `internal/spa/` パッケージ）に以下を追加:
     - `//go:embed frontend/dist` + `embed.FS`
     - `http.FileServer` + index.html fallback ハンドラ（chi router の NotFound or `Mount("/", ...)` で実装）
   - ルーティング順序: `/api/*` API ハンドラ → `/health` ハンドラ → その他 SPA fallback
4. **テスト**: SPA fallback の最小限の動作確認テスト追加
5. **Phase 5 やり直し**: EC2 上で再 build → systemd restart → ALB 経由でブラウザアクセス可能化

**工数見積**: 4-6h（Dockerfile 改修 1h + Go embed 実装 1-2h + テスト 1h + 再デプロイ 1-2h）

**メリット**:
- ADR-0004 / architecture.md §4.0 と整合
- 余計なインフラ追加なし（S3 / CloudFront / nginx 不要）
- 1 コンテナで完結、運用シンプル
- portfolio として「設計通りに実装した」と説明可能

### B 案: ADR を立て直して別方式に切替

S3 + CloudFront / nginx コンテナ追加 / ALB に listener rule 追加 / 等の別配信方式を新 ADR で正式化する。

**デメリット**:
- ADR-0004 の見直し → 設計成果物の改訂 → レビューが必要
- インフラ追加コスト（CloudFront / S3 容量 / nginx メモリ消費）
- 工数 1-2 日

### C 案: SPA 配信は post-MVP に退避

Phase 7 / 11-F を「API のみ確認」で完了扱いにし、SPA 配信は別 issue として post-MVP に倒す。

**デメリット**:
- UAT が実質不能 → MVP 完了基準を満たさない
- 「SaaS をデプロイした」と言える状態に到達しない
- portfolio の意義が大きく毀損

### 推奨

**A 案**を推奨。設計乖離の解消であり、追加スコープではない。Phase 7 を続けるための前提条件と位置付ける。

---

## 解決内容

A 案（設計通り `go:embed` 方式で実装）を採用し、PR #148 で対応・マージ済み（master `1739399`）。

### 実装内容

- **Dockerfile**: `node:20-alpine` の frontend-builder stage を追加（`npm ci` → `npm run build`）。Go builder stage で `COPY --from=frontend-builder /build/frontend/dist ./cmd/server/frontend/dist`
- **`cmd/server/embed.go`**: `//go:embed all:frontend/dist` で Vite build 成果物を埋め込み（`..` 禁止制約のため `cmd/server/frontend/dist` 配置）
- **`internal/spa/handler.go`**: SPA fallback ハンドラ。未定義 `/api/*` は `middleware.RespondError` で標準 JSON 404（`RESOURCE_NOT_FOUND`）、拡張子あり未定義 → 404、拡張子なし → `index.html` fallback
- **`cmd/server/main.go`**: `/api/*` / `/health` 登録後に SPA fallback を登録（既存ルート優先）
- **`internal/spa/handler_test.go`**: chi router ベースでルーティングを検証（回帰防止アサーション含む）

### レビュー経緯

- 内部レビュー（reviewer）3 ラウンド: blocker-1（未定義 `/api/*` が index.html を 200 で返す）、blocker-2（テストが `http.ServeMux` で偽 PASS）、blocker-3（404 エラーコードがプロジェクト標準と不一致）を全て解消し PASS
- codex レビュー: PASS / マージ可

### 残課題

新 Dockerfile の **実 `docker build`（multi-stage）は未検証**。reviewer / codex はいずれも「成果物生成・配置・Go build の流れ」までの確認で、実 `docker build` は実行していない。Step 11-E Phase 5 やり直し（EC2 上で新イメージをビルド + デプロイ）で実地検証する。

## 解決日
2026-05-20

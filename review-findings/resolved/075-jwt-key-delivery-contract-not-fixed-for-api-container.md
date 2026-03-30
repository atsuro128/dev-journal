# 075: JWT 鍵の供給方法が API コンテナで未固定

## 指摘概要
`JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH` は設定契約として定義されているが、`api` コンテナに鍵ファイルをどう供給するかが 8-2 の成果物内で固定されていない。現状の `Dockerfile` は実行バイナリしかイメージに含めず、`docker-compose.yml` も `keys/` をマウントしていないため、8-4 で JWT 認証ミドルウェアが鍵読み込みを実装した時点で `docker compose up` の起動契約が壊れる。

## 根拠
- `expense-saas/docker-compose.yml:27-34` では `JWT_PRIVATE_KEY_PATH=./keys/private.pem`、`JWT_PUBLIC_KEY_PATH=./keys/public.pem` を `api` サービスへ渡している。
- `expense-saas/Dockerfile:1-11` ではランタイムイメージに `/build/server` しかコピーしておらず、`/app/keys` を作成・コピーしていない。
- `expense-saas/.gitignore:34-35` では `keys/` を Git 管理外としており、イメージ build context に常に存在する前提も置けない。
- `expense-saas/scripts/generate-keys.sh:4-7` には鍵生成手段があるが、`docker-compose.yml` / `Dockerfile` / README からの接続がなく、どの手順でコンテナへ渡すかが未記載。
- チケット `dev-journal/progress-management/tickets/step8/8-2-backend-init.md` の責務には「環境変数・設定管理（internal/config/）— DB接続情報、認証鍵パス、CORS許可オリジン、ログレベル」「api 用 Dockerfile の作成（docker-compose.yml の api サービスが起動可能になる）」が含まれる。

## 判定
重大度: 高
分類: 下流作業可能性 / 共通契約未固定

## 修正方針案
以下のどれかを 8-2 で明示的に固定する。
- `docker-compose.yml` でホストの `./keys` を `/app/keys` に read-only mount する。
- 開発用鍵を起動前に生成する手順を `Makefile` や README に組み込み、`docker compose up` 前提手順として固定する。
- 開発環境ではファイルパスではなく PEM 文字列環境変数を使う方針に変更し、`config` 契約と compose を揃える。

どの方式でも、8-4 担当が追加判断なしに「どこから鍵を読むか」を一意に理解できる状態にする必要がある。

## 再レビュー結果（2026-03-30）

対応不十分（差し戻し継続）。

### 確認内容
- [docker-compose.yml](/root-project/expense-saas/docker-compose.yml#L30) で `JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH` のデフォルト値が `/app/keys/*.pem` に更新され、[docker-compose.yml](/root-project/expense-saas/docker-compose.yml#L35) で `./keys:/app/keys:ro` の mount も追加されている
- これにより、`.env` を使わず compose のデフォルト値だけで起動するケースでは、API コンテナ内の鍵配置契約は固定された

### 差し戻し理由
- [`.env.example`](/root-project/expense-saas/.env.example#L12) と [`.env.example`](/root-project/expense-saas/.env.example#L13) が依然として `./keys/private.pem` / `./keys/public.pem` のままで、開発者が通常どおり `.env.example` を `.env` にコピーすると、その値が [docker-compose.yml](/root-project/expense-saas/docker-compose.yml#L30) と [docker-compose.yml](/root-project/expense-saas/docker-compose.yml#L31) のデフォルトより優先される
- API コンテナ内では `./keys/*.pem` は `/app/keys/*.pem` を指さないため、8-4 で JWT 鍵読込を実装した時点で `.env` 利用時の起動契約が再び壊れる
- したがって、今回の修正は「compose のデフォルト値」を直したに留まり、開発環境の鍵供給方法を一意に固定した状態にはまだ達していない

### 追加で確認すべきファイルや観点
- [`.env.example`](/root-project/expense-saas/.env.example#L12) と [`.env.example`](/root-project/expense-saas/.env.example#L13) を `/app/keys/private.pem` / `/app/keys/public.pem` に揃えること
- 必要なら README か起動手順にも、`./keys` をホスト側に置いて compose で `/app/keys` へ read-only mount する契約を追記し、`.env` 利用時も同じ前提になることを確認すること

### このまま進めた場合の下流影響
- 8-4 担当が `.env.example` ベースで環境を作ると、JWT 鍵の読込先がコンテナ内実パスと不一致になり、`docker compose up` の再現性が失われる
- 結果として、JWT 実装側が鍵読込失敗をアプリ不具合として調査する無駄が発生し、8-2 で固定すべき起動契約が下流へ漏れる

## 再レビュー結果（2026-03-30, 2回目）

対応妥当（クローズ）。

### 確認内容
- [`expense-saas/.env.example`](/root-project/expense-saas/.env.example#L12) と [`expense-saas/.env.example`](/root-project/expense-saas/.env.example#L13) が `/app/keys/private.pem` / `/app/keys/public.pem` に更新されている
- [`expense-saas/docker-compose.yml`](/root-project/expense-saas/docker-compose.yml#L30) と [`expense-saas/docker-compose.yml`](/root-project/expense-saas/docker-compose.yml#L31) のデフォルト値も同じ `/app/keys/*.pem` で一致している
- [`expense-saas/docker-compose.yml`](/root-project/expense-saas/docker-compose.yml#L36) で `./keys:/app/keys:ro` の mount が定義されており、ホスト側配置とコンテナ内参照先の契約が揃っている

### 判定理由
- `.env.example` を `.env` にコピーした通常運用でも compose デフォルト値と矛盾しなくなり、JWT 鍵読込先が `/app/keys/*.pem` に一意化された
- これにより、8-4 担当が追加判断なしに API コンテナ内の鍵配置契約を理解でき、下流作業可能性は回復している

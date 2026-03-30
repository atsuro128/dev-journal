# 074: docker compose で全サービスが起動できる構成になっていない

## 指摘概要
`expense-saas/docker-compose.yml` は 8-1 の成果物として要求されている「DB、バックエンド API、フロントエンド開発サーバー」のコンテナ構成を満たしていない。`api` サービスは `build: .` を指定しているが、リポジトリには `Dockerfile` が存在しないためビルド不能である。さらに `frontend` サービスはコメントアウトされており、コメント内のパスも実在しない `./frontend` を参照している。これではチケットの完了条件「docker compose up で全サービスが起動する」を満たせない。

## 根拠
- チケット `dev-journal/progress-management/tickets/step8/8-1-dev-environment.md`
  - 責務: 「コンテナ構成（PostgreSQL、Go API サーバー、フロントエンド開発サーバー）の docker-compose.yml」
  - 完了条件: 「docker compose up で全サービスが起動する」
- 作業分解 `ai-dev-framework/guide/work-breakdown/step8-foundation.md`
  - 8-1 作業内容: 「コンテナ構成（DB、バックエンド API、フロントエンド開発サーバー）」
  - 8-1 完了条件: 「コンテナ起動コマンドで全サービスが起動し、マイグレーションが正常終了し、テナント分離設定が適用されている」
- 実装
  - `expense-saas/docker-compose.yml:20-21` で `api` は `build: .` を要求している
  - しかしリポジトリ直下に `Dockerfile` / `Dockerfile.*` は存在しない
  - `expense-saas/docker-compose.yml:38-48` で `frontend` サービスがコメントアウトされている
  - コメント内でも `context: ./frontend` と `./frontend:/app` を参照しているが、実在するフロントエンド配置は `expense-saas/apps/web/package.json` であり、`./frontend` ディレクトリは存在しない

## 判定
高 / FIX

## 修正方針案
- `docker compose up` で少なくとも `db`・`api`・`frontend` の 3 サービスが起動するよう、実在するファイル構成に合わせて compose 定義を修正する
- `api` 用の `Dockerfile` を追加するか、8-2 完了まで別の起動方式にするなら 8-1 の責務・完了条件と矛盾しないようタスク分割を見直す
- `frontend` はコメント解除ではなく、実際の配置先 `apps/web` を前提にビルドコンテキスト・volume・起動コマンドを定義する

## 対応内容
8-1 の責務を db サービス起動 + サービス定義枠に限定し、api・frontend のコンテナ実起動責務を 8-2/8-3 に移管。チケット・work-breakdown の両方で整合を確保。

- `dev-journal/progress-management/tickets/step8/8-1-dev-environment.md`: 責務・完了条件を修正
- `dev-journal/progress-management/tickets/step8/8-2-backend-init.md`: api 用 Dockerfile 作成・compose 起動を責務・完了条件に追加
- `dev-journal/progress-management/tickets/step8/8-3-frontend-init.md`: frontend 用 Dockerfile.dev 作成・compose 起動を責務・完了条件に追加
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md`: 8-1, 8-2, 8-3 の作業内容・出力・完了条件を修正

## 再レビュー結果
差し戻し。

8-1 の責務縮小自体は妥当だが、`api` / `frontend` のコンテナを `docker compose` で実起動可能にする責務が、移管先とされた 8-2 / 8-3 のチケットに落ちていない。現状の `dev-journal/progress-management/tickets/step8/8-2-backend-init.md` は `go build`、`dev-journal/progress-management/tickets/step8/8-3-frontend-init.md` は `npm run build` までしか完了条件を持たず、Dockerfile・compose 連携・コンテナ起動可能性を誰が満たすかが未定義のままである。

## 再レビュー結果（第2回）
差し戻し。

前回の差し戻し理由だった「8-2/8-3 に compose 実起動責務が落ちていない」「work-breakdown の 8-2/8-3 詳細が未整合」は解消された。`ai-dev-framework/guide/work-breakdown/step8-foundation.md` の 8-2/8-3 は Dockerfile 作成と `docker compose up` による `api` / `frontend` 起動を明示し、対応するチケット `dev-journal/progress-management/tickets/step8/8-2-backend-init.md` と `dev-journal/progress-management/tickets/step8/8-3-frontend-init.md` も同内容で整合している。

ただし、`dev-journal/progress-management/tickets/step8/8-1-dev-environment.md` の `出力先` が依然として `expense-saas/ (docker-compose.yml, Dockerfile, db/migrations/)` のままで、Dockerfile を 8-1 の成果物として読める状態が残っている。8-1 の責務本文と work-breakdown では Dockerfile 実装責務を 8-2 / 8-3 に移しているため、この 1 行だけが責務分解と矛盾している。

## 未解消の論点
- 8-2 もしくは 8-3、または別チケットに、`api` / `frontend` を既存 `docker-compose.yml` から起動可能にする責務と完了条件が明示されていない
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md` 全体では引き続き「全サービス起動」を Step 8 の完了条件・レビュー観点に置いているため、チケット単位の責務分解と整合していない
- `dev-journal/progress-management/tickets/step8/8-1-dev-environment.md` の `出力先` に `Dockerfile` が残っており、8-2 / 8-3 へ移した Dockerfile 責務と衝突している

## 追加で確認すべきファイルや観点
- `dev-journal/progress-management/tickets/step8/8-2-backend-init.md`
- `dev-journal/progress-management/tickets/step8/8-3-frontend-init.md`
- 必要なら `ai-dev-framework/guide/work-breakdown/step8-foundation.md` の 8-2 / 8-3 詳細、または Step 8 内にコンテナ起動責務を明示する別タスク
- `dev-journal/progress-management/tickets/step8/8-1-dev-environment.md` の `出力先` が、責務縮小後の成果物一覧になっているか

## このまま進めた場合の下流影響
8-2 / 8-3 完了後も `docker compose up` で `api` / `frontend` が起動しない状態をレビューで検知できず、Step 8 の総合完了条件と実際の成果物が再び乖離する。

今回のまま閉じると、8-1 と 8-2/8-3 のどちらが Dockerfile を持つべきかがチケット上で二重化し、担当者アサインやレビュー観点で再度ブレる。

## 再レビュー結果（第3回）
クローズ。

前回差し戻し理由だった `dev-journal/progress-management/tickets/step8/8-1-dev-environment.md` の `出力先` は、6 行目で `expense-saas/ (docker-compose.yml, db/migrations/, scripts/)` に修正されており、`Dockerfile` が除去されている。これにより、8-1 に Dockerfile 成果物が残っていた問題は解消された。

波及確認として `dev-journal/progress-management/tickets/step8/8-2-backend-init.md` と `dev-journal/progress-management/tickets/step8/8-3-frontend-init.md` を再確認したところ、`api` 用 Dockerfile と `frontend` 用 Dockerfile.dev の作成責務、および `docker compose up` で各サービスが起動する完了条件は引き続き明示されている。`ai-dev-framework/guide/work-breakdown/step8-foundation.md` 側も同じ責務分解で整合しており、前回指摘した 8-1 と 8-2/8-3 の Dockerfile 責務二重化は解消済みと判断する。

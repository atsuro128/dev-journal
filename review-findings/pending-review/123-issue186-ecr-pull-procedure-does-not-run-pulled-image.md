# 123: issue #186 の ECR pull 手順が pull したイメージを systemd から起動できない

## 指摘概要

`release.md` は UD-6=A（ECR pull）に従って EC2 上で ECR イメージを pull する手順へ書き換えられているが、pull したリモートイメージを systemd unit が参照するローカルタグへ付け替えていない。手順では `docker pull <ACCOUNT>.dkr.ecr.../expense-saas:<TAG>` の後に、unit ファイルを `expense-saas:<TAG>` へ sed で書き換えているため、ローカルに `expense-saas:<TAG>` が存在せず再起動に失敗する可能性が高い。

実装済み unit は `expense-saas:portfolio` を起動する構成なので、ECR pull 後は `docker tag <remote-image> expense-saas:portfolio` してから `systemctl restart expense-saas` するか、unit 自体をリモート ECR URI を参照する形に同期する必要がある。

## 根拠

- `dev-journal/deliverables/docs/70_operations/release.md:160`
  - `sudo docker pull <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG>` でリモートタグを pull している。
- `dev-journal/deliverables/docs/70_operations/release.md:162-167`
  - unit を `expense-saas:<TAG>` に書き換えて `systemctl restart expense-app` しているが、`docker tag <remote-image> expense-saas:<TAG>` がない。
- `dev-journal/deliverables/docs/70_operations/release.md:284-291`
  - ロールバック手順も同じく、前バージョンのリモートイメージを pull した後、ローカルタグ作成なしに unit を `expense-saas:<前TAG>` へ書き換えている。
- `expense-saas/infra/terraform/user_data.sh.tpl:75-80`
  - 実装済み systemd unit は `docker run --name expense-saas ... expense-saas:portfolio` を起動する。
- `expense-saas/infra/terraform/user_data.sh.tpl:89-93`
  - `image_tag` 指定時の実装は `docker pull "${image_tag}"` の後に `docker tag "${image_tag}" expense-saas:portfolio` を実行してから unit を起動している。

## 判定

重大度: 高

分類: リリース手順不備 / 実装不整合

受け入れ基準 #6（EC2 デプロイ手順）と #7（EC2 ロールバック手順）は、単に `aws ecs update-service` を消すだけでなく、`docker pull` / `docker tag` / `systemctl restart` として実行可能である必要がある。現状は実装側 `user_data.sh.tpl` のタグ付け方式とも、issue #186 §実装計画 v2 の `docker tag` 方針とも一致していない。

## 修正方針案

- `release.md` §4.2 の ECR pull 手順を、実装済み unit に合わせて以下の形へ修正する。
  - `sudo docker pull <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG>`
  - `sudo docker tag <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/expense-saas:<TAG> expense-saas:portfolio`
  - `sudo systemctl restart expense-saas`
- `release.md` §7.1 のロールバック手順も同じく、前タグを `expense-saas:portfolio` に付け替える方式へ揃える。
- unit ファイルを sed で直接書き換える方式を残す場合は、ローカルタグではなく実際に pull 済みの ECR URI を unit が参照するようにし、`user_data.sh.tpl` との設計差分を明示する。

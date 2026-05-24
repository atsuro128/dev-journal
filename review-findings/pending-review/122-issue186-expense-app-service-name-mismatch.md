# 122: issue #186 の EC2 運用手順が存在しない expense-app unit/container 名を参照している

## 指摘概要

issue #186 の解決方針は、下流ドキュメントを EC2 t3.micro 単一インスタンス構成の「動く手順書」に書き直すことだった。しかし `env_config.md` / `release.md` / `runbook.md` は systemd unit と Docker container を `expense-app` として記述している一方、実装済み `user_data.sh.tpl` が作成する unit / container は `expense-saas` である。

このままだと、リリース・ロールバック・障害一次対応で提示された `systemctl` / `journalctl` / `docker` コマンドが実インスタンス上で対象を見つけられず、受け入れ基準 #6 / #7 / #11 の「EC2 ベースの動く手順」になっていない。

## 根拠

- `dev-journal/deliverables/docs/70_operations/release.md:163`
  - `/etc/systemd/system/expense-app.service` を編集する手順になっている。
- `dev-journal/deliverables/docs/70_operations/release.md:167`
  - `sudo systemctl restart expense-app` を実行する手順になっている。
- `dev-journal/deliverables/docs/70_operations/release.md:196-197`
  - `expense-app` container / unit の起動確認を成功条件にしている。
- `dev-journal/deliverables/docs/70_operations/runbook.md:92-100`
  - `docker ps -a --filter name=expense-app`, `docker logs ... expense-app`, `journalctl -u expense-app` を一次対応コマンドとしている。
- `dev-journal/deliverables/docs/70_operations/runbook.md:550-565`
  - インフラ障害コマンド集も `expense-app` を参照している。
- `dev-journal/deliverables/docs/70_operations/env_config.md:277-294`
  - systemd unit スニペット自体が `expense-app.service` / `--name expense-app` になっている。
- `expense-saas/infra/terraform/user_data.sh.tpl:67-80`
  - 実装は `/etc/systemd/system/expense-saas.service` を作成し、`docker run --name expense-saas ... expense-saas:portfolio` を起動する。

## 判定

重大度: 高

分類: issue 解決不備 / 運用手順不整合

単なる命名揺れではなく、復旧・ロールバック時にコマンドが失敗する。issue #186 が問題にしていた「Step 11-E 実デプロイ時の参照手順が動かない」状態が、ECS から EC2 への置換後も別名で残っている。

## 修正方針案

- `env_config.md`, `release.md`, `runbook.md` の systemd unit / container 名を実装に合わせて `expense-saas` に統一する。
- コマンド例は少なくとも以下へ揃える。
  - `sudo systemctl restart expense-saas`
  - `sudo systemctl status expense-saas`
  - `sudo journalctl -u expense-saas --since "..."`
  - `sudo docker ps -a --filter name=expense-saas`
  - `sudo docker logs --tail 100 expense-saas`
- もし設計上 `expense-app` へ改名したいなら、別 issue で `expense-saas/infra/terraform/user_data.sh.tpl` 側の unit / container 名も同時に変更し、実装と手順を同期する。

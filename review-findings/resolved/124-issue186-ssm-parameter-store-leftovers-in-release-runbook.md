# 124: issue #186 後も release/runbook に SSM・Parameter Store 方針と矛盾する残置記述がある

## 指摘概要

UD-1=A（SSM Session Manager）と UD-5=B（SSM Parameter Store）を採用したにもかかわらず、`release.md` と `runbook.md` に旧方針・別サービス・未定義パラメータ名が残っている。これにより、EC2 接続方式とシークレット管理方式の横断整合性が崩れている。

特に `release.md` §4.1 は DB マイグレーションを bastion から実行すると記載しており、`db_schema.md` の「EC2 上の docker run（SSM Session Manager 経由）」とも矛盾する。また、リリース前チェックでは SSM Parameter Store ではなく Secrets Manager を確認対象にしている。

## 根拠

- `dev-journal/deliverables/docs/70_operations/release.md:75`
  - 環境変数チェックは `SSM Parameter Store パラメータ + systemd EnvironmentFile` としている。
- `dev-journal/deliverables/docs/70_operations/release.md:76`
  - 直後のシークレット確認が `Secrets Manager` のまま残っており、UD-5=B / env_config.md §5.3 の SSM Parameter Store 方針と矛盾する。
- `dev-journal/deliverables/docs/70_operations/release.md:101-105`
  - 本番 DB マイグレーションを「踏み台（bastion）から実行」としており、UD-1=A の SSM Session Manager 方針に沿っていない。
- `dev-journal/deliverables/docs/50_detail_design/db_schema.md:941`
  - 本番マイグレーションは「EC2 上の `docker run --rm`（ワンショット）でデプロイ前に実行（SSM Session Manager 経由）」としている。
- `dev-journal/deliverables/docs/70_operations/runbook.md:570-571`
  - Parameter Store 確認コマンドが `/expense-saas/db_password` を参照している。
- `dev-journal/deliverables/docs/70_operations/env_config.md:169-175`
  - 正本のパラメータ名は `/expense-saas/prod/database/url`, `/expense-saas/prod/database/app_url`, `/expense-saas/prod/jwt/private_key`, `/expense-saas/prod/jwt/public_key` であり、`/expense-saas/db_password` は定義されていない。

## 判定

重大度: 中

分類: 横断整合性不備 / SSM 方針未反映

EC2/Fargate の語彙置換は概ね完了しているが、ユーザー判断 UD-1 / UD-5 の運用面が `release.md` / `runbook.md` で一部未反映のまま残っている。後続の Step 11-E 実デプロイ時に、接続経路・シークレット保管先・確認すべき SSM パラメータ名を誤るリスクがある。

## 修正方針案

- `release.md` §3.3 #10 を Secrets Manager ではなく SSM Parameter Store（SecureString）確認へ修正する。
- `release.md` §4.1 の DB マイグレーション実行場所を、`db_schema.md` と一致するよう「SSM Session Manager で EC2 に接続し、EC2 上でワンショット `docker run --rm` または `migrate` を実行」に統一する。
- `runbook.md` の Parameter Store 確認コマンドを、`env_config.md` §5.2/§5.3.2 に定義された `/expense-saas/{env}/database/url` 等のパラメータ名へ揃える。
- SSM/Parameter Store の Terraform 実装が issue #187 側で未完である点は注記として残してよいが、運用手順内の確認対象サービス・パラメータ名は UD-1 / UD-5 の確定方針を正本にする。

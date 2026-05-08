# 116: Q1 の GHCR/ECR 案は runbook のコマンドが成立しておらず判断材料として不十分

## 指摘概要
計画書は Q1 で GHCR / EC2 上 build / ECR の 3 案を提示しているが、実際のコマンド例は EC2 build 案にしか一貫していない。  
GHCR/ECR 案では systemd が起動するイメージ名、migration 実行元の `db/migrations` 配置、seed 実行時の前提ディレクトリが揃っておらず、ユーザーは「どの案なら何を追加で実施すべきか」をコピペ可能な粒度で判断できない。

## 根拠
- Q1 は 3 案からの選択を要求し、案A を実質推奨している。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:377-396, 780-787`
- しかし GHCR 案の例は `ghcr.io/<user>/expense-saas:latest` を pull するだけで、systemd は別名の `expense-saas:portfolio` を起動している。retag か image 名切替の手順がない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:435-453`
- DB migration 手順は `cd ~/expense-saas` のローカル checkout と `migrate -path db/migrations ... up` を前提にしている。GHCR/ECR 案では EC2 上にリポジトリを配置する手順が定義されていない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:527-532`
- seed 実行も `expense-saas:portfolio` を前提にしており、GHCR/ECR 案で pull したイメージ名との整合がない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:605-612`
- `deploy.yml` は現時点で E2E/Smoke が `if: false`、Docker push もコメントアウトの参考実装であり、GHCR/ECR 案の裏付けとしては使えない。  
  `expense-saas/.github/workflows/deploy.yml:156-240`

## 判定
重大度: 中  
分類: warning

## 修正方針案
- Q1 ごとに runbook を分岐し、少なくとも以下を揃えること。
- GHCR/ECR 案: `docker pull` 後の image 名、systemd `ExecStart`、seed 実行コマンド、migration SQL の持ち込み方法。
- EC2 build 案: `git clone` 前提のままでよいが、他案と混在しないよう手順ブロックを分離する。
- どの案でも最終的に `/health`、seed、手動スモークまで同じ終点に到達することを示すこと。

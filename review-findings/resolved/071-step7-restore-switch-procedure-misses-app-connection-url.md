# 071: backup_restore.md の接続先切替手順が DATABASE_APP_URL を更新していない

## 指摘概要
`backup_restore.md` の PITR 復旧手順では、方法 A として `DATABASE_URL` を復元インスタンスへ切り替えて ECS を再デプロイするとしている。しかし `env_config.md` では本番アプリケーションが `DATABASE_URL`（オーナーロール）と `DATABASE_APP_URL`（アプリロール）の 2 系統を使う前提であり、実運用で `DATABASE_APP_URL` を切り替えなければ API は旧 DB へ接続し続ける。現行手順のままではリストア後にアプリケーションが復旧先 DB を参照できない。

## 根拠
- `dev-journal/deliverables/docs/70_operations/env_config.md:88-91,150-168,190-197`
  - DB 接続情報は `DATABASE_URL` と `DATABASE_APP_URL` の 2 変数で管理し、ECS タスク定義でも両方を注入すると定義している。
- `dev-journal/deliverables/docs/70_operations/backup_restore.md:145-148`
  - 方法 A は `DATABASE_URL` だけを復元インスタンスのエンドポイントへ変更して ECS を再デプロイすると書かれている。
- `dev-journal/deliverables/docs/70_operations/backup_restore.md:164-168`
  - 方法 B でも ECS の再デプロイを行うため、実際にアプリケーションが参照する接続情報の整合が必要である。

## 判定
高 / 復旧手順不備

## 修正方針案
- 方法 A の手順を、`DATABASE_URL` と `DATABASE_APP_URL` の両方を復元先へ切り替える記述に修正する。
- どちらの URL をどの目的で使うか（マイグレーション用 / アプリ実行用）を手順内に補足し、切替漏れを防ぐ。
- 復旧後確認に、アプリが `expense_app` ロールで復元先 DB へ接続できていることを確認する観点を追加する。

## 再レビュー結果（2026-03-26）

- **解消**
  - `dev-journal/deliverables/docs/70_operations/backup_restore.md` の手動切替手順自体は MVP スコープ外として削除された
  - そのうえで同ファイルには「`DATABASE_URL` と `DATABASE_APP_URL` の両方を更新すること」という注意事項が残されており、元指摘の切替漏れリスクは保持された
  - `dev-journal/deliverables/docs/70_operations/env_config.md` 側の 2 系統 DB 接続契約とも矛盾していない

- **判定**
  - クローズ可。元指摘の「復旧手順が `DATABASE_APP_URL` を見落としている」問題は、MVP 境界に合わせた手順削除と注意事項保持により解消された

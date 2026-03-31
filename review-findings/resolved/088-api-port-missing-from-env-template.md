# 088: API_PORT が環境変数テンプレートから欠落している

## 指摘概要
`step8/8-10-cleanup` で `.env.example` から `API_PORT` が削除されましたが、`docker-compose.yml` の `api` サービスは引き続き `${API_PORT:-8080}` を公開ポートとして使用しています。  
Step 8-10 の完了条件には「環境変数テンプレートファイルに必要な変数が列挙されているか」が含まれており、この変更によりローカル起動時に必要な可変設定がテンプレートから漏れています。特に Step 8-1 の判断ポイントで要求されている複数インスタンス時のポート競合回避手段が、後続担当から見えなくなります。

## 根拠
- [expense-saas/.env.example](/root-project/expense-saas/.env.example#L15): Server セクションに `PORT`, `CORS_ALLOWED_ORIGINS`, `LOG_LEVEL` はあるが、`API_PORT` が存在しない
- [expense-saas/docker-compose.yml](/root-project/expense-saas/docker-compose.yml#L37): `api` サービスの公開ポートに `${API_PORT:-8080}:8080` を使用している
- [ai-dev-framework/guide/work-breakdown/step8-foundation.md](/root-project/ai-dev-framework/guide/work-breakdown/step8-foundation.md#L195): 8-10 のレビュー観点として「環境変数テンプレートファイルに必要な変数が列挙されているか」を要求
- [ai-dev-framework/guide/work-breakdown/step8-foundation.md](/root-project/ai-dev-framework/guide/work-breakdown/step8-foundation.md#L267): 8-1 の判断ポイントとして複数インスタンス時のポート競合回避方式を固定する必要がある

## 判定
中 / FIX

## 修正方針案
`.env.example` に `API_PORT=8080` を戻し、Compose で利用する可変ポート設定をテンプレート上で明示する。  
もし `API_PORT` を廃止したいなら、`docker-compose.yml` 側も `PORT` に統一し、README や起動手順を含めて参照箇所を全て揃える必要があります。

## 解決内容
- `.env.example` の Server セクションに `API_PORT=8080` を復元し、`docker-compose.yml` の `api` サービスが参照するホスト側公開ポート設定をテンプレートから辿れる状態にした
- `PORT` はコンテナ内 listen ポート、`API_PORT` はホスト側公開ポートという役割分担のままで整合していることを再確認した
- `.env.example` / `docker-compose.yml` / `FRONTEND_PORT` など他の公開ポート設定との並びも確認し、同一 Step 内で新たな不整合は見当たらなかった

## 解決日
2026-03-31

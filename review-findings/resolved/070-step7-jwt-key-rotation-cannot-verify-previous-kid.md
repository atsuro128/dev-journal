# 070: env_config.md の JWT 鍵ローテーション設計では旧 kid のトークンを検証できない

## 指摘概要
`env_config.md` は JWT 鍵ローテーションを「二鍵並行方式」で運用すると定義している一方、環境変数一覧と ECS タスク定義の注入例には現行鍵 (`JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`) しか存在しない。ローテーション手順では Secrets Manager に `JWT_PUBLIC_KEY_PREVIOUS` を追加して 7 日間の猶予期間を設けるとしているが、その値をアプリケーションへ注入する経路が定義されていないため、旧 `kid` で署名された未失効トークンを検証できない。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/security.md:104-112`
  - JWT 鍵ローテーション方式を「二鍵並行方式」とし、`kid` で鍵識別すると定義している。
- `dev-journal/deliverables/docs/70_operations/env_config.md:96-104`
  - 環境変数一覧には `JWT_PRIVATE_KEY` と `JWT_PUBLIC_KEY` しかなく、旧公開鍵の受け口がない。
- `dev-journal/deliverables/docs/70_operations/env_config.md:172-179,181-233`
  - Secrets Manager 構造例と ECS タスク定義の `secrets` 注入例も現行 2 変数のみを前提としている。
- `dev-journal/deliverables/docs/70_operations/env_config.md:271-289`
  - ローテーション手順では `JWT_PUBLIC_KEY_PREVIOUS` をシークレットへ追加し、7 日間は旧公開鍵での検証を継続するとしている。

## 判定
高 / 下流作業可能性欠落

## 修正方針案
- 旧公開鍵をアプリケーションへ渡す正式な設定契約を追加する。
  - 例: `JWT_PUBLIC_KEYS_JSON` のように複数鍵をまとめて注入する、または `JWT_PUBLIC_KEY_PREVIOUS` / `JWT_PREVIOUS_KEY_IDS` を環境変数一覧と ECS タスク定義へ追加する。
- Secrets Manager の構造例、ECS タスク定義例、ローテーション手順を同じ契約に統一する。
- `kid` と公開鍵の対応表をどこで管理するかも、運用手順として明記する。

## 再レビュー結果（2026-03-26）

- **未解消**
  - `dev-journal/deliverables/docs/70_operations/env_config.md` には `JWT_PUBLIC_KEY_PREVIOUS` の注入経路が追加されたが、`kid` と公開鍵の対応表をアプリケーションがどう復元するかを示す設定契約は追加されていない
  - 同ファイルでは「`JWT_PUBLIC_KEY` が現行 kid、`JWT_PUBLIC_KEY_PREVIOUS` が旧 kid に対応する」と説明している一方、実際にその kid 値を渡す環境変数・Secrets Manager 項目・ECS 注入項目が存在しない
  - ローテーション手順では `kid` を UUID v4 で生成すると定義しているため、実装者は受信 JWT ヘッダーの `kid` とどちらの公開鍵を照合すべきかを `env_config.md` だけでは決定できない

## 差し戻し時の追加コメント

- 未解消の論点:
  - 旧公開鍵の注入経路は定義されたが、`kid` と公開鍵 PEM の対応を実装者が復元できる契約が未定義
- 追加で確認すべきファイルや観点:
  - `dev-journal/deliverables/docs/70_operations/env_config.md`
  - `dev-journal/deliverables/docs/50_detail_design/security.md`
  - Secrets Manager 構造例、ECS タスク定義例、ローテーション手順で同じ鍵識別契約になっているか
- そのまま進めた場合に下流で何が困るか:
  - Step 9 実装者は `kid` から公開鍵を選ぶ処理を一意に実装できず、UUID の `kid` を見ても `current/previous` のどちらに対応させるべきか判断できない。結果として二鍵並行方式を文書どおりに実装できない

## 再レビュー結果（2026-03-26 再判定）

- **解消**
  - `dev-journal/deliverables/docs/50_detail_design/security.md` で、JWT 鍵ローテーション（二鍵並行方式）は Phase 3 実装と明記され、MVP は単一鍵ペア運用に変更された
  - `dev-journal/deliverables/docs/70_operations/env_config.md` から `JWT_PUBLIC_KEY_PREVIOUS` を前提とする現行契約が除去され、環境変数一覧・Secrets Manager 構造例・ECS 注入例が単一鍵ペア運用に揃った
  - これにより、MVP の下流実装者は `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` のみを前提に実装でき、元指摘だった「旧 kid トークンを検証できない二鍵並行方式」が Step 7 の現行契約から外れた

## クローズ判断

- 指摘 070 は、MVP スコープの運用設計としては妥当な縮退であり、Step 7 の下流作業可能性は回復したためクローズ可
- 鍵ローテーション時の `kid` と複数公開鍵の対応契約は Phase 3 設計時の検討事項として扱う

---
step: 3
severity: medium
status: open
---

# 030: API URL 設計が SPA ルーティング前提と衝突している

## 指摘概要

SPA 配信方針では `/api/*` を API、`/*` を SPA 静的配信に振り分けると定義していますが、API 一覧では `/auth/login` や `/reports` など `/api` プレフィックスなしの URL を列挙しています。このままではフロントエンドが URL 設計どおりに呼ぶと SPA fallback 側と衝突し、経路設計を一意に解釈できません。

## 根拠

- SPA 配信方針では `/api/*` を API handler、`/*` を埋め込み静的ファイルへルーティングするとしている
  - `dev-journal/deliverables/docs/30_arch/architecture.md:290`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:303`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:304`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:306`
- diagrams.md の構成図でも ALB の経路は `/api/*, /health` と `/*` に分けている
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:30`
  - `dev-journal/deliverables/docs/30_arch/diagrams.md:32`
- 一方で API URL 設計は `/auth/*`, `/reports/*`, `/workflow/*`, `/dashboard`, `/tenant` とプレフィックスなしで定義している
  - `dev-journal/deliverables/docs/30_arch/architecture.md:397`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:400`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:408`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:425`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:431`
  - `dev-journal/deliverables/docs/30_arch/architecture.md:433`

## 判定

- 重大度: 中
- 分類: 文書内不整合 / ルーティング設計

SPA 同居配信では API とフロントの経路境界が実装の前提になります。ここが揺れていると Step 4 の画面/API 設計、Step 5 の OpenAPI 定義、Step 7 のルータ実装のいずれでも解釈が分岐します。

## 修正方針案

- 次のどちらかに統一する
  - API URL を `/api/auth/login`, `/api/reports` のように `/api` プレフィックス付きへ寄せる
  - ルーティング方針側を API 実 URL に合わせて `/auth/*`, `/reports/*`, `/workflow/*` などへ修正する
- architecture.md の 4.0 節、5.1 節、diagrams.md の構成図で同一の URL 体系を使う
- フロントエンド API クライアントの base path 方針も明記する

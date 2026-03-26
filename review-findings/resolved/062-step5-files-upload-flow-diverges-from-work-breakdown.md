# 062: files.md のアップロード方式が Step 5 正本と食い違っている

## 指摘概要
Step 5 の正本では `files.md` に「署名付き URL 発行・クライアント直接アップロード・完了通知」を含むアップロードフローを定義することを要求している。一方、現行の `files.md` と `openapi.yaml` は API 経由プロキシ方式を採用し、署名付きアップロード URL を不採用としている。添付ファイル設計の正本と成果物が矛盾しており、Step 9 実装者と Step 6 テスト設計者がどちらを前提に進めるべきか判断できない。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step5-detail-design.md:122-136`
  - `files.md` の含めるべき内容として「アップロードフロー（署名付き URL 発行・クライアント直接アップロード・完了通知）」を明示している
- `dev-journal/deliverables/docs/50_detail_design/files.md:140-147`
  - 「API 経由プロキシ方式を採用」「署名付きアップロード URL は不採用」と明記している
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1117-1156`
  - 添付アップロード API を `multipart/form-data` で受けるプロキシ方式として定義している

## 判定
- 重大度: 高
- 分類: 正本違反 / 設計契約不整合

## 修正方針案
- まず Step 5 の正本どおりに署名付き URL 方式へ設計を寄せるか、work-breakdown 側の要求を修正するかを決める
- 方式を確定したら `files.md`、`openapi.yaml`、関連 screen 仕様、Step 6 の入力契約が同じ前提になるよう一括で揃える
- 署名付き URL 方式を採る場合は「URL 発行 API」「アップロード完了通知」「失敗時の orphan object 処理」まで明記する

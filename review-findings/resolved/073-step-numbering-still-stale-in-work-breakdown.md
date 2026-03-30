# 073: work-breakdown に旧 Step 番号が残っており受け渡し契約が誤っている

## 指摘概要

Step 番号の新しい並びは `Step 7=運用設計, Step 8=基盤構築, Step 9=テストコード実装, Step 10=機能実装, Step 11=システムテスト` で統一する必要があるが、work-breakdown の一部に旧番号が残っている。

特に `step7-operations.md` と `step5-detail-design.md` の受け渡し契約が「Step 9（機能実装）」を指しており、`step7-operations.md` ではさらに「Step 7 基盤構築」と書かれている。下流担当の着手順や参照先を誤らせるため、今回の横断レビュー観点 3 に直接反する。

## 根拠

- `ai-dev-framework/guide/work-breakdown/step7-operations.md:193`
  - `実装者（Step 7 基盤構築 / Step 9 機能実装）が...`
- `ai-dev-framework/guide/work-breakdown/step7-operations.md:201`
  - `Step 9（機能実装）へ渡すもの:`
- `ai-dev-framework/guide/work-breakdown/step5-detail-design.md:347`
  - `Step 9（機能実装）へ渡すもの:`
- ユーザー指定レビュー観点 3
  - `Step 7=運用設計, Step 8=基盤構築, Step 9=テストコード実装, Step 10=機能実装, Step 11=システムテスト` で統一されているか確認すること

## 判定

中 / Step 番号の統一違反

## 修正方針案

`step7-operations.md` の下流作業可能性と受け渡し契約を `Step 8（基盤構築）` / `Step 10（機能実装）` に修正する。あわせて `step5-detail-design.md` の受け渡し契約も `Step 10（機能実装）` に更新し、Step 7〜11 の番号体系を work-breakdown 全体で再走査する。

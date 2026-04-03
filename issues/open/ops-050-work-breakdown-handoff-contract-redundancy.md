# work-breakdown の「次 Step への受け渡し契約」セクションが冗長かつ不正確

## 発見日
2026-04-03

## カテゴリ
project-management

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
全 Step（work-breakdown 共通構造）

## 問題

全 work-breakdown に存在する「次 Step への受け渡し契約」セクションが、「上流入力」セクションと同じ依存関係を両側から定義しており冗長。さらに、一部の記載が不正確。

1. **冗長性**: Step N の「受け渡し契約」と Step N+1 の「上流入力」が同じ情報を二重管理している。片方を更新してもう片方を更新し忘れると不整合が発生する（issue 039 対応中に実際に発生）
2. **不正確な記載**: Step 5 の受け渡し契約に `architecture.md（Step 3 成果物。環境構成の基盤）` が「Step 5 が渡すもの」として記載されているが、architecture.md は Step 3 の成果物であり Step 5 は作成していない

## 影響

- 依存関係の正本が2箇所に分散し、メンテナンスコストが増加する
- 不整合が発生した場合、作業者がどちらを正とすべきか迷う
- 不正確な記載があると、成果物の責務が曖昧になる

## 提案

1. 「次 Step への受け渡し契約」セクションを全 work-breakdown から廃止する
2. 依存関係の正本は各 Step の「上流入力」セクションに一元化する
3. work-breakdown テンプレート（`ai-dev-framework/templates/work-breakdown-template.md`）から当該セクションを削除する

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

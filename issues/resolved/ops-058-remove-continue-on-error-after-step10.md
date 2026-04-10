# CI の continue-on-error を Step 10 完了後に削除する

## 発見日
2026-04-07

## カテゴリ
infrastructure

## 影響度
高

## 発見経緯
proactive

## 関連ステップ
Step 10（機能実装）

## ブロッカー
Step 10 の全テスト PASS

## 問題

Step 9 で全テストが「期待通りの失敗」状態のため、CI ワークフロー（`.github/workflows/ci.yml`）の go test / vitest run ステップに `continue-on-error: true` を追加した。このまま放置すると Step 10 以降もテスト失敗を CI が検知しなくなる。

## 影響

Step 10 完了後にテストが退行しても CI が PASS し続け、品質ゲートが機能しなくなる。

## 提案

Step 10 の全テストが PASS した時点で `continue-on-error: true` を削除する。対象箇所:
- `.github/workflows/ci.yml` L94-95（go test ステップ）
- `.github/workflows/ci.yml` L118-119（vitest run ステップ）

---

## 解決内容
10-H（CI 安定化）で `continue-on-error: true` を削除済み。10-X 横断レビューで CI 全 PASS を確認し、master にスカッシュマージ (7373054)。

## 解決日
2026-04-11

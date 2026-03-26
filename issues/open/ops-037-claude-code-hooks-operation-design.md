# （タイトル：Claude Code 向け hooks 運用詳細が未定義）

## 発見日
2026-03-24

## カテゴリ
ai-ops

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 7（基盤構築）

## 問題

Step 7 では hooks による format / 軽量 lint を基盤要件として扱う方針にしたが、Claude Code 実行中にどのタイミングで何を走らせるかという運用詳細は未定義。
特に Edit / Write 後の format、自動 lint の対象、危険変更の判定、失敗時の扱いが整理されていない。

## 影響

- hooks の導入だけ先行し、運用時にノイズの多いチェックや過剰ブロックが発生する可能性がある
- 言語ごとの formatter / linter の責務分担が曖昧なまま実装される
- AI 運用設計として切り出す際に、GitHub Actions との役割分担が不明確になる

## 提案

- Step 7 では hooks の最低限実装に留め、運用詳細は別 issue で詰める
- Edit / Write 後の format、危険変更前 lint、失敗時挙動、Actions との重複許容範囲を整理する
- 将来的に `ai-dev-framework/operations/` 配下の運用設計ドキュメントへ反映する

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

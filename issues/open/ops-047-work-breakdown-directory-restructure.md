# work-breakdown をディレクトリ構造に分割してレビュワーの読み込み範囲を制御する

## 発見日
2026-03-30

## カテゴリ
ai-ops

## 影響度
低

## 発見経緯
user-report

## 関連ステップ
Step 8-11（実装系 Step）

## 問題

work-breakdown が1ファイル（例: step8-foundation.md = 378行）に全情報を集約しており、reviewer がレビュー観点だけ必要な場合でもファイル全体を読み込む必要がある。reviewer に不要な情報（タスク詳細、チケット起票手順、依存グラフ等）が76%を占め、コンテキストを圧迫する。

## 影響

- reviewer のコンテキストが圧迫され、実際のコード/成果物に使える余裕が減る
- codex レビューでも同様の問題が発生する

## 提案

Step 8-11 の work-breakdown を単一ファイルからディレクトリ構造に変更し、読者ごとにファイルを分離する。

```
work-breakdown/
  step8/
    main.md          # 指揮役用（タスク一覧、依存、チケット起票手順等）
    upstream.md       # 上流入力一覧（共有）
    review.md         # レビュー観点・完了条件・品質ゲート（reviewer 用）
```

- 指揮役は全ファイルを読む
- reviewer は upstream.md + review.md のみ読む
- Step 0-7 は完了済みのため対象外

### 変更影響

- workflow.md の Step 一覧テーブルのパス
- progress.md のチケットパス参照
- 既存チケット内の参照
- AGENTS.md の参照

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

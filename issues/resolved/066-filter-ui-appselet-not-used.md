# フィルタ UI で AppSelect が使われていない

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 10（機能実装）

## ブロッカー
なし

## 問題

ReportListPage のステータスフィルタが MUI `Select` を直接使用しており、共通コンポーネント `AppSelect` を経由していない。

`AppSelect` は `fullWidth` がデフォルトで有効になっており、フィルタ UI に適用すると画面幅いっぱいに広がってしまうため、現状の `AppSelect` API ではフィルタ用途に不適合。

## 影響

- 新しい一覧画面を追加する際に ReportListPage をコピーすると、素の MUI Select がそのまま増殖する
- `AppSelect` の機能改善（アクセシビリティ追加等）がフィルタ UI に反映されない

## 提案

`AppSelect` の `fullWidth` をオプション化する（`fullWidth?: boolean` でデフォルト true を維持）。
これにより既存のフォーム利用箇所（ItemForm 等）は影響なく、フィルタ UI でも `AppSelect` を使えるようになる。

---

## 解決内容
AppSelect に fullWidth prop を追加（デフォルト true 維持）、displayEmpty を常時有効化。ReportListPage のフィルタを AppSelect に切替。PR #37 でマージ済み。

## 解決日
2026-04-11
